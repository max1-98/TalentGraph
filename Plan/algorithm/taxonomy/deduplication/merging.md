# Deduplication Pipeline — Merging

How merge candidates are resolved into golden records. Follows the golden
record pattern: one node becomes canonical, others redirect to it, and the
operation is fully reversible.

Depends on: `deduplication/overview.md` (merge candidates),
`graph-model/contracts.md` (GraphMutator, ChangeObserver).

---

## Merge Execution — Seven Steps

A merge combines N source nodes into one target node. All seven steps succeed
atomically or all roll back — no partial merges.

### Step 1: Target Selection

The node with the highest usage_count becomes the canonical target. Rationale:
it is the most widely referenced in CandidateTensors, so choosing it as
canonical minimises downstream tensor updates. Ties are broken by oldest
created_at (longer-lived node wins).

### Step 2: Embedding Recomputation

The target's embedding is replaced by the usage-weighted mean of all merged
nodes' embeddings:

  new_embedding = SUM_i( usage_count_i * embedding_i ) / SUM_i( usage_count_i )

This preserves the semantic centre of gravity weighted by actual usage, rather
than giving equal influence to a rarely-used alias.

### Step 3: Name and Alias Consolidation

- The target retains its canonical_name (or adopts the most-used name if a
  source node has higher usage).
- All source node names become aliases. The union-find structure is updated
  so every alias in every source node's equivalence class now points to the
  target as its canonical representative.

### Step 4: Edge Transfer

All edges from source nodes migrate to the target:

- **Parent edges:** transferred with DAG validation. If transferring a parent
  edge would create a cycle (because the target already has a conflicting
  relationship), the edge is downgraded to "related" or discarded.
- **Related edges:** transferred and deduplicated. If the target already has
  a related edge to the same neighbour, the higher NPMI weight is kept.
- **Supersedes edges:** transferred directly (no conflict possible).
- **Alias edges:** subsumed by the union-find update in Step 3.

### Step 5: Source Node Status Update

Each source node's status is set to "superseded". A supersedes edge is
created from the target to each source, with migration_complete = false.

### Step 6: Usage Count Aggregation

The target's usage_count becomes the sum of all merged nodes' counts. The
frequency is similarly aggregated.

### Step 7: Change Event Emission

A merge_executed event is emitted via ChangeObserver, containing the target
ID, source IDs, and the full MergeRecord. Downstream listeners react:
tensor recomputation, cache invalidation, index updates.

---

## MergeExecutor Trait Contract

- **Input:** MergeCandidate (from DuplicationDetector), current graph state
- **Output:** MergeRecord (target_id, source_ids, pre-merge snapshot,
  timestamp)
- **Side effects:** mutates graph via GraphMutator; emits change events
- **Invariant:** atomic execution. DAG constraint holds after merge. No ID
  is reassigned or reused.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Full 7-step merge with snapshot and propagation | Normal operation |
| DryRun | Computes merge plan without executing | Preview and validation |
| TestMerger | In-memory, no downstream propagation | Unit testing |

---

## Downstream Propagation

After a merge commits, two downstream systems must update:

1. **CandidateTensor update:** for every tensor that references a source
   node's skill ID, the entry is merged into the target's position using
   max-pool (take the higher value). The source position is zeroed. This
   runs asynchronously; the supersedes edge's migration_complete flag is
   set to true when all affected tensors are updated.

2. **Cache invalidation:** SkillResolver caches and QueryTensor caches that
   reference any source node are invalidated via ChangeObserver.

---

## Rollback

Every merge stores a full pre-merge snapshot in the MergeRecord:

- All source nodes with their attributes (embedding, status, usage_count)
- All edges that were transferred or removed
- The target's pre-merge state

**Trigger:** if downstream scoring metrics degrade after a merge (detected
via A/B comparison of match quality before and after), the merge is reversed.

**Process:** GraphMutator.rollback(merge_record) restores source nodes,
re-transfers edges to their original owners, removes alias redirects, and
emits a merge_rolled_back event. CandidateTensors are reverted by restoring
the source skill positions from the snapshot.

**Retention window:** MergeRecords are retained for 30 days. After this
window, rollback is no longer possible and the snapshot is deleted.

---

## Batch Processing

When multiple merge candidates exist, they are processed sequentially in
descending composite score order (highest confidence first). After each
merge, the remaining candidates are re-evaluated:

- A candidate involving a now-superseded node is removed from the queue.
- A candidate whose composite score changed (because one member was merged
  with a third node) is re-scored.

This sequential approach prevents cascading errors from parallel merges that
might interact unexpectedly.

---

## Cross-References

- Merge candidate detection: `deduplication/overview.md`
- Graph mutation contracts: `graph-model/contracts.md`
- Schema and edge types: `graph-model/schema.md`
- CandidateTensor structure: `core/types.md`
- Change events and observers: `graph-model/contracts.md` (ChangeObserver)
