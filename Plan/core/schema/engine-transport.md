# Engine Transport — Tier 3 Types

Field-level definitions for types that cross the API-to-engine boundary but
are never exposed to the frontend. These carry richer metadata than Tier 2
types and support operational concerns (A/B testing, diagnostics, weight
management).

> See `boundary-types.md` for the full inventory and tier classification.

---

## EngineSearchRequest

Sent from the API to the engine when a recruiter submits a search. Wraps the
public SearchRequest with internal metadata.

- **job_description** — the raw JD text (same as SearchRequest)
- **filters** — structured constraints (same as SearchRequest)
- **result_limit** — maximum candidates to return
- **vacancies** — number of positions (triggers team selection when > 1)
- **search_id** — internal identifier for tracing and feedback linkage
- **ab_variant** — which experimental variant this search is assigned to
  (controls scorer selection, weight set, pipeline configuration)
- **weight_version_override** — optional; forces a specific weight set version
  instead of the currently active one (used for shadow-mode comparisons)
- **compilation_hints** — optional metadata passed to the query compiler
  (e.g. preferred taxonomy version, embedding model override for experiments)
- **requester_org_id** — the organisation initiating the search (for rate
  limiting and consent filtering)

## EngineSearchResponse

Returned from the engine to the API after pipeline execution.

- **search_id** — matches the request
- **scored_candidates** — ordered list, each containing:
  - **candidate_id** — unique identifier
  - **raw_composite** — the unscaled composite score (float)
  - **raw_breakdown** — per-scorer floats keyed by scorer ID (f1, f2, ...)
  - **matched_chunk_refs** — references to the experience chunks that
    contributed to the score (for generating experience highlights)
- **compilation_metadata** — details of query compilation: model version,
  cache hit/miss, compilation latency, extracted skill count
- **weight_version_used** — which weight set was applied
- **scorer_versions** — map of scorer IDs to their active version strings
- **diagnostics** — a PipelineDiagnostics record (see below)

The API maps raw_breakdown to ScoreBreakdownDisplay (Tier 2) by applying
display labels, scaling to 0–100, and ordering dimensions.

---

## PipelineDiagnostics

Per-stage execution metrics for a single search. Attached to every
EngineSearchResponse and also available to admin endpoints.

- **stages** — ordered list, each containing:
  - **stage_name** — human-readable identifier (e.g. "filter", "narrow",
    "precision")
  - **input_count** — candidates entering this stage
  - **output_count** — candidates surviving this stage
  - **wall_clock_ms** — execution duration in milliseconds
- **total_wall_clock_ms** — sum across all stages (excludes compilation)
- **timestamp** — when the search executed

See `pipeline/overview.md` for the stage architecture these diagnostics
describe.

## ProfileChangeEvent

Emitted to a message stream when a candidate profile is created or updated.
The engine consumes these events to trigger tensor recompilation.

- **candidate_id** — the affected candidate
- **change_type** — created, updated, or deleted
- **changed_fields** — list of field names that changed (allows the engine to
  decide whether a full recompilation is needed or a partial update suffices)
- **timestamp** — when the change occurred
- **source** — what triggered the change (manual edit, CV upload, enrichment)

## WeightSnapshot

A point-in-time capture of the active weight set, exposed to admin endpoints
for inspection and as input to retraining triggers.

- **version** — the weight set version identifier
- **weights** — map of scorer IDs to their current weights (w1 through wN)
- **bias** — the bias term
- **embedding_scaling** — perspective scaling factors (beta1 through beta4)
- **decay_parameters** — recency lambda, seniority sigma, and other curve
  parameters
- **scope** — which query cluster (industry + seniority) this set applies to,
  or "global" for the default
- **trained_at** — when this weight set was last trained
- **training_sample_size** — number of preference pairs used in training

See `scoring/overview.md` and `learning/ltr.md` for the weight system design.

## AlgorithmInvocation

A generic request envelope for invoking higher-level algorithms
(`algorithm/overview.md`) through the engine.

- **algorithm_id** — which algorithm to run (e.g. "gap-finder",
  "career-pathing", "cofounder-matching", "role-recommendation")
- **inputs** — algorithm-specific input payload (varies by algorithm; the
  envelope schema is shared, the payload shape is algorithm-dependent)
- **requester_id** — the user or service initiating the invocation
- **trace_id** — for distributed tracing across the API and engine
- **timeout_ms** — maximum allowed execution time
