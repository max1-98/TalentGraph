# Co-Founder Matching — Gap-Emphasis Scoring

How scoring weights are adjusted when searching for co-founders: a Bayesian
posterior shift that emphasises team gaps while preserving domain knowledge
from per-vertical production weights.

---

## Problem

Standard scoring weights (w1–w5 for f1–f5) are optimised for recruiter
search: find candidates who match a job description. Co-founder search is
different — it needs candidates who fill the *team's gaps*, not candidates
who match a generic description. The weights must shift to emphasise gap-
related sub-scores without discarding the learned vertical-specific priors.

---

## Bayesian Weight Posterior Adjustment

Treat the production per-vertical weights as a prior and the gap profile as
evidence. The posterior weights combine domain knowledge with gap focus.

### Prior

The per-vertical weight vector W_prior = (w1, ..., w5) selected by the
query's industry-seniority cluster (see `scoring/overview.md`). These
weights encode learned domain knowledge (e.g. fintech searches weight
industry match higher).

The prior is modelled as a Dirichlet distribution:

W_prior ~ Dirichlet(alpha_prior * w_prior)

where alpha_prior = 0.6 controls prior strength.

### Likelihood — Gap Informativeness

For each sub-score f_i, compute a gap informativeness signal Gap_fi from the
team gap profile:

| Sub-Score | Gap Signal | Interpretation |
|-----------|-----------|----------------|
| f1 (Skill Coverage) | Gap_f1 = missing_skills / total_required | Fraction of required skills the team lacks |
| f2 (Semantic Relevance) | Gap_f2 = 1 - cosine(team_embedding, venture_embedding) | Semantic distance between team and venture |
| f3 (Skill Synergy) | Gap_f3 = missing_synergies / total_synergies | Fraction of required skill pairs the team cannot form |
| f4 (Career Trajectory) | Gap_f4 = EMD(team_trajectory, target_trajectory) | Earth mover's distance between team shape and target |
| f5 (Practical Fit) | Gap_f5 = practical_mismatch_ratio | Fraction of practical constraints not met |

### Posterior

The posterior weight for each sub-score:

w_i_posterior = (alpha_prior * w_i_prior + alpha_gap * Gap_fi) / Z

where alpha_gap = 0.4 controls gap evidence strength and
Z = SUM_j (alpha_prior * w_j_prior + alpha_gap * Gap_fj) is the
normalisation constant ensuring weights sum to 1.

**Effect:** sub-scores where the team has large gaps receive higher weights.
If the team has strong skills but weak industry fit, f4 (career trajectory)
and f2 (semantic relevance) are upweighted. If the team lacks specific
skills, f1 and f3 dominate.

---

## Pareto-Optimal Weight Perturbation

The Bayesian posterior gives a single adjusted weight vector. For richer
decision support, explore the Pareto frontier between gap coverage and
overall quality.

### Bi-Objective Formulation

- Objective 1: gap coverage — the degree to which the selected candidate
  fills team gaps (higher is better)
- Objective 2: overall quality — the standard composite score with
  production weights (higher is better)

These two objectives are generally in tension: a candidate who perfectly
fills gaps may be weaker overall, and vice versa.

### Linear Scalarisation

Define a family of weight vectors parameterised by gamma:

W_gamma = (1 - gamma) * W_standard + gamma * W_gap

where W_standard is the production per-vertical weight vector and
W_gap = (Gap_f1, ..., Gap_f5) / SUM Gap_fi is the normalised gap vector.

Evaluate the top-50 candidates at 5 gamma values: 0.0, 0.25, 0.5, 0.75,
1.0. At each gamma, compute the composite score and rank.

### Non-Dominated Candidates

A candidate is Pareto-non-dominated if no other candidate is better on both
objectives. Flag non-dominated candidates in the results — they represent
the efficient frontier of the gap-coverage vs quality tradeoff. The user
(or a downstream algorithm) decides where on the frontier to operate.

---

## Per-Vertical Interaction

The gap adjustment is applied *after* vertical weight selection, not instead
of it. The composition order:

1. Select W_prior from the per-vertical weight table based on the venture's
   industry-seniority cluster.
2. Compute Gap_fi from the team gap profile.
3. Apply Bayesian posterior to produce W_posterior.
4. Score candidates using W_posterior.

This preserves domain knowledge (e.g. "in fintech, industry experience
matters more") while layering gap focus on top. A fintech co-founder search
will still weight industry match higher than a generic search, but the gap
adjustment may further boost it if the team's industry gap is large.

---

## WeightProvider Decorator

Gap-emphasis scoring is implemented as a runtime decorator around the
standard WeightProvider, not a separate implementation.

The decorator intercepts `get_weights(query)`. If the query includes a team
gap profile, it applies the Bayesian posterior; otherwise it is a no-op.
Composes cleanly with A/B testing — the base WeightProvider may itself be an
A/B variant, and the gap-emphasis decorator sits on top.

| Parameter | Default | Tuning Strategy |
|-----------|---------|-----------------|
| alpha_prior | 0.6 | A/B test with [0.4, 0.6, 0.8] |
| alpha_gap | 0.4 | Complement of alpha_prior |
| Pareto gamma values | [0.0, 0.25, 0.5, 0.75, 1.0] | Fixed |

---

## Cross-References

- Co-Founder overview and team tensor: `cofounder-matching/overview.md`,
  `cofounder-matching/team-tensor.md`
- Scoring weight system: `scoring/overview.md`
- Fault tolerance: `algorithm/fault-tolerance.md`
