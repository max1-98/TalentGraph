# Graph Data Model — Schema

The foundational data structure for the taxonomy. Every other taxonomy process
reads or writes this model; it depends on nothing outside `core/`.

---

## Node Types

### SkillNode

The primary entity. Each node represents a single, canonical skill concept.

**Attributes:**

- **id** (monotonic integer): unique, immutable, never reused. This ID indexes
  into the CandidateTensor skill vector — reassignment would silently corrupt
  every stored tensor. IDs are allocated from a monotonic counter; merges
  create alias edges, never reassign IDs.
- **canonical_name** (string): the preferred human-readable label. Updated only
  via deduplication merge or manual override.
- **embedding** (dense vector, 384 floats — likely all-MiniLM-L6-v2, or 768
  for E5-Base-v2 — unconfirmed): context-conditioned semantic representation.
  Recomputed when the skill's usage context shifts significantly.
- **status** (enum): candidate | active | archived | superseded.
  - *candidate:* awaiting promotion (see `ingestion/admission.md`)
  - *active:* fully referenced, eligible for matching
  - *archived:* zero usage for a configurable window, soft-deleted, recoverable
  - *superseded:* merged into another node, alias edge points to target
- **usage_count** (integer): how many CandidateTensors reference this node.
  Decremented on tensor deletion; triggers archival review when it hits zero.
- **frequency** (float): normalised appearances per time window across the
  ingested corpus. Used by admission gating and market signal integration.
- **confidence** (float, 0–1): how certain we are this is a valid, distinct
  skill. Set at promotion, refined by deduplication and hierarchy inference.
- **node_type** (enum): skill | certification | course. Defaults to "skill".
  Certifications and courses participate fully in graph traversal — a
  certification node connects to the skills it covers via parent edges.
  Introduced to support bridge suggestion mechanics in
  `gap-finder/bridge-suggestions.md`.
- **created_at** / **last_referenced_at** (timestamps): lifecycle tracking.

### ClusterNode (Virtual)

A higher-level grouping for navigation and display (e.g. "Frontend
Development", "Data Engineering"). Not referenced by CandidateTensors — these
exist purely for human browsing and hierarchy organisation.

- **embedding:** centroid (mean) of children's embeddings, recomputed when
  children change (triggered via ChangeObserver).
- **child_count:** maintained incrementally; used for display and balancing.

---

## Edge Types

### parent (directed, child to parent)

Represents "is-a" / specialisation. "React Hooks" is a child of "React".
Polyhierarchy is allowed — a node may have multiple parents ("TypeScript"
under both "JavaScript" and "Typed Languages").

Attributes: confidence (0–1), inference_method (hearst | cooccurrence |
distributional | manual), created_at.

### alias (undirected equivalence class)

Represents "same-as". "JS" and "JavaScript" are aliases. One node per class
is canonical; others have status = superseded and redirect to it.

Storage: union-find structure for O(alpha(n)) transitive lookup. The
canonical node is always the class representative.

### related (undirected, weighted)

Represents "frequently co-occurs but is distinct". "React" and "Redux" are
related but not equivalent. Weight is the NPMI score (see
`hierarchy/overview.md`). Only stored when NPMI exceeds a minimum threshold.

### supersedes (directed, new to old)

Represents "replaces". When a technology is renamed or split (e.g. a
framework rebrands), the new node supersedes the old. Carries a
migration_complete flag tracking whether downstream CandidateTensors have
been updated.

---

## Invariants

The following properties hold after every mutation:

1. **DAG constraint:** parent edges form a directed acyclic graph, verified by
   topological sort. Any mutation that would create a cycle is rejected.
2. **Alias transitivity:** the union-find guarantees exactly one canonical
   representative per equivalence class. No transitive alias chains.
3. **ID stability:** IDs are never reassigned or reused. Merges create alias
   edges; the source ID persists as a superseded redirect.
4. **Embedding consistency:** ClusterNode embeddings are recomputed whenever
   their children set changes, via ChangeObserver notification.
5. **Referential integrity:** every edge endpoint references an existing node.
   Archival checks for inbound edges before proceeding.

---

## Growth Bounds and Performance

- **Expected scale:** 5K–50K SkillNodes, ~3–5 edges per node on average.
- **Lookup by ID:** O(1) via array indexing (IDs are dense monotonic integers).
- **Embedding search:** delegated to the system's ANN index (see
  `storage/vector-index.md`).
- **Memory:** the full graph fits in memory at 50K nodes. No pagination needed.
- **CandidateTensor coordination:** when active node count approaches the
  tensor's skill-dimension limit (~5K active), the storage layer must be
  notified to plan dimension expansion (see `storage/tensor-store.md`).

---

## Versioning and Change Events

Every mutation produces a typed change event:

- node_added, node_archived, node_status_changed
- edge_added, edge_removed, edge_confidence_updated
- merge_executed, merge_rolled_back

Events are consumed by ChangeObserver implementations for cache invalidation,
tensor recomputation, and audit logging. A full pre-mutation snapshot is
stored for rollback capability (see `deduplication/merging.md`).
