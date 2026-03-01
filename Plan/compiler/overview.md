# Query Compiler — Overview

Transforms a free-text job description into a structured QueryTensor that the
scoring pipeline can evaluate against candidate tensors.

---

## Purpose

The query compiler is the bridge between human-written job descriptions and the
mathematical scoring engine. It extracts structured signals — required skills,
target seniority, industry, location constraints — and produces a fixed-format
tensor suitable for pairwise scoring operations.

## Compilation Steps

1. **Parse the job description** — extract skills, seniority level, industry,
   location, salary, notice period, employment type, and other constraints
2. **Map skills to the global taxonomy** — resolve free-text skill mentions
   to canonical skill IDs, handling aliases and synonyms
3. **Assign importance weights** — classify each skill as required (1.0),
   preferred (0.5), or nice-to-have (0.2) based on JD language
4. **Generate query embedding** — produce a 384-dimensional embedding from the
   full JD text, using the same embedding model as candidate tensors
5. **Build skill bloom filter** — for Stage 1's fast probabilistic check
6. **Assemble the QueryTensor** — populate all fields (see `core/types.md`)

## Implementations

### LLM-Backed Compiler (production)

Uses a large language model (likely Anthropic API — unconfirmed) to parse the
JD with high accuracy. Handles ambiguous language, implicit requirements, and
context-dependent skill extraction.

- Latency: ~800 ms per cold compilation
- This is the only LLM call in the search hot path

### Rule-Based Compiler (fallback)

Regex + taxonomy lookup. Lower accuracy but deterministic and fast. Used as a
fallback when the LLM is unavailable or when latency requirements are strict.

### Fixed Query Compiler (testing)

Returns hardcoded QueryTensors for unit and integration testing.

## Caching

Compiled QueryTensors are cached aggressively (likely in Redis — unconfirmed).
Cache key: a hash of the normalised JD text. Identical or near-identical JDs
reuse the same compiled tensor, reducing the effective compilation latency to
near zero for repeat searches.

Cache invalidation: when the skill taxonomy is updated or the compiler model
version changes, cached tensors are flushed.

## Model Version Tracking

The QueryCompiler trait exposes a `model_version` method. This version string
is included in the cache key and in search diagnostics, ensuring that:
- Cached tensors from an old model are not reused after an upgrade
- A/B tests can run different compiler versions and track results separately

## Edge Cases

- **Vague JD with no extractable skills:** falls back to embedding-only
  matching (f2 dominates, f1 and f3 contribute little)
- **JD in a non-English language:** depends on the underlying model's
  multilingual capabilities; the rule-based fallback is English-only
- **Extremely long JD:** truncated to the model's context window; the
  embedding is computed from the truncated text
