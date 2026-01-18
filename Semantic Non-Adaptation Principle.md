# Semantic Non-Adaptation Principle (SNAP)

## Principle Statement
In systems governed by the **SMA-SIB invariant**, shared semantic representations **MUST NOT** adapt based on user data, usage patterns, or aggregate statistics. 

Semantic consistency across users **SHALL** be achieved exclusively through static, predefined, and versioned semantic definitions, not through learning, optimization, or feedback derived from user interactions.

---

## Rationale
**Adaptive semantics introduce implicit correlation.**

When shared semantic structures evolve in response to user behavior—even in aggregate, anonymized, or statistical form—the system begins to encode population-level information. This creates a **covert memory channel** that violates the core SMA-SIB guarantee: that no stored or shared representation may enable inference about specific users, rare conditions, collective events, or temporal patterns.

> **Semantic Non-Adaptation ensures that shared meaning does not become shared memory.**

---

## Normative Requirements
To comply with the Semantic Non-Adaptation Principle, an SMA-SIB system **MUST** enforce all of the following:

| Requirement | Description |
| :--- | :--- |
| **Frozen Semantics** | Shared semantic codebooks, ontologies, or equivalence classes **MUST** be frozen at deployment time and **MUST NOT** be modified based on user inputs, frequencies, or inferred distributions. |
| **No Global Statistics** | The system **MUST NOT** maintain global counters, trends, heatmaps, popularity metrics, or usage analytics tied to semantic classes. |
| **Deterministic Mapping** | Identical inputs processed under the same semantic version **MUST** map to identical semantic representations, regardless of user identity, time, or system load. |
| **Explicit Versioning** | Semantic evolution **MUST** occur through explicit, discontinuous version changes. Silent refinement, online learning, or gradual drift is prohibited. |
| **No Retrospective** | Stored semantic representations **MUST NOT** be reinterpreted or reclassified under newer semantic versions. |

---

## Non-Goals (Clarifications)
Semantic Non-Adaptation does **not** require semantic completeness, optimal categorization, or expressive richness. 
* Lossy, coarse, or over-general semantic collapse is acceptable and often desirable. 
* Precision optimization is explicitly subordinate to **irreversibility** and **non-correlation**.

---

## Security Interpretation
Under this principle:
* Semantics function as a **static lattice**, not a learned model.
* Meaning is shared; **experience is not.**
* Improvement pressure is redirected from "better fit" to **"stable collapse."**

This aligns SMA-SIB semantics more closely with standards (e.g., medical codes, legal taxonomies, protocol enums) than with adaptive machine learning systems.

---

## Summary
**Semantic Non-Adaptation ensures that shared meaning never becomes a side-channel for shared experience.** It is a necessary condition for scaling SMA-SIB systems across users without reintroducing implicit data correlation or population-level leakage.
