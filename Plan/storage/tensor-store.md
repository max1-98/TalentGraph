# Tensor Store

Persistence and retrieval layer for pre-compiled candidate tensors.

---

## Purpose

The tensor store holds every candidate's CandidateTensor in a format optimised
for the matching engine's access patterns: sequential scans (Stage 1), random
access by ID (Stage 3), and batch retrieval.

## Access Patterns

| Consumer | Pattern | Requirement |
|----------|---------|-------------|
| Stage 1 (Filter) | Sequential scan of all FilterPacks | Contiguous memory, cache-line aligned |
| Stage 3 (Scoring) | Random access by candidate ID (~2K lookups) | Low-latency, zero-copy |
| Tensor Compiler | Write/update one tensor at a time | Concurrent with reads |
| Startup | Load entire store into memory | Fast initial load |

## Implementations

### Memory-Mapped Binary Store (production)

- Tensors stored as fixed-size structs in a flat binary file
- Memory-mapped (mmap) by the matching engine process
- Zero serialisation cost, zero-copy reads — the OS handles paging
- FilterPacks stored in a separate contiguous array for SIMD-friendly
  sequential scanning
- Approximate sizes: ~4 KB per candidate. 1M candidates = ~4 GB.
  Fits in RAM on a single machine.

### In-Memory Store (testing)

- Tensors held as a collection in heap memory
- Used for unit and integration tests where disk I/O is undesirable
- Same trait interface — tests exercise the same code paths as production

### Sharded Store (future)

- Distributes tensors across multiple nodes for datasets that exceed single-
  machine memory
- The TensorStore trait abstracts away sharding — callers are unaware

## Tensor Compilation Pipeline

Tensors are not written by the API layer directly. A background worker watches
for profile change events (chunk create/update/delete, position change) via a
message stream (likely Redis Streams or Kafka — unconfirmed) and recompiles
the affected candidate's tensor.

1. Load all positions and chunks for the candidate
2. Build each tensor component (skill vector, embeddings, career trajectory,
   meta scores)
3. Write the updated tensor to the store
4. Notify the vector indexes to update their entries

Compilation time: ~2 ms per candidate (CPU-bound after initial data load).
Full reindex of 1M candidates: ~35 minutes on 16 cores.

Staleness tolerance: up to ~1 hour. Tensor recompilation is asynchronous —
a small delay between profile update and tensor refresh is acceptable.

## Concurrency

The store supports concurrent reads and writes. The mmap implementation uses
atomic writes at the struct level — a reader always sees either the old or
the new tensor, never a partial update. No locks on the read path.

## Relationship to PostgreSQL

PostgreSQL (likely — unconfirmed) is the source of truth for all profile data.
The tensor store is a derived, compute-optimised projection. If the tensor
store is lost, it can be fully rebuilt from the database.
