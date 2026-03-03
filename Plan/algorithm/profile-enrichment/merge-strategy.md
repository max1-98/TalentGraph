# Profile Enrichment — Merge Strategy

Rules for combining external signals with an existing CandidateTensor.

---

## Conflict Resolution Hierarchy

When external data covers a field already present in the tensor, conflicts are
resolved by a four-tier precedence hierarchy:

1. **Explicit CV data** — always wins. The candidate deliberately stated this.
2. **High-confidence external (confidence >= 0.80)** — strong corroboration from
   a verified source (e.g. certification registry, verified employer record).
3. **Inferred external (0.50–0.79)** — probable but not verified (e.g. skill
   inferred from open-source commit patterns).
4. **Low-confidence external (< 0.50)** — speculative signal; stored as
   metadata but never merged into primary tensor fields.

**Invariant:** CV data is never silently overwritten. When a higher-confidence
external signal contradicts the CV, the conflict is logged in the enrichment
log, and the external value is stored as an alternative — not as a replacement.
Overwriting requires explicit candidate approval.

## MergeStrategy Trait Contract

The abstract interface for all merge implementations.

- **Inputs:** existing CandidateTensor, a list of extracted signals (each with
  source metadata, confidence, and timestamp), and the candidate's privacy
  preferences (see `privacy.md`)
- **Output:** merged CandidateTensor + a list of EnrichmentLogEntry records
  (one per field touched or conflict detected)
- **Invariants:**
  - Confidence never decreases for a field unless the candidate explicitly
    removes data
  - Every change is traceable to a source and a run ID
  - Opt-out preferences are respected before any merge begins
  - The merge is idempotent: re-running with the same inputs produces the
    same tensor

### Variants

| Variant | Behaviour | Use Case |
|---------|-----------|----------|
| **Production** | Full merge with conflict detection and enrichment log | Live system |
| **Additive** | Only fills empty fields; never overrides existing data | Conservative mode, new deployments |
| **Test** | Deterministic, no external calls, fixed timestamp | Unit and integration tests |

## Field-Level Merge Rules

Each tensor field type has specific merge behaviour:

- **Skills** — union of CV skills and external skills. Each skill carries a
  provenance tag (cv, external, inferred). When the same skill appears from
  both sources, the higher proficiency estimate wins, and both sources are
  recorded. New skills from external sources inherit the extraction confidence
  of the source.
- **Embeddings** — full recomputation from the updated profile text after merge.
  Embeddings are never averaged or interpolated across sources.
- **Experience entries** — append non-overlapping entries. Overlapping entries
  (same employer + overlapping dates) are merged by taking the richer
  description and logging the conflict.
- **Education** — append if the credential is verified (source confidence
  >= 0.80). Unverified education signals are stored in the enrichment log
  only.
- **Identity fields** — never modified by enrichment. Name, email, and
  location are candidate-controlled exclusively.
- **Career trajectory** — recomputed from the merged experience timeline after
  all experience entries are reconciled.

## Confidence Update Model

Field-level and aggregate confidence follow a Bayesian update pattern,
consistent with the model defined in `cv-to-profile/confidence.md`.

- **Corroborating evidence** — when an external source confirms an existing
  field, the posterior confidence increases:
  P(correct | corroboration) = P(correct) * P(source_agrees | correct) /
  P(source_agrees)
- **New fields** — receive the source's extraction confidence as their prior,
  discounted by the source's historical accuracy rate
- **Aggregate confidence** — the tensor-level confidence score is recomputed
  after each enrichment run as a weighted mean of field-level confidences,
  weighted by field importance (skills and experience weigh more than
  inferred metadata)

## Enrichment Generation Counter

Each enrichment run increments an `enrichment_generation` counter on the
tensor. This counter is part of the version vector (see `versioning.md`) and
enables:

- Staleness detection (tensor enriched with generation N vs current generation M)
- Audit trail (which generation introduced which fields)
- Rollback (revert to a previous generation if enrichment introduced errors)

---

## Cross-References

- Parent: `overview.md`
- Confidence model: `cv-to-profile/confidence.md`
- Core tensor schema: `core/types.md`
- Privacy and opt-out: `privacy.md`
- Version vector: `versioning.md`
