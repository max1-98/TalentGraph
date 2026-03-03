# Career Pathing — Time Estimation

How time-to-close-gaps is estimated for each career path: survival analysis
as the primary model, with Bayesian confidence intervals and an ensemble
fallback for large datasets. The event modelled is "candidate closes enough
gaps to be competitive for the target role."

---

## Primary Model — Cox Proportional Hazards

h(t | X) = h_0(t) * exp(beta^T X)

where h(t | X) is the hazard rate (instantaneous probability of transition
at time t given covariates X), h_0(t) is the baseline hazard (common to all
candidates), and beta is the learned coefficient vector.

### Covariates (X)

| Covariate | Source | Rationale |
|-----------|--------|-----------|
| Gap count | Gap Finder output | More gaps = longer time |
| Sum of proficiency deltas | Gap Finder output | Larger total deficit = longer time |
| EMD distance (f1) | ProfileComparator | Aggregate skill distribution distance |
| Current velocity | Career vector dim 10 | Fast movers close gaps faster |
| Career stage | Peer similarity stage ID | Stage constrains transition speed |
| Market demand score | MarketSignalProvider | High-demand skills have more learning resources |
| Specialisation index | Career vector dim 12 | Specialists take longer for breadth moves |

### Baseline Hazard — Breslow Estimator

The Breslow estimator provides a nonparametric step-function for h_0(t).
At each observed transition time t_j:

H_0(t) = SUM_{j: t_j <= t} d_j / SUM_{i in R_j} exp(beta^T X_i)

where d_j is the number of events at time t_j and R_j is the risk set
(candidates still active at t_j). The survival function follows:

S(t | X) = exp(-H_0(t) * exp(beta^T X))

### Censored Observations

Candidates who have not yet transitioned contribute to the likelihood as
right-censored observations. They inform the risk set without being treated
as "never transitioning." This is critical for avoiding bias toward fast
movers.

---

## Confidence Intervals — Bayesian Bootstrap

Point estimates without uncertainty are misleading. The system provides
confidence bands via Bayesian bootstrap of the Breslow estimator.

### Procedure

1. Draw B = 500 bootstrap samples of the training data (with Dirichlet-
   distributed weights — the Bayesian bootstrap, not the classical one).
2. For each sample, fit beta and compute H_0(t).
3. For a given candidate's covariates X, compute S(t | X) for each
   bootstrap sample.
4. Point estimate: median survival time (the t at which the median S(t)
   crosses 0.5).
5. Confidence band: 10th and 90th percentile survival times across the
   500 samples.

### Small-Sample Fallback — Weibull Regression

When fewer than 200 transition observations are available (e.g. niche
career paths), the Cox model's nonparametric baseline is unreliable. Fall
back to Weibull parametric regression:

h(t | X) = (k / lambda) * (t / lambda)^(k-1) * exp(beta^T X)

with weakly informative priors on k (shape) and lambda (scale):
k ~ Gamma(2, 1), lambda ~ LogNormal(3, 1). These priors encode a belief
that most transitions take 1-5 years and the hazard is unimodal.

---

## Ensemble — Quantile Regression Forest

When 500 or more transitions are available, a Quantile Regression Forest
(QRF) is trained alongside the Cox model. QRF directly predicts quantiles
of the time distribution without assuming a proportional hazards structure.

### Combination Rule

Final time estimate = w_cox * T_cox + w_qrf * T_qrf

where T_cox and T_qrf are the median predictions from each model. Weights
are set by Brier score calibration on a held-out validation set:

w_i = (1 / BS_i) / SUM_j (1 / BS_j)

The Brier score measures calibration: how well predicted probabilities match
observed frequencies at each time horizon. The better-calibrated model
receives higher weight.

---

## Per-Gap Time Decomposition

The overall time estimate covers the full transition. For actionable
planning, the system decomposes time into per-gap contributions.

### Additive Hazard Decomposition

Model the total hazard as a sum of per-gap hazards:

h_total(t | X) = SUM_g h_g(t | X_g)

where each gap g contributes proportionally to its proficiency delta times
its importance weight:

h_g(t | X_g) proportional to delta_g * importance_g

### Gap Dependency Ordering

Gaps are not independent — some must be closed before others (e.g. learning
a framework requires knowing the underlying language). The bridge suggestion
ordering from Gap Finder (`gap-finder/bridge-suggestions.md`) defines a
partial order. Time estimates for downstream gaps are conditional on upstream
gaps being closed first, adding their estimated durations sequentially along
each dependency chain.

---

## Trait Contract — TimeEstimator

- **In:** CandidateTensor, gap list (from Gap Finder), path destination
- **Out:** overall time estimate (median, 10th/90th percentiles), per-gap
  time contributions, confidence indicator (high/medium/low by sample size)
- **Invariants:** non-negative estimates; per-gap times sum to overall
  estimate; confidence bands contain the point estimate

---

## Cross-References

- Career Pathing overview and peer similarity: `career-pathing/overview.md`,
  `career-pathing/peer-similarity.md`
- Gap Finder contracts and bridge suggestions: `gap-finder/contracts.md`,
  `gap-finder/bridge-suggestions.md`
- Fault tolerance: `algorithm/fault-tolerance.md`
