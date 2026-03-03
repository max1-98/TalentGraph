# Role Recommendation — Compilation Strategy

How role descriptions are pre-compiled into QueryTensors and kept fresh,
so that query-time role recommendation incurs no compilation latency.

---

## Pre-Compilation

Every active role in the system has a pre-compiled QueryTensor stored in a
dedicated RoleTensorStore. Compilation uses the same Query Compiler pipeline
(`compiler/overview.md`) that handles recruiter searches — the same steps
(parse, map skills, assign weights, generate embedding, build bloom filter,
assemble tensor) applied in batch rather than on-demand.

### RoleTensorStore

A secondary data store (likely the same storage backend as candidate tensors —
unconfirmed) holding compiled role tensors keyed by role ID. Each entry
includes the compiled QueryTensor plus a version triple (see Staleness
Management below).

### Secondary HNSW Index

A separate HNSW index is built over the 384-dimensional role embeddings
(extracted from compiled QueryTensors). This enables ANN pre-filtering:
before exact scoring, retrieve the top-500 roles nearest to the candidate's
embedding, reducing the exact-scoring workload.

---

## Corpus Sizing

Expected role corpus: 10K–100K active roles, depending on deployment scale.

### Cold Compilation

Processing the full corpus from scratch:

- Per-role compilation: ~800 ms (dominated by the LLM-backed compiler)
- Parallelism: 8 concurrent compilations (rate-limited to avoid overloading
  the LLM provider)
- Full corpus at 50K roles: ~1.4 hours wall-clock time
- Cold compilation is a one-time or rare event (model upgrade, fresh
  deployment)

### Incremental Compilation

Steady-state: roles are compiled reactively on create or update events.

- New role created → compile and insert into RoleTensorStore + HNSW index
- Role updated → recompile and replace entry; HNSW index updated in-place
  (mark old vector deleted, insert new)
- Role deactivated → remove from HNSW index; optionally retain tensor for
  analytics

Incremental compilation is event-driven (likely via a message queue —
unconfirmed), not polling-based. Latency from role change to compiled
tensor availability: typically under 5 seconds.

---

## Staleness Management

A compiled role tensor becomes stale when any of its inputs change. The
system tracks a version triple per tensor:

| Component | What It Tracks | Change Trigger |
|-----------|---------------|----------------|
| `role_content_hash` | Hash of the normalised role description text | Role edited by employer |
| `taxonomy_snapshot_version` | Version of the skill taxonomy at compilation time | Taxonomy ingestion, deduplication, or hierarchy change |
| `compiler_model_version` | Version string of the Query Compiler model | Compiler model upgrade |

### Staleness Detection

On each query, the system compares the stored version triple against the
current system versions. Any mismatch marks the tensor as stale.

### Stale Tensor Handling

Stale tensors are served with a degradation notice (see
`algorithm/fault-tolerance.md`) rather than blocking the query. The stale
tensor is simultaneously queued for recompilation.

### Recompilation Priority

Stale tensors are recompiled in priority order:

1. **Query frequency** — roles that appear frequently in recommendation
   results are recompiled first (they affect the most candidates)
2. **Staleness severity** — a compiler model version change is higher
   priority than a minor taxonomy update
3. **Age** — older stale entries are prioritised over newer ones

### TTL Safety Net

Regardless of detected staleness, every compiled tensor is recompiled after
a maximum TTL of 7 days. This catches edge cases where a change was not
propagated through the version tracking system.

---

## Query-Time Cost

With pre-compilation, role recommendation at query time involves no
compilation. The full latency breakdown:

| Stage | Latency | Description |
|-------|---------|-------------|
| ANN pre-filter | ~5 ms | HNSW search over role embeddings for top-500 nearest roles |
| Practical filter | ~1 ms | Location, salary, seniority band filter on 500 candidates |
| Exact scoring | ~10 ms | f1–f5 scoring of the candidate against ~500 surviving roles |
| Diversification | ~1 ms | DPP or MMR pass over top results (see `diversification.md`) |
| **Total** | **~17 ms** | Well within real-time latency targets |

This makes role recommendation viable as a real-time feature (page load,
notification trigger) rather than a batch-only operation.

---

## Edge Cases

- **Burst of role updates** (e.g. employer edits 500 roles at once):
  compilation queue absorbs the burst; stale tensors served in the interim
- **Compiler unavailable:** stale tensors continue serving with degradation
  notice; recompilation queue pauses and drains when compiler recovers
- **HNSW index rebuild:** incremental updates handle steady state; full
  rebuild is scheduled during low-traffic windows if the index degrades
  beyond a fragmentation threshold

---

## Cross-References

- Query Compiler pipeline: `compiler/overview.md`
- Role Recommendation overview: `role-recommendation/overview.md`
- Diversification strategy: `role-recommendation/diversification.md`
- Staleness and version vectors: `algorithm/versioning.md`
- Fault tolerance: `algorithm/fault-tolerance.md`
