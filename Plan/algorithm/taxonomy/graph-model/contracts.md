# Graph Data Model — Contracts

Abstract interfaces for accessing and mutating the taxonomy graph. Applies
Interface Segregation (consumers depend only on the slice they need) and
Dependency Inversion (all logic depends on these abstractions, never on
concrete storage).

---

## Sub-Trait Architecture

Rather than one monolithic interface, the graph exposes four narrow sub-traits.

### SkillProvider (read-only skill lookup)

**Purpose:** resolve skill identifiers and discover skills by similarity.

- **get_by_id(id) -> SkillNode | None:** O(1) array lookup.
- **get_by_name(name) -> SkillNode | None:** exact canonical name match,
  following alias redirects transparently.
- **search_by_embedding(vector, k, threshold) -> list of (SkillNode, score):**
  returns the k nearest active nodes above the cosine similarity threshold.
  Delegates to the ANN index internally.
- **list_aliases(id) -> list of SkillNode:** all nodes in the same alias
  equivalence class, including the canonical.
- **get_status(id) -> status enum:** candidate | active | archived | superseded.

**Consumers:** all 7 algorithms, the scoring pipeline, the query compiler.

**Invariants:** never returns superseded nodes as primary results; alias
resolution is transparent. Results sorted by descending similarity.

### HierarchyReader (read-only graph traversal)

**Purpose:** navigate the parent-child and relatedness structure.

- **parents(id) / children(id) -> list of (SkillNode, confidence)**
- **ancestors(id, max_depth) -> list of (SkillNode, depth):** breadth-first
  traversal up the DAG, respecting the depth limit.
- **descendants(id, max_depth) -> list of (SkillNode, depth):** breadth-first
  traversal down the DAG.
- **related(id, min_weight) -> list of (SkillNode, npmi_score):** nodes
  connected by related edges above the minimum NPMI weight.
- **shortest_path(id_a, id_b) -> list of SkillNode | None:** the shortest
  undirected path through any edge type, or None if disconnected.
- **topological_order() -> list of id:** topological sort of parent edges.
  Used for DAG verification and batch processing order.

**Consumers:** Gap Finder, Career Pathing, hierarchy inference, ClusterNode
recomputation.

**Invariants:** traversals never cross alias edges (transparent). Results
reflect the current DAG state.

### GraphMutator (write access)

**Purpose:** modify the graph structure. Restricted to taxonomy pipelines.

- **add_node(name, embedding, status) -> id:** allocates a new monotonic ID,
  creates the node. Status is typically "candidate" at creation.
- **promote(id):** transitions candidate to active. Fails if the node is not
  in candidate status.
- **archive(id):** transitions active to archived. Fails if inbound parent
  or alias edges exist (must be cleaned up first).
- **add_edge(source_id, target_id, edge_type, metadata) -> edge_id:** creates
  an edge. For parent edges, validates the DAG constraint before committing
  (rejects if a cycle would form). For alias edges, merges union-find sets.
- **remove_edge(edge_id):** deletes an edge. For alias edges, splits the
  union-find (only if no other path connects the nodes).
- **merge(source_ids, target_id) -> MergeRecord:** executes a golden-record
  merge (see `deduplication/merging.md`). Returns a MergeRecord containing
  the pre-merge snapshot for rollback.
- **rollback(merge_record):** reverses a merge, restoring the pre-merge state.
- **update_embedding(id, new_vector):** replaces a node's embedding and
  triggers ChangeObserver notification.
- **update_confidence(id, new_confidence):** updates a node's confidence score.

**Consumers:** ingestion, deduplication, hierarchy inference only. Never
exposed to scoring, search, or external algorithms.

**Invariants:** every mutation emits a change event. DAG validated before
parent-edge additions. ID monotonicity guaranteed.

### ChangeObserver (notification)

**Purpose:** react to graph mutations without polling.

- **on_change(event):** called synchronously after each mutation commits.
  Implementations decide whether to act (e.g. invalidate a cache) or ignore.

**Consumers:** tensor recomputation, scoring cache invalidation, SkillResolver
cache.

**Invariant:** observers must not mutate the graph within on_change (prevents
recursive loops). Side effects limited to caches and external notifications.

---

## Composition

The full TaxonomyGraph is the union of all four sub-traits:

  TaxonomyGraph = SkillProvider + HierarchyReader + GraphMutator + ChangeObserver

Consumers declare only the sub-traits they need:

| Consumer | Depends On |
|----------|-----------|
| Scoring pipeline | SkillProvider |
| Gap Finder | SkillProvider + HierarchyReader |
| Career Pathing | SkillProvider + HierarchyReader |
| Query compiler | SkillProvider |
| Ingestion pipeline | SkillProvider + GraphMutator |
| Deduplication pipeline | SkillProvider + HierarchyReader + GraphMutator |
| Hierarchy inference | SkillProvider + HierarchyReader + GraphMutator |
| Tensor recomputation | ChangeObserver |

This means changes to merge logic (GraphMutator internals) can never break
the scoring pipeline (which only sees SkillProvider).

---

## SkillResolver Contract

The SkillResolver is the shared building block used by all 7 algorithms (see
`algorithm/overview.md`). It composes SkillProvider internally.

- **resolve(text, context_embedding) -> Resolution:** takes a free-text skill
  mention and an optional context embedding for disambiguation. Returns:
  - **matched(id, confidence):** mapped to an existing active node.
  - **candidate(text, embedding):** unrecognised; forwarded to ingestion.
- **resolve_batch(mentions) -> list of Resolution:** batch variant.

**Disambiguation:** when multiple nodes match (e.g. "Rust"), the context
embedding breaks the tie — the closest node wins. Without context, the
highest-usage node is returned. Always returns the canonical ID, even if the
input text matches a superseded alias.

---

## Implementations (LSP)

Each sub-trait has production, fallback, and test implementations:

| Sub-Trait | Production | Fallback | Test |
|-----------|-----------|----------|------|
| SkillProvider | ANN-backed index | Linear scan | Fixed dictionary |
| HierarchyReader | Adjacency-list graph | (same) | Hardcoded tree |
| GraphMutator | Persistent store + WAL | In-memory + snapshot | In-memory only |
| ChangeObserver | Multi-listener dispatch | No-op | Event recorder |
