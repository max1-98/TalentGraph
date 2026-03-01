# Testing Strategy

Three layers of testing plus shadow mode for zero-risk production experiments.
Trait boundaries make each layer possible.

---

## Layer 1: Unit Tests (per Scorer)

Each scorer is tested in isolation with synthetic candidate tensors. No
pipeline, no storage, no indexes. Pure function: tensor in, float out.

**What is tested:**

- **Edge cases** — empty skill vector, missing fields, extreme values
- **Monotonicity** — adding a relevant skill never decreases the score
- **Determinism** — same inputs always produce the same output
- **Bounds** — output is always finite and non-negative
- **Known scenarios** — a perfect-match candidate scores high, a zero-match
  candidate scores low

Synthetic tensors are built with a builder pattern that lets tests specify
only the fields relevant to that scorer, with sensible defaults for the rest.

## Layer 2: Integration Tests (per Stage)

Each stage is tested with test-double implementations of its dependencies:

- InMemoryTensorStore instead of memory-mapped files
- BruteForceIndex instead of HNSW (exact results, no approximation error)

**What is tested:**

- Correct candidate counts (e.g. filter eliminates the right candidates)
- Ordering stability (same inputs produce same output order)
- Contract adherence (output conforms to the StageOutput structure)
- Latency budgets (stage completes within its allocated time)

Because dependencies are injected via traits, integration tests require
zero infrastructure — no disk, no network, no external services.

## Layer 3: Regression Tests (full Pipeline)

A golden suite of 200 job descriptions with expected top-10 results. The
full pipeline runs end-to-end.

**What is tested:**

- Known-good candidates appear in the top 50 for their matching JDs
- NDCG@50 stays above a threshold (no ranking degradation)
- No version change is merged if regression metrics degrade

The golden suite is maintained as a test fixture. When the scoring logic
improves, the golden results are updated after manual review.

## Shadow Mode

A shadow pipeline runs alongside the primary pipeline in production. It
processes the same queries with a different configuration (e.g. new scorer,
updated weights) but its results are never served to users.

**How it works:**

1. Primary pipeline handles the request and returns results to the user
2. Shadow pipeline runs asynchronously on the same query (fire-and-forget)
3. A comparison logger records: ranking deltas, NDCG differences, candidate
   movements between primary and shadow results
4. Shadow results are never visible to users — zero risk

**Use cases:**

- Validate a new scorer before enabling it in production
- Test updated LTR weights before switching traffic
- Measure the impact of an embedding model upgrade

## Deployment Gate Sequence

Every change follows this sequence before reaching production:

1. Unit tests pass — scorer-level, synthetic data
2. Integration tests pass — stage-level, test doubles
3. Regression suite passes — NDCG@50 on golden queries does not degrade
4. Shadow deploy for one week — new config runs alongside production
5. Review shadow metrics — ranking changes, latency, edge cases
6. Canary deploy — 5 % of live traffic
7. Full rollout — 100 %. Previous version remains registered for instant
   rollback via config change

## Property-Based Testing

Scorers are additionally tested with randomly generated tensors (property
testing) to verify invariants hold across the input space, not just for
hand-crafted examples.
