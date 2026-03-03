# Algorithm Layer — Tensor Versioning

How tensor consistency is maintained when multiple algorithms and system
changes can modify a candidate's tensor.

---

## Version Vector

Each CandidateTensor carries a four-component version vector:

| Component | Type | What It Tracks |
|-----------|------|----------------|
| **tensor_version** | Monotonic counter | Incremented on every write to this tensor |
| **embedding_model_version** | String | Which embedding model produced the current embeddings |
| **taxonomy_snapshot_version** | Counter | Which taxonomy version the skill IDs resolve against |
| **enrichment_generation** | Counter | How many enrichment passes have been applied |

The existing `last_recomputed` timestamp (see `core/types.md`) is retained as
a convenience field for human readability and coarse monitoring, but is not
sufficient for consistency decisions — the version vector is authoritative.

## Staleness Detection

A tensor is **stale** when any component of its version vector lags behind the
current system version for that component. Staleness is a per-tensor property,
not a system-wide state.

| Component Lag | Meaning | Severity |
|---------------|---------|----------|
| embedding_model_version behind | Embeddings computed with an older model | High — affects ANN retrieval and f2 scoring |
| taxonomy_snapshot_version behind | Skill IDs may reference merged or deprecated nodes | Medium — affects f1 and f3 accuracy |
| enrichment_generation behind | Profile may be missing available external signals | Low — affects completeness, not correctness |

A **staleness histogram** (count of tensors by lag magnitude per component)
is the primary monitoring metric. Alerts fire when the fraction of stale
tensors exceeds a configurable threshold per component.

## Re-Tensoring Triggers

Four events cause a tensor to be rebuilt or updated:

| Trigger | Scope | Rebuild Type | Scheduling |
|---------|-------|-------------|------------|
| CV upload or update | Single candidate | Full rebuild via CV-to-Profile | On-demand, synchronous |
| Enrichment run | Single candidate | Partial update via MergeStrategy | Background priority queue |
| Embedding model update | All candidates | Re-embed only (profile text unchanged) | Batch, prioritised by staleness |
| Taxonomy structural change | Affected candidates | Re-resolve skill IDs, recompute f1/f3 | Batch, scoped to affected subgraph |

## Update Ordering

When multiple triggers apply to the same candidate, the canonical sequence is:

1. **CV-to-Profile** creates or rebuilds the base tensor
2. **Profile Enrichment** augments with external signals
3. **Re-embed** on embedding model change (independent of enrichment)
4. **Re-resolve** on taxonomy change (independent of embedding)

Steps 1 and 2 are ordered (enrichment depends on a base tensor). Steps 3 and
4 are independent of each other but depend on the output of steps 1–2. A
dependency-aware scheduler ensures this ordering (see `orchestration.md`).

## Concurrent Safety

**Invariant:** no two algorithms mutate the same candidate's tensor
concurrently. Two mechanisms enforce this:

- **Writer serialisation** — compare-and-swap on the tensor_version counter.
  A writer reads the current version, computes the update, and writes only
  if the version has not changed. On conflict, the writer retries with the
  updated tensor. Alternatively, a per-candidate mutex (likely a lightweight
  lock in the storage layer — unconfirmed) serialises writes.
- **Reader isolation** — readers see a consistent snapshot. Implementation
  options: copy-on-write (readers hold a reference to the pre-update tensor
  until their operation completes) or a generation counter (readers check
  generation before and after reading; if it changed, re-read).

Readers are never blocked by writers. Writers may briefly contend with other
writers on the same candidate, but cross-candidate parallelism is unlimited.

## Audit Trail

Previous tensor versions are retained for auditability. Each version record
includes:

- **reason_for_change** — which trigger caused this update (CV upload,
  enrichment, model change, taxonomy change)
- **previous_version_vector** — the full version vector before the update
- **changed_fields** — list of tensor fields that were modified
- **actor** — which algorithm or system process performed the update

Retention is configurable: production systems may keep the last N versions
or retain versions for a fixed duration (see
`profile-enrichment/privacy.md` for enrichment-specific retention).

## TensorVersionManager Trait Contract

The abstract interface for version management operations.

- **Inputs:** candidate ID, proposed tensor update, expected version vector
- **Output:** success (new version vector) or conflict (current version vector
  for retry)
- **Invariants:** version counter never decreases; every write is logged in
  the audit trail; stale reads are detectable

### Variants

| Variant | Behaviour | Use Case |
|---------|-----------|----------|
| **Production** | CAS-based writes, audit logging, staleness tracking | Live system |
| **InMemory** | Simple version counter, no persistence | Integration tests |
| **Test** | Fixed version vector, no conflict detection | Unit tests |

---

## Cross-References

- Core tensor schema: `core/types.md`
- CV-to-Profile: `cv-to-profile/overview.md`
- Enrichment overview: `profile-enrichment/overview.md`
- Enrichment privacy: `profile-enrichment/privacy.md`
- Gap Finder integration: `gap-finder/integration.md`
- Composition root: `infrastructure/composition.md`
- Orchestration: `orchestration.md`
