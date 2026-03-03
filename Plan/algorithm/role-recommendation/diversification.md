# Role Recommendation — Diversification

How the top-N role recommendations are diversified to avoid surfacing
near-identical positions. After scoring, results are often clustered;
diversification balances relevance against variety across employers,
industries, and seniority levels.

---

## Determinantal Point Process (DPP)

A DPP defines a probability distribution over subsets of items where
high-quality, diverse subsets are most probable. Selection from a DPP
naturally balances quality and diversity.

### Kernel Construction

The DPP L-matrix (positive semi-definite) is constructed as:

L_ij = q_i * S_ij * q_j

where:
- q_i = exp(alpha * score_i) is the quality term for role i. score_i is the
  composite score from the scoring pipeline.
- S_ij is the similarity kernel between roles i and j.
- alpha controls the quality-diversity tradeoff. Higher alpha favours
  quality; lower alpha favours diversity. Initial setting: alpha = 1.0,
  tuned via A/B testing.

### Similarity Kernel

S_ij is a weighted combination of three similarity signals:

| Signal | Weight | Computation |
|--------|--------|-------------|
| Employer similarity | 0.4 | 1.0 if same employer, 0.5 if same parent company, 0.0 otherwise |
| Skill Jaccard | 0.3 | Jaccard index over required skill sets of the two roles |
| Embedding cosine | 0.3 | Cosine similarity between the 384-dim role embeddings |

S_ij = 0.4 * employer_sim(i, j) + 0.3 * jaccard(i, j) + 0.3 * cosine(i, j)

Employer similarity receives the highest weight because showing multiple
roles from the same company is the most visually redundant outcome.

### Greedy MAP Inference

Exact MAP inference for DPPs is NP-hard. The system uses the greedy
algorithm of Chen et al.: iteratively select the item that maximises the
log-determinant of the selected sub-matrix. This runs in O(N * k^2) where
k is the selected subset size and N is the candidate pool (typically 50-100
top-scored roles).

The greedy solution is initialised with the MMR-selected subset (see below)
for faster convergence.

---

## Cluster Hard Constraints

DPPs provide soft diversity — they make homogeneous subsets unlikely but not
impossible. Hard constraints enforce minimum variety.

### Cluster Definition

Each role is assigned to a cluster defined by the triple:
(employer, industry, seniority band).

Seniority bands: junior, mid, senior, executive (4 levels).

### Constraint Rule

No single cluster may occupy more than ceil(N / 3) slots in the final
result set of size N. If the DPP-selected subset violates this constraint:

1. Identify over-represented clusters.
2. Remove the lowest-scoring role from each over-represented cluster.
3. Replace with the highest-scoring role from the most under-represented
   cluster that was not already selected.
4. Repeat until the constraint is satisfied.

This post-processing typically changes 0-2 items in the result set.

---

## MMR Fallback

Maximal Marginal Relevance serves two purposes: fallback when the DPP
computation is unavailable, and initialisation for the greedy DPP solver.

MMR(r) = lambda * score(r) - (1 - lambda) * max_{s in Selected} sim(r, s)

where sim uses the same S_ij kernel as the DPP. Lambda = 0.7.

### Selection Procedure

1. Select the highest-scoring role as the first item.
2. Iteratively select the role with the highest MMR score until N items are
   chosen.
3. Apply the same cluster hard constraints as the DPP path.

MMR runs in O(N * k) — faster than the greedy DPP — and produces results
that are typically within 5-10% of the DPP solution's diversity score.

---

## Evaluation

Diversification quality is measured offline and in A/B tests.

### Offline Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Intra-List Diversity (ILD) | Average pairwise dissimilarity across the result set: ILD = (2 / (N*(N-1))) * SUM_{i<j} (1 - S_ij) | >= 0.6 |
| alpha-nDCG | nDCG variant that penalises redundant items at lower ranks. Alpha parameter controls redundancy penalty (alpha = 0.5). | >= 0.85 relative to undiversified nDCG |
| Cluster entropy | Shannon entropy over the cluster distribution in the result set | >= log(3) (at least 3 effective clusters) |

### A/B Testing Protocol

Compare diversified vs undiversified results on:
- Click-through rate (primary): does diversity increase engagement?
- Application rate: do diverse results lead to more applications?
- Return rate: do users come back more often with diversified results?

Alpha is tuned based on A/B results: increase if CTR improves with more
diversity, decrease if relevance drops too much.

---

## Trait Contract — ResultDiversifier

- **In:** ranked list of (role, score) pairs, target result size N
- **Out:** diversified list of N (role, score) pairs with diversity metadata
- **Invariants:** output size equals N (or input size if smaller); cluster
  constraint satisfied; all returned roles appeared in the input list

---

## Cross-References

- Role Recommendation overview and compilation: `role-recommendation/overview.md`,
  `role-recommendation/compilation.md`
- Scoring pipeline: `scoring/overview.md`
- Fault tolerance: `algorithm/fault-tolerance.md`
