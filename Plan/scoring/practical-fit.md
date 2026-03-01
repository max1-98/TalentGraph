# f5 — Practical Fit Score

Evaluates logistical compatibility — can this candidate actually take this role
given real-world constraints?

---

## Formula

f5(c, q) = location_factor * salary_factor * notice_factor * type_factor

The score is a product of smooth penalty functions, each in the 0–1 range.
A perfect practical fit yields 1.0. Each constraint applies a multiplicative
decay rather than a hard cutoff.

## Components

### Location Decay (Gaussian)

location_factor = exp( -0.5 * (distance / max_radius)^2 )

- distance = great-circle distance between candidate and query locations
  (Haversine formula)
- max_radius = the query's specified location radius in kilometres
- A candidate 5 km outside the radius is gently penalised, not eliminated
- If no location constraint is specified, factor = 1.0

### Salary Overlap

salary_factor = overlap(candidate_range, budget_range)

- Computes the ratio of overlap between the candidate's salary expectations
  and the employer's budget
- 0.0 = no overlap at all, 1.0 = candidate range fully within budget
- Partial overlaps produce intermediate values

### Notice Period (Sigmoid)

notice_factor = 1 / (1 + exp((notice_days - max_notice) / 15))

- Smooth sigmoid centred on the query's maximum notice period
- Candidates exactly at the boundary score ~0.5
- Beyond the boundary, penalty increases gradually (15-day half-life)
- If no notice constraint is specified, factor = 1.0

### Employment Type Match

type_factor = 1.0 if any overlap in employment type bitflags, else 0.1

- Bitwise AND between candidate's accepted types and query's required types
- Mismatch applies a heavy penalty (0.1) but does not zero out the score —
  the recruiter can still see the candidate and decide

## Design Rationale — Smooth Penalties

Previous approaches used hard SQL WHERE clauses: candidate outside the radius
was invisible. This fails because recruiter requirements are usually soft
preferences. A candidate 5 km over the radius or 5 days past the notice limit
may still be the best match.

Smooth decay functions (Gaussians, sigmoids) let near-miss candidates remain
visible with a proportional penalty. The hard pre-filter stage (Stage 1) still
eliminates truly impossible matches — but with wider bounds than the scorer
uses, ensuring no viable candidate is lost.

## Edge Cases

- **Remote role with no location constraint:** location_factor = 1.0 for all
  candidates; the term has no effect
- **Candidate with open salary range (0–max):** full overlap with any budget
- **Notice period of zero:** always scores 1.0 on the notice factor
- **No employment type overlap:** heavy penalty (0.1) but not zero — allows
  the score to remain visible in edge cases where the candidate might
  reconsider their employment type preference

## Relationship to Stage 1

Stage 1 (bitwise pre-filter) uses the same constraints but with wider bounds
and integer comparisons. f5 then applies exact, continuous-valued penalties to
the survivors. The two-tier approach preserves both speed (Stage 1) and
precision (f5).
