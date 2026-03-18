# SMA-SIB: Reference Implementation Guide

**Version 1.0** · *Reference Pipeline*
**January 2026** · *Frozen Engineering Standard*

---

## Overview

This document provides the canonical engineering interpretation of the SMA-SIB specification. It emerged from adversarial refinement across multiple independent implementation approaches and represents the convergent solution that survived stress-testing.

**Key Insight:** Privacy must be a *geometric* property, not a *lexical* one. We do not filter sensitive data — we project it into a space where specifics cannot geometrically exist.

---

## 1. The Core Pipeline

To satisfy the SMA-SIB invariant ("No stored representation may correspond to a unique real-world entity"), the system adheres to a strict one-way data flow.

```
┌─────────────────┐
│   Raw Input     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Pre-Sanitization │  ← Optional (optimization only)
│     Layer        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Ephemeral    │  ← Transient computation, non-persistent
│     Encoder     │
└────────┬────────┘
         │ Transient vector (volatile memory only)
         ▼
┏━━━━━━━━━━━━━━━━━━━┓
┃   IRREVERSIBILITY  ┃  ← Semantic Irreversibility Boundary
┃     BOUNDARY       ┃
┗━━━━━━━━┳━━━━━━━━━━━┛
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐  ┌───────┐
│ Index │  │  NULL │  ← Residual destroyed permanently
└───┬───┘  └───────┘
    │
    ├──────────────┐
    ▼              ▼
┌─────────┐  ┌──────────┐
│Topology │  │   Time   │
│  Store  │  │  Store   │
│(Bitmask)│  │ (Bucket) │
└─────────┘  └──────────┘
```

**Critical Rule:** Nothing before the Irreversibility Boundary persists. Nothing after it can express specifics. The encoder output exists only in volatile memory during processing and is never written to any persistent medium.

---

## 2. Component Specifications

### A. Pre-Sanitization Layer

**Role:** Optimization and attack surface reduction.
**Mechanism:** Fast pattern matching (regex).

**Operational Constraint:** Encoder execution MUST occur in volatile memory with crash-dump suppression enabled (or equivalent sandboxing). Intermediate representations MUST NOT be written to disk under any circumstances, including error conditions. Equivalent isolation guarantees MAY be provided by hardware-enforced enclaves or confidential computing environments.

**Function:** Strips obvious PII patterns (SSNs, emails, phone numbers) before they reach the heavy encoder.

**Constraint:** Failure here is acceptable — the Irreversibility Boundary catches what this layer misses. This layer exists for speed, not safety.

```python
# Example patterns (non-exhaustive)
PATTERNS = [
    r'\b\d{3}-\d{2}-\d{4}\b',        # SSN
    r'\b[\w.-]+@[\w.-]+\.\w+\b',     # Email
    r'\b\d{10,}\b',                   # Long numbers
    r'\b\d{4}-\d{2}-\d{2}\b',        # ISO dates
]
```

**Warning:** Regex is brittle. Typos, synonyms, and paraphrases will bypass it. This is why it is not the safety boundary.

---

### B. The Irreversibility Boundary

**Role:** Enforcement of the SMA-SIB invariant.
**Mechanism:** Hard Vector Quantization (VQ).

This is where irreversibility is enforced mathematically.

#### How It Works

1. **Codebook (C):** A frozen, pre-computed matrix of *K* vectors representing only allowed generic concepts.

2. **Projection:** The transient input vector is compared to all vectors in the codebook.

3. **Snap:** The nearest neighbor index is selected:
   ```
   index = argmin_i ||x - c_i||²
   ```

4. **Discard:** The residual represents non-representable specificity and is discarded without materialization. It is never computed, stored, or logged.

#### The Safety Guarantee

Without the residual, the transformation from index back to input is mathematically impossible. The specific information is not hidden — it is gone.

#### Codebook Design

Codebooks SHOULD be constructed from synthetic, publicly available, or domain-abstract corpora. Codebooks MUST NOT be trained or refined on user data. Concept labels are illustrative; actual deployments SHOULD use domain-appropriate abstractions with sufficiently large equivalence classes to prevent meaningful inference from index values alone.

Auditable generation procedures are recommended for regulated environments.

#### Reference Implementation

```python
import numpy as np

class IrreversibleQuantizer:
    def __init__(self, codebook: np.ndarray):
        """
        codebook: Shape (K, D) where K = number of concepts, D = embedding dim
        CRITICAL: Codebook must be frozen. No learning after deployment.
        """
        self.codebook = codebook
        self.codebook.flags.writeable = False  # Enforce immutability
    
    def quantize(self, input_vector: np.ndarray) -> int:
        """
        Returns ONLY the index. The residual is never computed or stored.
        """
        distances = np.linalg.norm(self.codebook - input_vector, axis=1)
        index = np.argmin(distances)
        
        # CRITICAL: We return the index, not the vector.
        # The difference (residual) is never materialized.
        return index
```

---

### C. Topology Store (Plane 2)

**Mechanism:** Bitmasks.

Each bit represents a domain. Co-occurrence is recorded via bitwise OR.

```python
# Domain definitions
HEALTH  = 0b0001  # bit 0
LEGAL   = 0b0010  # bit 1
WORK    = 0b0100  # bit 2
FINANCE = 0b1000  # bit 3

# Update operation
user_state = 0b0000
user_state |= HEALTH  # User touched health domain
user_state |= LEGAL   # User touched legal domain
# Result: 0b0011

# Query operation
has_health_and_legal = (user_state & (HEALTH | LEGAL)) == (HEALTH | LEGAL)
```

