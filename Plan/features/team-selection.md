# Team Selection — Multi-Vacancy Optimisation

When a search specifies multiple vacancies, the system selects a team of
candidates that maximises collective coverage rather than simply taking the
top N individuals.

---

## Problem

Given K vacancies and a ranked list of candidates, selecting the top-K
individually often produces a redundant team. If the top three candidates
all specialise in the same area, the team has a critical gap. Multi-vacancy
optimisation selects candidates whose skills *complement* each other.

## Approach: Greedy Submodular Optimisation

Skill coverage across a team is a submodular function — adding a candidate
has diminishing marginal returns as the team's collective coverage grows.
The greedy algorithm exploits this property:

1. Start with an empty team
2. For each vacancy:
   a. Evaluate every remaining candidate's *marginal contribution* — how
      much new coverage they add beyond what the current team already covers
   b. Select the candidate with the highest marginal contribution
3. Repeat until all vacancies are filled

This greedy approach achieves a (1 - 1/e) approximation ratio (~63 %) to
the optimal solution — a well-known guarantee for submodular maximisation.

## Marginal Contribution

The marginal contribution of a candidate considers:

- **Skill coverage delta** — which required skills does this candidate add
  that the current team lacks?
- **Embedding diversity** — how different is this candidate's embedding from
  the current team centroid? (prevents clustering)
- **Career trajectory diversity** — does this candidate bring a different
  seniority, industry, or specialisation profile?

The exact combination of these factors is weighted by the query's requirements
and the active weight set.

## Input / Output

- **In:** scored candidate list from Stage 3 (up to 250), requirement list,
  number of vacancies, shortlist size per vacancy
- **Out:** a set of seat assignments — each vacancy mapped to a shortlist of
  candidates optimised for team-level coverage

## Alternative Implementations

| Strategy | Description | Status |
|----------|-------------|--------|
| Greedy Submodular | (1 - 1/e) approximation, fast | Current |
| Top-N Split | Simply split top-N evenly across vacancies | Baseline (no diversity) |
| Integer Linear Programming | Exact optimisation (slow for large inputs) | Future |
| Genetic Algorithm | Evolutionary search (experimental) | Future |

All implementations share the TeamSelector trait — swappable via config.

## When It Activates

Team selection only runs when the query specifies vacancies > 1. For single-
vacancy searches, this stage is a pass-through (or skipped entirely).

## Edge Cases

- **Vacancies exceed candidate pool:** fill as many seats as possible,
  leave the rest empty
- **All candidates are identical in skill profile:** degrades to top-N
  (no diversity to exploit)
- **Single highly specialised vacancy:** marginal contribution is dominated
  by the skill coverage delta for that specialisation
