# Core Traits — Interface Contracts

Every component in the matching engine depends on abstract interfaces, never on
concrete implementations. This file defines the seven core trait boundaries.

---

## Scorer

The unit of scoring logic. Each scorer computes one sub-score dimension.

- **Inputs:** a candidate tensor and a search context (query + weights + config)
- **Output:** a single raw float (normalisation is the pipeline's job)
- **Additional methods:**
  - `id` — unique string identifier (used in config and registry)
  - `display_name` — human label for UI and debugging
  - `explain` — optional prose explanation of why this score was given
  - `required_fields` — declares which tensor fields the scorer reads
  - `version` — string that changes when scoring logic changes; used to
    invalidate caches and mark A/B test boundaries
- **Invariants:** must be deterministic, stateless per call, Send + Sync safe

## Stage

The unit of pipeline composition. Each stage is one step in the search funnel.

- **Inputs:** a `StageInput` (candidate set + optional scores) and a shared
  `SearchContext`
- **Output:** a `StageOutput` (filtered/re-scored candidate set + diagnostics)
- **Additional methods:**
  - `name` — human-readable identifier for logging and metrics
  - `estimated_cost_per_candidate` — lets the pipeline sort stages by cost
- **Invariants:** stages are composable in any order; each must be self-contained

## TensorStore

Read access to pre-compiled candidate tensors.

- `get(id)` — retrieve one tensor by candidate ID
- `get_batch(ids)` — retrieve many tensors (batch-optimised)
- `count` — total number of stored tensors
- `get_filter_pack(id)` — retrieve the packed filter struct for one candidate
- `all_filter_packs` — return the full filter array (used by Stage 1)
- **Implementations:** memory-mapped binary file (production), in-memory
  collection (testing), sharded store (future)

## VectorIndex

Approximate nearest-neighbour search over embedding vectors.

- `search(query, k)` — return top-k neighbours with distances
- `search_filtered(query, k, filter_bitmap)` — ANN restricted to a candidate
  subset (critical: the filter is applied *during* graph traversal, not after)
- `index_name` — identifies which embedding this index covers
- **Implementations:** in-process HNSW graph (production), brute-force linear
  scan (testing), quantised search or external vector DB (future)

## QueryCompiler

Transforms a free-text job description into a structured query tensor.

- `compile(job_description)` — returns a query tensor or an error
- `model_version` — identifies the underlying model for cache keying
- **Implementations:** LLM-backed compiler (production, likely Anthropic API),
  rule-based parser using taxonomy lookup (fallback), fixed query (testing)

## TeamSelector

Optimises candidate selection when filling multiple vacancies simultaneously.

- `select(candidates, requirements, vacancies, shortlist_size)` — returns a
  set of seat assignments that maximises team-level coverage
- **Implementations:** greedy submodular with (1 − 1/e) approximation
  (current), top-N split (baseline), integer linear programming (future),
  genetic algorithm (experimental)

## WeightProvider

Supplies the scoring weight vector for a given query.

- `weights_for(query)` — returns the weight set (w1–wN, bias, embedding
  perspective weights, decay parameters)
- `active_version` — identifies the weight set for logging
- **Implementations:** fixed hand-tuned defaults (cold start), LTR-trained
  per-vertical weights (production), A/B test randomisation (experiments)

---

## Dependency Rule

All modules depend on these traits. No module depends on another module's
concrete type. The only place concrete types appear is the composition root
(the pipeline builder), which wires implementations to traits at startup.

## Algorithm Layer Traits

The algorithm layer (see `algorithm/overview.md`) introduces five additional
shared traits — building blocks reused across multiple algorithms:

- **SkillResolver** — maps free-text skill mentions to canonical taxonomy IDs
- **TaxonomyGraph** — living ontology of skill nodes and relationship edges
- **RoleContextAnalyser** — extracts structured requirements from role descriptions
- **ProfileComparator** — computes structured diffs between two profiles or a
  profile and a role
- **MarketSignalProvider** — aggregates skill demand, salary ranges, and trends

These follow the same contract pattern as the core traits above — abstract
inputs/outputs, named invariants, swappable implementations — but live in the
algorithm subfolders because they serve the algorithm layer specifically.
