# Deduplication Pipeline — Overview (Multi-Signal Detection)

How redundant nodes are detected. Duplicates are inevitable: "JavaScript" vs
"JS" vs "ECMAScript", concept drift, parallel ingestion from different sources.
No single similarity measure is sufficient — we combine four independent
signals.

Depends on: `graph-model/contracts.md` (SkillProvider, HierarchyReader).

---

## Why Four Signals?

Each signal captures a different dimension of similarity and has distinct
failure modes. Combining them produces robust duplicate detection that no
single signal achieves alone.

### Signal 1: Embedding Cosine Similarity (weight 0.40)

Captures semantic meaning regardless of surface form. "Machine Learning" and
"ML" have similar embeddings because they appear in similar contexts.

- **Strength:** handles synonyms, abbreviations, and cross-lingual variants
  without any manual mapping.
- **Weakness:** may conflate related-but-distinct skills. "Python" and
  "Cython" have close embeddings but are different skills.

Computed via SkillProvider.search_by_embedding. Uses the same embedding model
as CandidateTensor vectors.

### Signal 2: String Similarity (weight 0.20)

A battery of text comparisons, combined as the maximum of all measures:

- **Jaro-Winkler:** effective for short strings and typo detection.
  Prioritises matching prefixes, which suits skill names.
- **Normalised Levenshtein:** edit distance divided by max length. Catches
  minor spelling variations ("PostgreSQL" vs "Postgresql").
- **Acronym expansion:** maps common patterns — "JS" expands to "JavaScript",
  "ML" to "Machine Learning". Uses a small abbreviation lookup table (the
  only manually curated component in the pipeline).
- **Token-set overlap:** for multi-word skills, compares the set of words
  regardless of order ("Machine Learning" vs "Learning, Machine").

- **Strength:** catches surface variations that embeddings miss (e.g. when
  two spellings produce distant embeddings due to tokenisation differences).
- **Weakness:** fails for different-root synonyms ("car" vs "automobile").

### Signal 3: Context Overlap (weight 0.25)

Jaccard similarity of the document sets where each skill appears:

  context_overlap(A, B) = |docs(A) intersection docs(B)| / |docs(A) union docs(B)|

If two skills always appear in the same documents, they are likely the same
thing or at least used interchangeably.

- **Strength:** captures usage equivalence independent of the skill name or
  embedding. Two completely different strings that always co-occur in the
  same documents are flagged.
- **Weakness:** co-occurring but distinct skills (HTML and CSS) also show
  high overlap. This signal alone has limited precision.

### Signal 4: Usage Pattern Similarity (weight 0.15)

Compares the proficiency-level distributions across all candidates using
Jensen-Shannon Divergence:

  usage_similarity(A, B) = 1 - JSD(proficiency_dist(A), proficiency_dist(B))

True duplicates will have nearly identical distributions — if "JS" and
"JavaScript" are the same skill, candidates' self-reported proficiency levels
should follow the same pattern.

- **Strength:** captures behavioural equivalence that no other signal
  measures. Independent of text, embeddings, and document co-occurrence.
- **Weakness:** requires sufficient data per skill (at least 30 candidates).
  Unreliable for rare or newly promoted skills.

---

## Composite Scoring

The four signals are combined into a weighted composite:

  composite = 0.40 * embedding_sim + 0.20 * string_sim
            + 0.25 * context_overlap + 0.15 * usage_similarity

Weights are initial values calibrated on a labelled duplicate/non-duplicate
dataset. They may be refined via logistic regression on production data.

### Three-Tier Decision

| Composite Score | Decision | Action |
|----------------|----------|--------|
| > 0.95 | **Auto-merge** | Execute merge immediately |
| 0.85 – 0.95 | **Human review** | Queue for lightweight accept/reject |
| <= 0.85 | **Ignore** | No action; pair is distinct |

Thresholds are tunable per deployment.

---

## Efficiency: ANN Pre-Filter

Naively comparing all pairs is O(n^2), infeasible at 10K+ nodes. Instead:

1. For each active node, query the ANN index for the top-k=20 nearest
   neighbours by embedding similarity.
2. Only compute the full 4-signal composite for these candidate pairs.

This reduces the comparison space to O(n * k), with an ANN query cost of
O(k * log(n)) per node. The embedding pre-filter is the fastest signal and
has the highest weight, so it serves as an effective first-pass filter.

---

## DuplicationDetector Trait Contract

- **Input:** set of active SkillNodes (via SkillProvider), corpus statistics
  (document sets, proficiency distributions)
- **Output:** list of MergeCandidate (node_a, node_b, composite_score,
  per-signal scores, decision tier)
- **Invariant:** every candidate pair has all four signals computed (or
  explicitly marked as insufficient-data for usage similarity).

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | ANN pre-filter + 4-signal composite | Normal operation |
| EmbeddingOnly | Cosine similarity only, no string/context/usage | Fast sweep, cold start |
| Exhaustive | All pairs, no ANN pre-filter | Small graphs, validation |

---

## Cross-References

- Merge execution and rollback: `deduplication/merging.md`
- Embedding model and similarity: `graph-model/schema.md`
- Graph contracts: `graph-model/contracts.md`
- JSD and distributional comparison: used here for usage patterns
