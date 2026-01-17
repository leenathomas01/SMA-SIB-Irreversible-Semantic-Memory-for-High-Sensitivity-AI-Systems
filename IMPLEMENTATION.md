# SMA-SIB: Reference Implementation Guide

**Version 1.0** · *The "Mutant" Protocol*  
**January 2026** · *Frozen Engineering Standard*

---

## Overview

This document provides the canonical engineering interpretation of the SMA-SIB specification. It emerged from adversarial debate between multiple implementation approaches and represents the convergent solution that survived stress-testing.

**Key Insight:** Privacy must be a *geometric* property, not a *lexical* one. We do not filter sensitive data—we project it into a space where specifics cannot geometrically exist.

---

## 1. The Core Pipeline

To satisfy the SMA-SIB invariant ("No stored representation may correspond to a unique real-world entity"), the system must adhere to a strict **One-Way Data Flow**.

```
┌─────────────────┐
│   Raw Input     │
└────────┬────────┘
         │ Dirty Context
         ▼
┌─────────────────┐
│  Pre-Sanitizer  │  ← Optional (optimization only)
└────────┬────────┘
         │ Cleaned Context
         ▼
┌─────────────────┐
│    Ephemeral    │  ← High-fidelity, non-persistent
│     Encoder     │
└────────┬────────┘
         │ High-Dim Vector
         ▼
┏━━━━━━━━━━━━━━━━━┓
┃   THE EVENT     ┃  ← Semantic Irreversibility Boundary
┃    HORIZON      ┃
┗━━━━━━━━┳━━━━━━━━┛
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐  ┌───────┐
│ Index │  │  NULL │  ← Residual destroyed forever
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

**Critical Rule:** Nothing before the Event Horizon persists. Nothing after it can express specifics.

---

## 2. Component Specifications

### A. The Pre-Sanitizer ("Grok Layer")

**Role:** Optimization & Attack Surface Reduction  
**Mechanism:** Fast Regex / Pattern Matching

**Operational Constraint:** Encoder execution MUST occur in volatile memory with crash-dump suppression enabled (or equivalent sandboxing). Intermediate representations MUST NOT be written to disk under any circumstances, including error conditions. Equivalent isolation guarantees MAY be provided by hardware-enforced enclaves or confidential computing environments.

**Function:** Strips obvious PII (SSNs, emails, phone numbers) before they reach the heavy encoder.

**Constraint:** Failure here is *acceptable*—the Event Horizon will catch it. This layer exists for speed, not safety.

```python
# Example patterns (non-exhaustive)
PATTERNS = [
    r'\b\d{3}-\d{2}-\d{4}\b',        # SSN
    r'\b[\w.-]+@[\w.-]+\.\w+\b',     # Email
    r'\b\d{10,}\b',                   # Long numbers
    r'\b\d{4}-\d{2}-\d{2}\b',        # ISO dates
]
```

**Warning:** Regex is brittle. Typos, synonyms, and paraphrases will bypass it. This is why it's *not* the safety boundary.

---

### B. The Event Horizon ("Gemini Layer")

**Role:** The Semantic Irreversibility Boundary (SIB)  
**Mechanism:** Hard Vector Quantization (VQ)

This is where irreversibility is enforced mathematically, not lexically.

#### How It Works

1. **Codebook (C):** A frozen, pre-computed matrix of *K* vectors representing only allowed generic concepts (e.g., `Health_Action`, `Legal_Risk`, `Work_Domain`).

2. **Projection:** Input vector **x** (from the dirty encoder) is compared to all vectors in **C**.

3. **Snap:** Find the nearest neighbor:
   ```
   index = argmin_i ||x - c_i||²
   ```

4. **Discard:** The residual vector **r = x - c_index** contains the specifics ("Dr. Smith", "Lupus", "$45,000"). This residual is **permanently discarded**.

#### The Safety Guarantee

Without **r**, the transformation `c_index → x` is mathematically impossible to invert. The specific information isn't hidden—it's **gone**.

#### Codebook Provenance (Non-Normative)

Codebooks SHOULD be constructed from synthetic, publicly available, or domain-abstract corpora. Codebooks MUST NOT be trained or refined on user data. Auditable generation procedures are recommended for regulated environments.

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
    
    def get_concept(self, index: int) -> str:
        """
        Maps index to human-readable concept (for output generation).
        """
        # Example mapping - define based on your domain
        concepts = {
            0: "HealthInteraction",
            1: "LegalRisk", 
            2: "WorkDomain",
            3: "FinanceAnomaly",
            # ... etc
        }
        return concepts.get(index, "GenericInteraction")
```

---

### C. Topology Store (Plane 2)

**Mechanism:** Bitmasks

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

**Mechanism:** Discrete Buckets

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

Compliance is enforced at **build time**, not runtime.

### The Test Harness

