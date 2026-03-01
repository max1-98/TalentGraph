# Search Pipeline — Overview

A three-stage funnel that reduces the full candidate universe to a ranked
shortlist. Designed for speed: total latency under 20 ms for one million
candidates (excluding query compilation).

---

## Architecture

```
Job Description
     |
     v
Query Compiler  ← ~800 ms cold / 0 ms cached (only LLM call)
     |
     v  QueryTensor
Stage 1: Hard Pre-Filter       ← ~1 ms   | 1M  → ~50K
     |
     v  ~50K candidate IDs
Stage 2: ANN Retrieval          ← ~5 ms   | 50K → ~2K
     |
     v  ~2K candidate tensors
Stage 3: Exact Scoring          ← ~10 ms  | 2K  → 250 ranked
     |
     v
Output: Ranked list + score breakdowns + matched chunk references
```

## Latency Budget

| Stage | 100K profiles | 1M profiles | 10M profiles | Bottleneck |
|-------|--------------|-------------|--------------|------------|
| Query compile | 800 ms cold / 0 cached | same | same | LLM latency |
| Stage 1: Filter | 0.1 ms | 1 ms | 8 ms | Memory bandwidth |
| Stage 2: ANN | 2 ms | 5 ms | 12 ms | HNSW traversal |
| Stage 3: Score | 3 ms | 10 ms | 15 ms | FP math on ~2K tensors |
| **Total (cached)** | **5 ms** | **16 ms** | **35 ms** | |
| **Total (cold)** | **805 ms** | **816 ms** | **835 ms** | |

## Stage Contract

Every stage implements the Stage trait:
- Receives a candidate set (bitset or ID list) and a shared search context
- Returns a (possibly smaller) candidate set, optional scores, and diagnostics
- Reports estimated cost per candidate (lets the pipeline optimise ordering)

The pipeline itself is a simple sequential loop over its stage list. All
intelligence lives in the stages and scorers, not in the pipeline runner.

## Key Design Decisions

- **Stages are composable.** The pipeline reads its stage list from
  configuration. Adding, removing, or reordering stages is a config change.
- **Each stage narrows progressively.** Stage 1 is cheap and broad. Stage 3
  is expensive and precise. Each stage only processes survivors from the
  previous stage.
- **Early exit on empty.** If any stage produces zero survivors, the pipeline
  returns immediately with an empty result and full diagnostics.
- **Diagnostics at every stage.** Input count, output count, and wall-clock
  duration are recorded for every stage of every search — enabling latency
  monitoring and funnel analysis.

## Query Compilation

The query compiler (a separate module, see `compiler/overview.md`) transforms
free-text job descriptions into structured query tensors. This is the only
LLM call in the hot path and is cached aggressively (identical or near-
identical JDs reuse the same compiled tensor).
