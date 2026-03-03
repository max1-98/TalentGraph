# Core Types — Data Structures

The shared data structures that flow between all components. Defined alongside
the traits in the core module so every other module can import them without
pulling in implementation dependencies.

---

## CandidateTensor

A fixed-size, dense numerical representation of one candidate. Pre-computed from
raw profile data and stored for fast scoring. Approximate size: ~4 KB per
candidate (1M candidates fits in ~4 GB of RAM).

### Fields

**Identity (filterable)**
- `user_id` — unique identifier (128-bit)
- `location` — latitude and longitude (two floats)
- `notice_days` — notice period in days
- `salary_min`, `salary_max` — salary range
- `employment_types` — bitflags (permanent, contract, freelance, part-time)
- `open_to_work` — boolean availability signal
- `years_experience` — total years
- `last_active` — days since last profile update

**Skill Vector**
- Sparse vector over a global skill taxonomy (~5 000 dimensions)
- Each entry encodes proficiency, recency, depth, and context weight in a
  single continuous float using the formula:
  S(skill, candidate) = alpha * P * exp(-lambda * t) * log(1 + d) * w_ctx
- Alpha is the IDF weight: log(N / df(skill)), making rare skills score higher
- Max-pooled across all occurrences — quality over quantity

**Semantic Embeddings (four per candidate, each 384-dim)**
- `embed_overall` — weighted mean of all chunk embeddings
- `embed_recent` — weighted mean of chunks from last three years
- `embed_deepest` — mean of top-five highest-weight chunks
- `embed_skills` — mean of skill-context chunk embeddings

**Career Shape**
- `career_trajectory` — 16-dimensional encoded vector covering seniority slope,
  scope slope, tech diversity rate, industry transitions, average tenure,
  tenure variance, recent momentum, gap months, specialisation index
  (Herfindahl), leadership onset, current velocity (3-dim PCA of skill vector
  delta), company tier trend, education recency
- Scalar scores: seniority (0–1), leadership, stability, breadth, depth

**Industry Vector** — sparse, weighted by recency and duration

**Precomputed Signals** — chunk count (richness proxy), confidence (0–1),
last recomputed timestamp

## FilterPack

A 32-byte packed struct used by Stage 1 for SIMD-friendly bitwise filtering.
Contains quantised location, salary range, notice days, bitflags, years of
experience, and bloom filters for skills and industries. Aligned to one cache
line for maximum throughput.

## QueryTensor

The structured output of query compilation. Mirrors the candidate tensor's
dimensions so scoring reduces to pairwise operations.

- Required skills with importance weights (required = 1.0, preferred = 0.5,
  nice-to-have = 0.2)
- Query embedding (384-dim, from the job description text)
- Target seniority level, industry vector, location + radius, salary budget
  range, maximum notice period, acceptable employment types
- Feature flags and search configuration

## ScoreBreakdown

A per-candidate record of every sub-score dimension, kept alongside the final
composite score. Structure: one float per active scorer (f1 through fN), plus
the composite total. Used for:

1. **Feedback loop** — the learning-to-rank system trains on breakdowns
2. **User intelligence** — aggregated across searches to produce career insights
3. **Recruiter explainability** — shows why a candidate ranked where they did

## WeightSet

The vector of learned (or hand-tuned) parameters consumed by the scoring stage:
- Per-scorer weights (w1 through wN) and a bias term
- Embedding perspective scaling factors (beta1 through beta4)
- Decay and penalty curve parameters (recency lambda, seniority sigma, etc.)
- Scoped by query cluster: different verticals may use different weight sets

---

## Relationship to the Shared Schema

All types in this file are **Tier 1 — engine-internal**. They are optimised for
memory layout and compute speed and never cross a service boundary directly.
Types that cross the API-to-frontend or API-to-engine boundary are defined in
`schema/` (see `schema/overview.md`). The engine implements mapping functions
between Tier 1 types here and the Tier 2/3 projections in the shared schema.

## Data Origin and Evolution

CandidateTensor is not created in one step — it evolves through the algorithm
layer:

1. **CV-to-Profile** (`algorithm/cv-to-profile/overview.md`) creates the
   initial tensor from an unstructured CV
2. **Profile Enrichment** (`algorithm/profile-enrichment/overview.md`) augments
   it with external signals, adding skills and raising confidence
3. The **confidence** field (0–1) reflects the tensor's extraction and
   enrichment state — low after a sparse CV parse, higher after corroboration
4. The **last_recomputed** timestamp enables staleness detection: when the
   taxonomy or embedding model changes, tensors older than the change are
   candidates for re-tensoring (see `algorithm/versioning.md`)
