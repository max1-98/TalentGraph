# Stage 3 — Exact Scoring (Precision)

Loads full candidate tensors for the ~2K survivors and computes all sub-scores
to produce a precise, explainable ranking.

---

## Purpose

Apply the complete scoring function (f1 through fN) to a manageable shortlist,
producing the final ranked output with per-candidate score breakdowns.

## Input / Output

- **In:** ~2K candidate IDs from Stage 2
- **Out:** 250 ranked results, each with a composite score and full breakdown

## Algorithm

1. Load candidate tensors from the tensor store (zero-copy memory-mapped
   access — no serialisation cost)
2. For each candidate (parallelised across CPU cores):
   a. Run every active scorer independently → vector of raw floats
   b. Apply sigmoid normalisation to each raw score
   c. Multiply by the corresponding weight and sum → composite score
   d. Attach the full ScoreBreakdown (all normalised sub-scores)
3. Sort by composite score descending
4. Truncate to the configured limit (default 250)

## Scorer Execution

The precision stage retrieves the active scorers from the scorer registry
(see `infrastructure/composition.md`) and runs them via the Scorer trait. It
does not know which scorers are active or how many there are — this is
entirely config-driven.

Each scorer is independent: no scorer reads another scorer's output. This
enables safe parallelism — all scorers for a single candidate can run
concurrently, and all candidates can be scored concurrently.

## Weight Application

The weight provider (see `core/traits.md`) supplies a weight set scoped to
the query's cluster (industry x seniority bucket). The precision stage
multiplies each normalised sub-score by its weight, sums, and adds the bias
term.

## Score Breakdown

Every scored candidate retains a full breakdown: one normalised value per
active scorer, plus the composite total. This breakdown is not discarded — it
flows downstream to:

- **Recruiter UI** — "this candidate scored 92 % on skill coverage but 31 %
  on skill synergy"
- **Feedback loop** — the learning-to-rank system trains on these breakdowns
- **User intelligence** — aggregated across searches for career insights

## Multi-Vacancy Extension

When the query specifies multiple vacancies, a TeamSelector stage follows
precision scoring. It reads the scored candidate list and optimises team-level
coverage (see `features/team-selection.md`). The precision stage itself is
unaware of multi-vacancy logic.

## Latency

- ~10 ms for 2K candidates at 1M scale
- Most compute-intensive stage, but operates on only ~0.2 % of the universe
- Parallelised across CPU cores (likely Rayon-style work-stealing)
- SIMD-friendly floating-point math on fixed-size tensor fields

## Why Not Score All Candidates?

Computing f1–fN on 1M tensors would take ~5 seconds. The three-stage funnel
ensures that the expensive exact scoring runs on only ~2K candidates, keeping
total latency under 20 ms.
