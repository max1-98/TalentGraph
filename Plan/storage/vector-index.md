# Vector Index

Approximate nearest-neighbour (ANN) index for embedding-based retrieval in
Stage 2 of the search pipeline.

---

## Purpose

Provides fast semantic search: given a query embedding, find the K candidates
whose embeddings are closest, restricted to the subset that survived Stage 1.

## Index Structure

Four independent HNSW (Hierarchical Navigable Small World) graphs, one per
embedding perspective:

| Index | Embedding | Dimension |
|-------|-----------|-----------|
| overall | embed_overall | 384 |
| recent | embed_recent | 384 |
| deepest | embed_deepest | 384 |
| skills | embed_skills | 384 |

Each graph connects candidates by embedding proximity using a multi-layer
navigable small-world structure. Search starts at the top layer and descends,
narrowing to a local neighbourhood at each level.

## Key Parameters

- **M** — connections per node (default 16). Higher M improves recall but
  increases memory and build time.
- **ef_construction** — search breadth during index building (default 200).
  Controls index quality.
- **ef_search** — search breadth at query time. Higher values improve recall
  at the cost of latency. Tunable per query.

These are configurable in the engine configuration file.

## Filtered Search

The VectorIndex trait exposes a `search_filtered` method that accepts a
bitmap of valid candidate IDs. During graph traversal, the index skips nodes
not present in the bitmap.

This is the critical advantage over external vector databases: filtering
happens *during* traversal, not after. External services that filter after
ANN return candidates that fail hard constraints, wasting retrieval slots
and degrading recall within the valid set.

## Implementations

### HNSW Index (production)

In-process graph held in memory alongside the tensor store. Likely using a
purpose-built or existing HNSW library (unconfirmed). Supports filtered
search natively.

### Brute-Force Index (testing)

Linear scan computing exact cosine similarity. Slow but produces perfect
results — used as a reference implementation for verifying HNSW recall.

### Quantised Search (future)

Quantised embeddings (e.g. product quantisation) for reduced memory at scale.
Same VectorIndex trait interface.

## Index Updates

When a candidate's tensor is recompiled, the corresponding entries in all
four indexes are updated incrementally. Full index rebuilds are only required
when the embedding model version changes.

## Memory

Each HNSW graph for 1M candidates with M = 16 uses approximately 200–400 MB.
Four indexes total: ~1–1.5 GB. Combined with the tensor store (~4 GB), the
matching engine requires ~5.5 GB of RAM for 1M candidates.

## Scaling

At 10M candidates, the indexes require ~10–15 GB total. Options:
- Vertical scaling (larger instance)
- Sharding the indexes across multiple engine replicas, each holding a subset
- Quantised representations to reduce per-vector memory
