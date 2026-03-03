# Co-Founder Matching — Team Tensor Aggregation

How individual CandidateTensors are combined into a single team tensor that
represents the collective capabilities of an existing founding team. This
tensor becomes the "query" for co-founder search.

---

## Skill Dimensions — Choquet Integral

Simple aggregation (max, mean, sum) fails for skills: max ignores redundancy,
mean penalises specialisation, sum grows unboundedly with team size. The
Choquet integral provides non-additive aggregation using a fuzzy measure that
captures skill interactions.

### Fuzzy Measure (2-Additive)

A 2-additive fuzzy measure mu is defined by:

- Shapley values phi_i for each team member i, representing individual
  contribution to the team's skill portfolio.
- Interaction indices I_ij for each pair (i, j), representing synergy
  (positive) or redundancy (negative).

The measure for any subset A of team members:

mu(A) = SUM_{i in A} phi_i + SUM_{i<j, i,j in A} I_ij

### Choquet Integral Computation

For each skill dimension d, the Choquet integral aggregates team members'
proficiency values:

C_mu(x_d) = SUM_{i=1}^{n} (x_{sigma(i),d} - x_{sigma(i-1),d}) * mu(A_i)

where sigma orders team members by increasing proficiency in dimension d,
x_{sigma(0),d} = 0, and A_i = {sigma(i), ..., sigma(n)}.

### Interaction Index Calibration

I_ij measures skill overlap between members i and j:

I_ij = -epsilon * jaccard(skills_i, skills_j)

where epsilon = 0.1 controls redundancy penalty strength. Two members with
identical skill sets receive a redundancy penalty; two with disjoint skills
receive no interaction penalty. This naturally captures "the team has a
skill if any member has it" while penalising pure redundancy.

### Cold Start

When insufficient team interaction data exists to learn Shapley values,
fall back to simple centroid aggregation (element-wise mean) for skill
dimensions, which is the uniform-weight special case of the Choquet
integral.

---

## Embedding Dimensions — Multi-Head Attention Pooling

The four 384-dimensional embedding vectors per member (technical depth,
breadth coverage, recency-weighted, uniform) are aggregated using a Set
Transformer attention mechanism.

### Architecture

Four attention heads: (1) **technical depth** — attends to the most
specialised member; (2) **breadth coverage** — attends broadly across all
members; (3) **recency** — attends to members with the most recent skill
acquisition; (4) **uniform** — equal weights, equivalent to centroid pooling.
Each head produces a 96-dim output; concatenated to form 384-dim team
embedding. Cold start (< 3 members or untrained): simple centroid.

---

## Career Trajectory — Dual Representation

The 16-dimensional career trajectory is represented as two complementary
views:

### Centroid

Element-wise mean of all members' career vectors. Represents the average
team "shape" — typical seniority, typical industry breadth, typical
stability.

### Spread

Per-dimension range: spread_d = (max_d - min_d) / sqrt(team_size).

Normalisation by sqrt(team_size) prevents spread from growing unboundedly
with team size. A 4-person team with diverse backgrounds has meaningful
spread; a 20-person company does not have 5x the spread.

The spread vector is appended to the centroid, producing a 32-dimensional
career representation for the team tensor (vs 16 for an individual).

---

## Practical Fit Dimensions

Practical fit fields are aggregated by their natural semantics:

| Dimension | Aggregation | Rationale |
|-----------|-------------|-----------|
| Location | Convex hull of member locations | Team is "present" in the geographic area spanned by its members |
| Salary | Sum of individual expectations | Total team compensation budget |
| Notice period | Maximum across members | Team is available only when the last member can start |
| Employment type | Union of accepted types | Team accepts any arrangement that any member accepts |

---

## Size Normalisation

Choquet is bounded by construction; attention pooling is fixed-dimension;
career spread is normalised by sqrt(team_size); salary sum intentionally
grows (correct — larger teams cost more).

### Size Penalty

penalty = 1 / (1 + beta * max(0, size - target))

Beta = 0.3. Teams at or below target receive no penalty; teams above are
progressively penalised. This multiplies the final composite score.

---

## Trait Contract — TeamTensorAggregator

- **In:** list of CandidateTensors (team members), target team size
- **Out:** team tensor with skills (Choquet), embeddings (attention-pooled),
  career (centroid + spread), practical fit, and size penalty factor
- **Invariants:** skill values bounded within input range; embedding = 384-
  dim; career spread non-negative; size penalty in (0, 1]

---

## Cross-References

- Co-Founder overview and gap-emphasis: `cofounder-matching/overview.md`,
  `cofounder-matching/gap-emphasis.md`
- Team selection pattern: `features/team-selection.md`
- Candidate tensor schema: `core/types.md`
- Fault tolerance: `algorithm/fault-tolerance.md`