**Properties:**
- O(1) operations
- No edges, no ordering, no narrative
- Scales to ~64 domains with a single integer

**Note:** For >50 domains, consider Hyperdimensional Computing (HDC) with orthogonal random vectors. For most applications, bitmasks suffice.

---

### D. Time Store (Plane 3)

**Mechanism:** Discrete buckets.

```python
from enum import Enum
from datetime import datetime, timedelta

class Recency(Enum):
    RECENT = "0-30 days"
    MODERATE = "30-180 days"
    DISTANT = "180+ days"

class Frequency(Enum):
    LOW = "1-10"
    MEDIUM = "11-50"
    HIGH = "51+"

def bucket_recency(last_activation: datetime) -> Recency:
    delta = datetime.now() - last_activation
    if delta < timedelta(days=30):
        return Recency.RECENT
    elif delta < timedelta(days=180):
        return Recency.MODERATE
    return Recency.DISTANT

def bucket_frequency(count: int) -> Frequency:
    if count <= 10:
        return Frequency.LOW
    elif count <= 50:
        return Frequency.MEDIUM
    return Frequency.HIGH
```

**Critical:** After bucketing, the precise count is discarded. Store only the bucket enum.

---

## 3. The Adversarial Compiler (CI/CD Enforcement)

Compliance is enforced at build time, not runtime.

### The Test Harness

```python
def adversarial_build_test(pipeline, num_samples=1000):
    """
    If this test passes, the build proceeds.
    If it fails, the build is rejected.
    """
    # 1. Generate synthetic inputs with known identifiers
    inputs = generate_synthetic_pii_logs(num_samples)
    labels = extract_proper_nouns(inputs)
    
    # 2. Process through the SMA-SIB pipeline
    stored_states = [pipeline.process(inp) for inp in inputs]
    
    # 3. Train an adversarial probe
    probe = LogisticRegression()
    probe.fit(stored_states, labels)
    
    # 4. Evaluate
    accuracy = probe.score(stored_states, labels)
    baseline_chance = 1.0 / len(set(labels))
    
    # 5. FAIL if probe can recover specifics
    threshold = baseline_chance + 0.01  # 1% above chance
    if accuracy > threshold:
        raise BuildError(
            f"SECURITY VIOLATION: Probe accuracy {accuracy:.2%} "
            f"exceeds threshold {threshold:.2%}. "
            f"Information is leaking through the boundary."
        )
    
    return True
```

### What This Means

The system cannot be deployed unless it empirically demonstrates opacity under adversarial probing. This turns privacy from a policy ("we promise not to look") into a verifiable property ("looking reveals nothing").

**Critical:** The probe model MUST be discarded after evaluation and MUST NOT be reused across builds. Adversarial test artifacts MUST NOT be persisted beyond the build environment.

**Framing:** Irreversibility is structural. The adversarial test verifies that the structure holds — it does not create the guarantee.

---

## 4. Forbidden Patterns

These patterns compromise the invariant and are strictly prohibited.

### No Debug Mode

```python
# FORBIDDEN
def process(input, debug=False):
    residual = compute_residual(input)
    if debug:
        save_to_disk(residual)  # VIOLATION
```

There is no flag to preserve residuals. Ever.

### No Softmax

```python
# FORBIDDEN
probabilities = softmax(distances)
store(probabilities)  # Leaks proximity to other concepts
```

Store only the hard ArgMin index. Probability distributions leak information about how close the input was to alternative concepts.

### No Fine-Tuning

```python
# FORBIDDEN
codebook.update(user_data)  # The codebook becomes the memory
```

The codebook is frozen at deployment. Fine-tuning on user data encodes user specifics into the model weights. See the [Semantic Non-Adaptation Principle](.Semantic_Non-Adaptation_Principle.md).

### No Temporary Storage

```python
# FORBIDDEN
temp_file = save_input_for_later_processing(input)
# Even "temporary" creates a window for compromise
```

The input exists only in volatile memory during processing.

---

## 5. Validation Checklist

Before deployment, verify:

| Check | Requirement |
|-------|-------------|
| Codebook frozen | `codebook.flags.writeable == False` |
| No residual storage | Grep codebase for residual/difference/error storage |
| Adversarial test passes | Probe accuracy ≤ chance + 1% |
| No debug flags | No conditional paths that preserve specifics |
| No probability storage | Only hard indices stored |
| Topology is flat | No edges, only set membership |
| Time is bucketed | No precise timestamps or counts |

---

## 6. Architecture Decision Records

### Why VQ Over Pattern Matching?

Pattern matching strips known patterns. VQ projects to a constrained space. If an attacker finds a pattern the matcher doesn't know, they win. With VQ, unknown patterns still snap to the nearest generic concept — the attacker cannot recover what does not geometrically exist.

### Why Bitmasks Over Graphs?

Graphs preserve structure. "Health → Legal → Work" tells a story. Bitmasks say only "these domains were touched." No narrative, no timeline, no edges.

### Why Buckets Over Continuous Time?

Continuous values can be correlated. A precise activation pattern is a fingerprint. A coarse bucket is not.

---

## Appendix: Convergence Note

This implementation emerged from adversarial refinement across multiple independent approaches. When independent systems converge on the same solution under adversarial conditions, the solution space has typically collapsed to a narrow, correct basin.

---

*This is a reference constraint, not a living project. Do not add features.*
