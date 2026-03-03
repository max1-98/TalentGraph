# Profile Enrichment — Privacy Model

Consent, disclosure, retention, and regulatory alignment for external data
enrichment.

---

## Opt-Out Schema

Per-candidate flags checked before any enrichment processing begins:

- **opt_out_enrichment** (boolean) — if true, no external sources are consulted
  and the candidate's tensor is never modified by enrichment
- **opt_out_sources** (list of source identifiers) — selective opt-out; the
  candidate blocks specific platforms while allowing others
- **data_retention_preference** — one of: standard, minimal, none (see Data
  Retention below)

**Invariant:** opt-out flags are checked before any external API calls are
made. A candidate who opts out never generates external network traffic on
their behalf.

## Source Disclosure

Every enriched field traces back to its source. The disclosure record contains:

- **source_url** — the public URL from which the signal was extracted
- **extraction_timestamp** — when the data was retrieved
- **source_platform** — normalised platform identifier (e.g. github, linkedin,
  certification_registry)
- **verification_level** — one of: verified (platform-confirmed), public
  (self-reported on external platform), inferred (derived from activity
  patterns)

Source disclosure records are visible to the candidate in their profile
settings. Candidates can review which external sources contributed to their
profile and request removal of specific source contributions.

## Enrichment Log Schema

Every enrichment run produces exactly one log entry, even if no changes result:

- **candidate_id** — the target candidate
- **run_timestamp** — when the enrichment run executed
- **enrichment_generation** — monotonic counter, incremented per run
- **sources_consulted** — list of sources queried (regardless of whether they
  yielded data)
- **fields_modified** — list of tensor fields changed, each with before/after
  values
- **conflicts_detected** — list of field-level conflicts between CV data and
  external signals, with resolution applied
- **resolution_applied** — which tier of the conflict hierarchy was invoked
  (see `merge-strategy.md`)
- **confidence_before** — aggregate tensor confidence before this run
- **confidence_after** — aggregate tensor confidence after this run

**Invariant:** one entry per run, even if the run produces zero changes. This
ensures the audit trail is complete and proves that enrichment was attempted.

## Candidate Notification

Configurable notification policy, set per candidate or as a system default:

- **First enrichment** — always notify when a candidate's profile is enriched
  for the first time
- **Conflict detected** — notify when external data conflicts with CV data,
  so the candidate can review
- **Significant confidence change** — notify when aggregate confidence shifts
  by more than 0.15 in either direction
- **Low-impact additive** — silent by default; new skills added from high-
  confidence sources without conflict do not trigger notification

Candidates can override notification settings to receive all notifications
or none.

## Data Retention

Three retention tiers, selectable per candidate:

| Tier | Enrichment Log | Source Data | Tensor Fields |
|------|---------------|-------------|---------------|
| **Standard** | Account lifetime + 90-day buffer after deletion | Account lifetime | Merged permanently |
| **Minimal** | 90 days rolling | 90 days rolling | Merged permanently |
| **Right-to-delete** | Purged on request | Purged on request | Reverted to CV-only tensor (re-run CV-to-Profile) |

Right-to-delete triggers a full tensor rebuild from the original CV, discarding
all enrichment contributions. The enrichment log entry for the deletion is
itself retained as a legal audit record (recording that deletion occurred, not
the deleted data).

## Regulatory Alignment

The enrichment privacy model maps to GDPR requirements:

- **Lawful basis** — legitimate interest (improving match quality); candidates
  are informed at signup and can object at any time
- **Transparency** — source disclosure records make all enrichment visible to
  the candidate
- **Right to object** — opt-out flags allow full or selective enrichment
  blocking
- **Right to erasure** — right-to-delete retention tier purges enrichment data
  and rebuilds the tensor from CV only
- **Audit trail** — the enrichment log provides a complete, timestamped record
  of all data processing decisions

---

## Cross-References

- Merge rules: `merge-strategy.md`
- Parent: `overview.md`
- Feedback data source: `learning/feedback.md`
