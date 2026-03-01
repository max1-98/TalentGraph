# f2 — Semantic Relevance Score

Measures how well the candidate's professional experience semantically matches
the job description, evaluated across multiple embedding perspectives.

---

## Formula

f2(c, q) = max(
  beta_1 * cos(c_overall, q_embed),
  beta_2 * cos(c_recent, q_embed),
  beta_3 * cos(c_deepest, q_embed),
  beta_4 * cos(c_skills, q_embed)
)

- cos(a, b) = cosine similarity between two 384-dimensional embeddings
- beta_1..4 = learnable scaling factors per perspective
- The MAX operation selects the single strongest perspective

## Four Embedding Perspectives

| Embedding | Source | What It Captures | When It Wins |
|-----------|--------|------------------|--------------|
| overall | Weighted mean of all chunk embeddings | Holistic professional identity | Broad role matches |
| recent | Weighted mean of last-three-year chunks | Current focus | Recency-sensitive searches |
| deepest | Mean of top-five highest-weight chunks | Peak expertise | Specialist searches |
| skills | Mean of skill-context embeddings | How skills are applied in practice | Technical depth queries |

## Why Max Instead of Weighted Average?

Averaging would punish a candidate whose recent work is highly relevant but
whose early career was in a different field. The max operation says: "if any
facet of your experience is a strong match, capture that."

The beta weights let the model learn which perspectives matter more for
different job types — recent work matters more for fast-moving roles, deepest
expertise matters more for specialist positions.

## Inputs

- **Four candidate embeddings** (384-dim each), pre-computed during tensor
  compilation using weighted means of chunk-level embeddings
- **One query embedding** (384-dim), produced by the query compiler from the
  job description text

## Edge Cases

- **Candidate has no recent chunks:** embed_recent falls back to embed_overall
  (both are computed; the max operation handles graceful degradation)
- **Career changer:** overall embedding may be diluted, but recent and deepest
  embeddings capture the current direction — the max operation ensures the
  strongest signal is used
- **Very short profile:** fewer chunks means embed_deepest and embed_overall
  converge; the confidence field in the tensor can down-weight low-data
  candidates in the final assembly

## Relationship to Other Sub-Scores

f2 captures semantic similarity at the embedding level. It complements f1
(which measures explicit skill coverage) and f3 (which measures skill
co-occurrence patterns). A candidate might score high on f2 without having
explicitly listed every skill — the embedding captures implicit relevance.