```python
def adversarial_build_test(pipeline, num_samples=1000):
    """
    If this test passes, the build proceeds.
    If it fails, the build is rejected.
    """
    # 1. Generate synthetic inputs with known proper nouns
    inputs = generate_synthetic_pii_logs(num_samples)
    labels = extract_proper_nouns(inputs)  # Ground truth
    
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
    
    return True  # Build passes
```

### What This Means

The system cannot be deployed unless it is **proven opaque**. This turns privacy from a policy ("we promise not to look") into a mathematical property ("looking reveals nothing").

**Critical:** The probe model MUST be discarded after evaluation and MUST NOT be reused across builds. Adversarial test artifacts MUST NOT be persisted beyond the build environment. This prevents accidental meta-learning across iterations.

---

## 4. Forbidden Patterns (Anti-Patterns)

These patterns compromise the invariant and are **strictly prohibited**:

### ❌ No "Debug Mode"

```python
# FORBIDDEN
def process(input, debug=False):
    residual = compute_residual(input)
    if debug:
        save_to_disk(residual)  # VIOLATION
```

There is no flag to save residuals for debugging. Ever.

### ❌ No Softmax

```python
# FORBIDDEN
probabilities = softmax(distances)
store(probabilities)  # Leaks nuance about the input
```

Store only the hard ArgMin index. Probability distributions leak information about how "close" the input was to other concepts.

### ❌ No Fine-Tuning

```python
# FORBIDDEN
codebook.update(user_data)  # The codebook becomes the memory
```

The Codebook is **frozen** at deployment. If you fine-tune on user data, you've encoded user specifics into the model weights.

### ❌ No "Temporary" Storage

```python
# FORBIDDEN
temp_file = save_input_for_later_processing(input)
# Even "temporary" creates a window for compromise
```

The dirty input exists only in volatile memory during processing, then is gone.

---

## 5. Validation Checklist

Before deployment, verify:

| Check | Requirement |
|-------|-------------|
| ☐ Codebook frozen | `codebook.flags.writeable == False` |
| ☐ No residual storage | Grep codebase for residual/difference/error storage |
| ☐ Adversarial test passes | Probe accuracy ≤ chance + 1% |
| ☐ No debug flags | No conditional paths that preserve specifics |
| ☐ No probability storage | Only hard indices stored |
| ☐ Topology is flat | No edges, only set membership |
| ☐ Time is bucketed | No precise timestamps or counts |

---

## 6. Example: Full Pipeline Trace

**Input:** "Dr. Vance at Orion Clinic diagnosed my lupus on 2026-01-10, prescribed 20mg Plaquenil, and I took work leave."

| Stage | State |
|-------|-------|
| **Pre-Sanitizer** | Strips "2026-01-10" → `[DATE]`, "20mg" → `[DOSE]` |
| **Encoder** | Produces vector `[0.9, 0.1, 0.4, 0.0]` (health-heavy) |
| **VQ Snap** | Nearest to index 0 (`HealthInteraction`), distance 0.17 |
| **Residual** | `[-0.1, 0.1, 0.4, 0.0]` — **DISCARDED** |
| **Topology** | `user_state \|= HEALTH \| WORK` → `0b0101` |
| **Time** | Recency: `RECENT`, Frequency: `LOW` |

**Stored State:**
```json
{
  "concept_index": 0,
  "domains": 5,
  "recency": "RECENT",
  "frequency": "LOW"
}
```

**What can be recovered:** "User had a health interaction, also touched work domain, recently, low frequency."

**What cannot be recovered:** Dr. Vance, Orion Clinic, lupus, 2026-01-10, 20mg, Plaquenil.

---

## 7. Architecture Decision Records

### Why VQ over Regex?

Regex strips known patterns. VQ projects to a constrained space. If the attacker finds a pattern regex doesn't know, they win. With VQ, unknown patterns still snap to the nearest safe concept—the attacker can't "find" what doesn't geometrically exist.

### Why Bitmasks over Graphs?

Graphs preserve structure. "Health → Legal → Work" tells a story. Bitmasks say only "these domains were touched." No narrative, no timeline, no edges.

### Why Buckets over Continuous Time?

Continuous values can be correlated. "Activated 147 times in 30 days starting January 3rd" is a fingerprint. "HIGH frequency, RECENT" is not.

---

## Appendix: The Convergence Story

This implementation emerged from adversarial debate:

- **Approach A** (Regex/Rules): Fast, brittle, leaks on unknown patterns
- **Approach B** (VQ/Geometric): Slower, robust, mathematically verifiable

The "Mutant" combines:
- Regex as optional pre-filter (speed)
- VQ as mandatory boundary (safety)
- Bitmasks for topology (simplicity)
- Buckets for time (auditability)

Independent implementations converged on this architecture, which is a strong signal of design correctness.

---

*This is not a living project. It's a reference constraint. Do not add features.*
