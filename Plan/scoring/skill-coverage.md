# f1 — Skill Coverage Score

Measures what percentage of the required skills a candidate possesses, weighted
by the importance the query assigns to each skill.

---

## Formula

f1(c, q) = SUM_{s in R}( q_s * min(c_s, 1) ) / SUM_{s in R}( q_s )

- R = set of skills in the query (required + preferred)
- q_s = importance weight of skill s in the query
  - required = 1.0, preferred = 0.5, nice-to-have = 0.2
- c_s = candidate's score for skill s (from the skill vector — already
  incorporates proficiency, recency, depth, IDF)
- min(c_s, 1) = caps individual skill contributions so that being
  over-qualified in one skill does not compensate for missing another

## Inputs

- **Candidate skill vector** (sparse, ~50–200 non-zero entries over ~5 000
  dimensions). Each entry is a continuous proficiency signal computed as:
  S(skill, candidate) = alpha * P * exp(-lambda * t) * log(1 + d) * w_ctx
  where P = normalised proficiency, t = years since last use, d = duration,
  w_ctx = context weight, alpha = IDF weight
- **Query skill set** with per-skill importance weights

## Algorithm

1. Initialise weighted_hits = 0, total_weight = 0
2. For each required skill: accumulate importance * min(candidate_score, 1)
3. For each preferred skill: add importance * min(candidate_score, 1) * 0.3
   (preferred skills contribute 30 % uplift without diluting the required
   denominator)
4. Divide accumulated hits by total weight

## Edge Cases

- **Candidate has zero required skills:** score approaches zero
- **Candidate has all required skills at maximum:** score approaches 1.0
  (preferred skills add a small bonus above the required baseline)
- **Query has no required skills (only preferred):** total weight is the
  sum of preferred importance * 0.3; the score still normalises correctly
- **IDF effect:** a candidate with a rare required skill (e.g. VHDL)
  receives a higher c_s than one with a ubiquitous skill (e.g. Python)
  because alpha = log(N / df)

## Why min(c_s, 1)?

Without the cap, a candidate scored 5.0 on a single skill could mask gaps
in other skills. The cap ensures each skill contributes at most its query
importance weight, making the score a true coverage measure.

## Why Max-Pooling in the Skill Vector?

If a candidate used Python in three projects, the skill vector keeps only
the single best signal — not the sum. This prevents verbosity bias: a
candidate who mentions a skill once in a deeply relevant context scores
equally to someone who mentions it ten times in shallow contexts.
