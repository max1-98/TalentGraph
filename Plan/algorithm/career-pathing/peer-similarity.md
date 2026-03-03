# Career Pathing — Peer Similarity

How trajectory peers are found: a learned distance metric over the 16-dim
career vector (see `scoring/career-trajectory.md`), stage-gated retrieval,
and diversity-aware selection.

---

## Distance Metric — Learned Mahalanobis

Standard Euclidean distance treats all 16 dimensions equally and ignores
correlations. A Mahalanobis distance with a learned metric matrix captures
which dimensions matter most for "similar career outcome."

d(x, y) = sqrt( (x - y)^T M (x - y) )

where M is a 16x16 positive semi-definite matrix.

### Learning M via Large Margin Nearest Neighbour (LMNN)

Training data: triplets (anchor, positive, negative) derived from career
outcome labels. Two candidates are "positive" if they share a career outcome
cluster (e.g. both transitioned into engineering management within 2 years).
"Negative" pairs are drawn from different outcome clusters.

LMNN objective: pull positives closer, push negatives beyond a margin. This
produces M that compresses within-outcome distances and expands between-
outcome distances.

**Cold start:** before sufficient outcome data exists, M is initialised as the
inverse covariance matrix of the career vector corpus. This Mahalanobis
distance already accounts for dimension correlations and scales — a
reasonable prior that is smoothly replaced as LMNN training data accumulates.

### ANN Compatibility via Factorisation

M = L^T L (Cholesky-like factorisation). Pre-transform all career vectors
by L: z = L * x. In the transformed space, Mahalanobis distance equals
standard Euclidean distance: d(x, y) = ||z_x - z_y||_2. This enables
standard HNSW indexing (likely hnswlib or similar — unconfirmed) over
transformed vectors with no custom distance function.

The L-transformed vectors are stored alongside the raw vectors and
recomputed whenever M is retrained.

---

## Career Stage Gating

Peer search is restricted by career stage to prevent comparing a junior
engineer to a C-suite executive, even if their raw vectors happen to be
geometrically close on non-seniority dimensions.

### Stage Definition

K-means clustering (k = 5) over three features: seniority level (dimension
1), years of experience (dimension 2), and leadership onset (dimension 7).
The five clusters map roughly to: early-career, developing, mid-career,
senior, and executive. Cluster boundaries are recomputed quarterly from
the full candidate corpus.

### Gating Rule

A peer candidate must be in the same stage or an adjacent stage (+/- 1).
This is implemented as a bitmap filter on the HNSW index: each candidate
is tagged with their stage ID, and the ANN query includes a filter that
accepts only stages within the allowed range.

---

## Peer Retrieval

Target: k = 50 trajectory peers per candidate.

### Two-Stage Retrieval

1. **ANN pre-filter.** Query the HNSW index over L-transformed vectors with
   the stage bitmap filter. Retrieve top-100 approximate neighbours. This
   runs in approximately 1-2 ms for a corpus of 1M candidates.

2. **Exact Mahalanobis reranking.** Compute exact d(x, y) for the 100
   candidates using the full M matrix (not the ANN approximation). Rerank
   by exact distance. This corrects ANN approximation errors, especially
   near decision boundaries.

### Diversity Pass — Maximal Marginal Relevance (MMR)

After exact reranking, apply MMR to select the final 50 peers from the 100:

MMR(p) = lambda * (1 / d(x, p)) - (1 - lambda) * max_j (1 / d(p, s_j))

where s_j are already-selected peers. Lambda = 0.7 (favouring relevance over
diversity). This prevents all 50 peers from clustering in the same industry or
company, which would bias the destination distribution.

---

## Fisher-Rao Refinement

For downstream path ranking (not for peer retrieval itself), model each
career trajectory as a point on a statistical manifold.

### Intuition

Two candidates may have identical current career vectors but arrived there
via very different paths — one steady, one volatile. The Fisher-Rao distance
captures trajectory *variability*, not just current position.

### Construction

Model each candidate's career history as a parametric distribution (Gaussian
over the sequence of career vectors observed at each career stage). The
Fisher information matrix G(theta) defines a Riemannian metric on the
parameter space. The geodesic distance under this metric is the Fisher-Rao
distance.

For Gaussian families, the Fisher-Rao distance has a closed-form expression
involving the means and covariance matrices of the two distributions.

### Application

Fisher-Rao distance is computed only for the 50 selected peers (not at ANN
scale). It is used as a secondary signal in the PathRanker: peers with
similar Fisher-Rao distance to the target candidate provide stronger
trajectory evidence than peers who merely share the same current position.

---

## Trait Contract — TrajectoryPeerFinder

- **In:** CandidateTensor (16-dim career vector), optional stage override,
  optional industry filter
- **Out:** ordered list of up to 50 peer CandidateTensors with Mahalanobis
  and Fisher-Rao distances
- **Invariants:** peers within +/- 1 career stage; no single industry
  exceeds 40% of results; distances non-negative and monotonically ordered

---

## Cross-References

- Career trajectory vector: `scoring/career-trajectory.md`
- Career Pathing overview and time estimation: `career-pathing/overview.md`,
  `career-pathing/time-estimation.md`
- Fault tolerance: `algorithm/fault-tolerance.md`
