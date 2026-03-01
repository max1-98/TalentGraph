# Stage 2 — ANN Retrieval (Narrow)

Finds the approximate top candidates by semantic similarity using four parallel
nearest-neighbour searches, one per embedding perspective.

---

## Purpose

Reduce ~50K filtered candidates to ~2K by finding those whose embeddings are
closest to the query embedding. This is the semantic narrowing step — it finds
candidates whose experience *means* the right things, even if their explicit
skill lists differ from the query.

## Input / Output

- **In:** ~50K candidate IDs from Stage 1, plus a filter bitmap
- **Out:** ~1 500–2 000 unique candidate IDs (union of four top-500 lists)

## Algorithm

1. Run four ANN searches in parallel, one per embedding index:
   - overall index → top-500
   - recent index → top-500
   - deepest index → top-500
   - skills index → top-500
2. Union all results with deduplication
3. Typical result: ~1 500–2 000 unique candidates

Each search is a **filtered ANN query** — the filter bitmap from Stage 1 is
passed directly into the graph traversal, so the index only visits candidates
that survived pre-filtering. This is critical: filtering after ANN (as most
external vector databases do) would return candidates that fail hard
constraints, wasting retrieval slots.

## Index Structure

Each index is an HNSW (Hierarchical Navigable Small World) graph over
384-dimensional embeddings, held in-process (not in an external service).

Key parameters (tunable via configuration):
- **M** — number of connections per node (default 16)
- **ef_construction** — search breadth during index building (default 200)
- **ef_search** — search breadth at query time (trades speed for recall)

## Why Four Separate Indexes?

A single embedding would force a compromise — recent experience vs. deep
expertise vs. holistic identity. Four indexes let the system find candidates
who are strong from *any* perspective. The union operation ensures no strong
candidate is missed because they happen to be weak on the "wrong" embedding.

## Why In-Process, Not an External Vector DB?

External vector databases (e.g. Qdrant, Pinecone) handle filtered ANN poorly
— they typically filter *after* the ANN step, not during. With in-process
HNSW, the filter bitmap is passed directly to the graph traversal, which is
5–10x faster for filtered queries. It also eliminates network round-trip
latency.

## Latency

- 100K profiles: ~2 ms
- 1M profiles: ~5 ms
- 10M profiles: ~12 ms

Bottleneck: HNSW graph traversal depth and cache locality.

## Relationship to Stage 3

Stage 2 is an approximate retrieval step — it finds candidates that are
*probably* relevant based on embedding proximity. Stage 3 then loads the
full tensors for these ~2K candidates and computes exact multi-signal scores.
The ANN recall imperfection is acceptable because Stage 2's goal is high
recall (not missing good candidates), while Stage 3 handles precision
(correct ranking).
