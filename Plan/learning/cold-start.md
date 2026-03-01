# Cold Start Strategy

How the system operates before enough outcome data has accumulated to train
learned weights.

---

## The Problem

The scoring function uses learned weights. But on day one, there are zero
outcomes — no hires, no shortlists, no feedback. The system must produce
useful results from the start while building toward data-driven optimisation.

## Three-Phase Ramp

### Phase 1: Expert Priors (Day 1)

Hand-tuned weights from recruiting domain experts:

| Weight | Value | Rationale |
|--------|-------|-----------|
| w_1 (skill coverage) | 0.30 | Most directly answers "can they do the job?" |
| w_2 (semantic relevance) | 0.25 | Captures implicit fit beyond explicit skills |
| w_3 (skill synergy) | 0.20 | Differentiator: skills used together matter |
| w_4 (career trajectory) | 0.15 | Progression signals predict longevity |
| w_5 (practical fit) | 0.10 | Logistics — important but not dominant |

These are reasonable defaults that produce acceptable results. The system is
usable on day one.

### Phase 2: Bayesian Update (Months 1–3)

As implicit signals accumulate (profile views, click-through rates, time
spent on profiles), the system performs Bayesian updating of the prior
weights.

- Small, conservative updates — high regularisation prevents overfitting
  on sparse data
- The prior acts as an anchor: weights shift gradually from the expert
  defaults rather than being replaced wholesale
- Implicit signals only (views, clicks) — explicit outcomes (hires) are
  too sparse at this stage to be reliable

### Phase 3: Full LTR (Month 3+)

With ~500 or more searches that have resulted in hiring outcomes, the system
switches to full LambdaRank training (see `learning/ltr.md`).

- Per-vertical weight specialisation begins (industry x seniority buckets)
- Continuous A/B testing of learned weights against prior-based weights
- The transition is gradual: A/B test traffic starts at 10 % for learned
  weights, scaling up as confidence builds

## Vertical-Level Cold Start

Even after the global weights are well-trained, entering a new industry
vertical restarts the cold-start cycle at the vertical level:

1. New vertical inherits global weights as its prior
2. Bayesian updating from the first signals in that vertical
3. Full per-vertical training once enough data accumulates

This means the system is never fully "cold" — global weights provide a
floor, and vertical specialisation layers on top.

## Monitoring During Ramp

- Track NDCG and recruiter satisfaction metrics from day one
- Compare against a baseline (random ranking, keyword matching) to
  validate that even prior-based weights outperform alternatives
- Alert if learned weights produce significantly worse outcomes than priors
  (triggering automatic rollback to the previous weight set)

## Rollback

At any phase, the system can instantly revert to the previous weight set
via a configuration change (see `infrastructure/deployment.md`). Both the
prior weights and the latest learned weights are registered in the weight
provider — switching between them requires no recompilation.
