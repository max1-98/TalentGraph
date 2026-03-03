# Ingestion Pipeline — Admission

Frequency gating and initial placement for candidate skills. Not every
extracted mention deserves a taxonomy node — typos, one-off jargon, and noise
must be filtered. This stage separates signal from noise.

Depends on: `graph-model/contracts.md` (SkillProvider, GraphMutator),
MarketSignalProvider (shared building block).

---

## Candidate Lifecycle

Each potential new skill passes through a stateful lifecycle:

1. **Observed:** first sighting. A candidate record is created with the
   mention text, embedding, source document type, and timestamp.
2. **Accumulating:** subsequent independent sightings from different documents
   increment the observation count. Each sighting records source type (CV or
   JD) and source ID to prevent double-counting from the same document.
3. **Decision point:** after sufficient observations or after a timeout
   window, the candidate is either promoted or rejected.
   - *Promoted:* transitions to active SkillNode via GraphMutator.add_node
     followed by GraphMutator.promote.
   - *Rejected:* discarded after 90 days below the promotion threshold.
     The embedding is retained briefly in case a near-duplicate reappears.

---

## Poisson Anomaly Gating

Rather than a naive fixed-count threshold ("promote after 5 sightings"), we
use a statistical test based on the Poisson distribution.

**Background noise rate (lambda_bg):** estimated from the historical rate of
candidate mentions that were ultimately rejected (typos, jargon, etc.). This
is the expected number of false-positive mentions per time window.

**Promotion test:** given lambda_bg and the observation window T, we compute
the probability of seeing k or more independent mentions of this exact term
under the null hypothesis (it's noise):

  P(X >= k) = 1 - SUM_{i=0}^{k-1}( (lambda_bg * T)^i * exp(-lambda_bg * T) / i! )

If P < 0.01, the term is appearing far too often to be noise — promote it.
This adapts to the actual noise level rather than using an arbitrary fixed
count.

**Fallback:** when insufficient historical data exists to estimate lambda_bg
(e.g. cold start), fall back to a fixed count threshold of 5 independent
sightings from at least 2 source types.

---

## Quality Checks

Before promotion, every candidate must pass:

1. **Distinctness:** embedding must be sufficiently distant from all existing
   active nodes (max cosine similarity < 0.80). If too close, redirect to
   the deduplication pipeline instead of promoting.
2. **Name validation:** canonical name must pass length bounds (2–100 chars),
   must not be purely numeric, and must not match a known stop-phrase list
   (e.g. "experience", "proficient", "skills").
3. **Source diversity:** sightings must come from at least 2 source types
   (e.g. CV + JD) to prevent single-source bias. A skill appearing only in
   JDs might be recruiter jargon; one appearing only in CVs might be
   self-description noise.

---

## AdmissionGate Trait Contract

- **Input:** CandidateRecord (text, embedding, observation count, source types,
  timestamps, observation window)
- **Output:** AdmissionDecision (promote | reject | accumulate)
- **Invariant:** a promoted candidate always satisfies all quality checks.
  A rejection is final for that candidate ID (though the same text may
  re-enter as a new candidate after the retention window).

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Poisson gating + all quality checks | Normal operation |
| FixedThreshold | Promote after N sightings, skip Poisson | Cold start, simple deployments |
| AutoPromote | Promote immediately (no gating) | Testing and bulk import |

---

## Initial Placement

Upon promotion, the new node needs provisional edges to connect it to the
existing graph:

- **Parent assignment:** computed via asymmetric conditional probability (see
  `hierarchy/overview.md`). If AR(new, existing) > 2.0 for any existing node,
  that node becomes a provisional parent. Multiple parents are allowed.
- **Related edges:** if NPMI between the new node and an existing node exceeds
  0.3 (but the relationship is symmetric, AR between 0.5 and 2.0), a related
  edge is created.
- **No placement:** if no sufficiently strong relationships are found, the node
  is added as an orphan. The next hierarchy inference cycle will attempt
  placement (see `hierarchy/inference.md`).

All initial placements are provisional — confidence is marked as low (0.3–0.5)
and refined during the next hierarchy inference cycle.

---

## MarketSignalProvider Integration

When market signals indicate a surge in a domain (e.g. a new framework
trending across JDs), the promotion threshold is adjusted:

  N_adjusted = max(3, N * (1 - demand_boost))

Where demand_boost is a 0–1 signal from MarketSignalProvider reflecting how
strongly demand for this skill category is growing. This fast-tracks
genuinely emerging skills while the base Poisson gating still filters noise.

---

## Cross-References

- Discovery and classification: `ingestion/overview.md`
- Graph mutation contracts: `graph-model/contracts.md`
- Hierarchy inference for placement refinement: `hierarchy/inference.md`
- Market signals: `algorithm/overview.md` (MarketSignalProvider)
- Deduplication redirect: `deduplication/overview.md`
