# Scoring — Overview

The scoring function combines multiple independent sub-scores into a single
composite value. It is fully differentiable so that weights can be learned from
outcome data.

---

## Composite Formula

Score(c, q) = SUM_i( w_i * sigma(f_i(c, q)) ) + b

- c = candidate tensor
- q = query tensor
- f_i = the i-th sub-score function
- sigma = sigmoid normalisation (maps each raw sub-score to the 0–1 range)
- w_i = learned weight for the i-th sub-score
- b = bias term

Each sub-score is computed independently, normalised, then combined via a
weighted sum. The sigmoid prevents any single sub-score from dominating through
an extreme raw value.

## Sub-Score Inventory

| ID | Name | What It Measures |
|----|------|------------------|
| f1 | Skill Coverage | Weighted hit rate across required skills |
| f2 | Semantic Relevance | Embedding similarity from multiple perspectives |
| f3 | Skill Synergy | Co-occurrence of skills in shared contexts |
| f4 | Career Trajectory | Seniority fit, industry alignment, stability |
| f5 | Practical Fit | Location, salary, notice period, employment type |

Additional scorers (e.g. confidence-adjusted, market scarcity, Mahalanobis
distance) can be registered without modifying existing scorers — the registry
and weight vector grow by one dimension.

## Weight System

Weights are not hardcoded. They progress through three phases:

1. **Expert priors** — hand-tuned defaults (day one)
2. **Bayesian update** — refined from implicit signals (months one to three)
3. **Full LTR** — pairwise LambdaRank training (month three onward)

Weights are scoped per industry-by-seniority vertical, so a fintech search
may weight industry match higher than a startup search does.

## Per-Vertical Weights

A weight set is selected based on the query's cluster (industry + seniority
bucket). If the cluster has insufficient outcome data, it falls back to the
global weight set with Bayesian regularisation.

## Score Assembly

1. Compute each f_i independently (parallelisable across scorers)
2. Apply sigmoid normalisation to each raw score
3. Multiply each normalised score by its weight
4. Sum and add bias
5. Attach the full breakdown (all normalised sub-scores) to the result

The breakdown is the product — it powers the feedback loop, recruiter
explainability, and user intelligence features.

## Design Principles

- **Smooth penalties, not hard filters.** Gaussians and sigmoids replace
  boolean cutoffs so that near-miss candidates are penalised gently rather
  than eliminated. Truly impossible matches are caught by the pipeline's
  pre-filter stage, which uses wider bounds than the scorer.
- **Scorer independence.** No scorer reads another scorer's output. Each
  depends only on the candidate tensor and the query context.
- **Extensibility.** Adding a scorer is a single isolated change: implement
  the Scorer trait, register it, add it to the config. Zero modifications
  to existing scorers or the pipeline.

## Gap-Emphasis Weight Adjustment

For co-founder matching (`algorithm/cofounder-matching/gap-emphasis.md`), the
standard per-vertical weights are adjusted at runtime via a Bayesian posterior
that shifts emphasis toward sub-scores where the team has the largest gaps.
This is implemented as a WeightProvider decorator — the core weight system
and per-vertical selection are unchanged.

## Relationship to Algorithm Layer

The sub-scores f1–f5 operate on candidate-vs-query pairs in the real-time
scoring pipeline. The algorithm layer's **ProfileComparator** building block
(see `algorithm/overview.md`) generalises these same dimensions into structured
diffs between any two profiles or a profile and a role. This enables gap
analysis (see `algorithm/gap-finder/overview.md`), co-founder complementarity
scoring, and career path evaluation — all reusing the scoring logic in an
offline, richer context where latency constraints are relaxed.
