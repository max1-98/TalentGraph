# Algorithm Layer — Orchestration

How algorithms are scheduled, how they coordinate through shared state, and
how failures are contained.

---

## Execution Modes

Algorithms run in one of three modes depending on the trigger:

| Mode | Trigger | Latency Target | Example |
|------|---------|----------------|---------|
| **On-demand** | API request | Seconds | Gap analysis for a candidate viewing a role |
| **Reactive** | Internal event (CV upload, enrichment signal) | Minutes | CV-to-Profile after a new CV is submitted |
| **Batch** | Scheduled or ad-hoc system event | Hours | Re-tensoring all candidates after a model update |

## Dependency Resolution

The algorithm dependency graph (see `overview.md`) determines valid execution
orders. Before an algorithm runs, its prerequisites are validated:

- **Taxonomy** must be available (it has no prerequisites itself)
- **CV-to-Profile** requires a taxonomy snapshot
- **Profile Enrichment** requires a base tensor from CV-to-Profile
- **Gap Finder** requires a candidate tensor and a role context
- **Career Pathing** requires Gap Finder results
- **Role Recommendation** requires compiled role tensors
- **Co-Founder Matching** requires team tensor and scoring pipeline

Missing prerequisites are handled by mode:
- **On-demand** — reject with a descriptive error (e.g. "candidate has no
  tensor; CV-to-Profile must run first")
- **Reactive** — queue the blocked algorithm and schedule the prerequisite
- **Batch** — resolve the full dependency chain before starting

## Call Chains

Algorithms that compose other algorithms:

- **Career Pathing → Gap Finder** — iterative calls (5–15 per candidate) to
  evaluate gaps against each candidate career move. Candidate-side
  computations (proficiency map, current tensor) are reused across calls.
- **Co-Founder Matching → scoring pipeline** — invokes the standard pipeline
  in a modified mode (gap-emphasis weights, team tensor as query). Not a
  full pipeline rebuild — the existing pipeline is reused with adjusted
  configuration.
- **Profile Enrichment → CV-to-Profile confidence** — calls the confidence
  model as a function (not a full CV-to-Profile re-run) to update confidence
  after merge.
- **Role Recommendation → Query Compiler** — batch-compiles role descriptions
  into QueryTensors. Compiled role tensors are cached and reused across
  candidates.

## Shared Intermediate Caching

Frequently reused computations are cached and shared across algorithm calls:

| Cached Value | Produced By | Consumed By | Cache Lifetime |
|-------------|-------------|-------------|----------------|
| Candidate proficiency map | CV-to-Profile / Enrichment | Gap Finder, Career Pathing | Until tensor_version changes |
| Compiled QueryTensors | Query Compiler | Role Recommendation, Gap Finder | Until taxonomy or model version changes |
| TaxonomyGraph distance matrices | Taxonomy algorithms | Gap Finder (bridge suggestions), Career Pathing | Until taxonomy_snapshot_version changes |
| Skill centrality metrics | TaxonomyGraph | Gap Finder (prioritisation) | Until taxonomy_snapshot_version changes |

Cache lifetime is bounded by the version vector (see `versioning.md`). When
any relevant version component changes, the affected cache entries are
invalidated.

## Batch Scheduling

Recurring and ad-hoc batch processes:

| Process | Frequency | Scope | Priority |
|---------|-----------|-------|----------|
| Taxonomy ingestion | Continuous (streaming) | New skill mentions from all sources | High |
| Taxonomy deduplication | Weekly | Full taxonomy graph | Medium |
| Taxonomy hierarchy inference | Quarterly | Full taxonomy graph | Low |
| LTR training | Nightly or weekly | All accumulated feedback data | High |
| Profile enrichment | Background priority queue | Candidates ordered by staleness and demand | Medium |
| Re-tensoring (model change) | Ad hoc | All candidates, prioritised by staleness | High |
| Re-tensoring (taxonomy change) | Ad hoc | Candidates affected by changed subgraph | Medium |

## Fault Tolerance

Three layers of failure containment:

**Isolation** — one algorithm's failure does not cascade to others. Each
algorithm runs in its own error boundary. A failed enrichment run does not
prevent the scoring pipeline from serving the existing (pre-enrichment) tensor.

**Retry with backoff** — transient failures (network timeouts, rate limits)
are retried with exponential backoff and jitter. Maximum retry count is
configurable per algorithm. After exhausting retries, the failure is logged
and the algorithm moves to the next candidate.

**Circuit breaker** — sustained failures (external API down, corrupted
taxonomy) trigger a circuit breaker that stops further attempts and alerts
operations. The breaker resets after a configurable cool-down period with a
probe request.

**Graceful degradation** — when a component is unavailable, algorithms
degrade rather than fail (see `fault-tolerance.md` for the full per-component
specification and unified error model):

- Scoring pipeline serves stale tensors with a staleness indicator attached
  to results
- Enrichment runs without an unavailable source, logging the skip
- Gap prioritisation runs without market demand signals, falling back to
  graph centrality only
- Career Pathing runs without time estimation, returning paths without
  duration forecasts

## AlgorithmScheduler Trait Contract

The abstract interface for algorithm execution coordination.

- **Inputs:** algorithm identifier, execution mode, target candidate or
  candidate set, priority level
- **Output:** execution result (success with outputs, or failure with reason
  and retry recommendation)
- **Invariants:** dependency ordering is always respected; no algorithm runs
  before its prerequisites; execution is idempotent (re-running produces the
  same result given the same version vector)

### Variants

| Variant | Behaviour | Use Case |
|---------|-----------|----------|
| **Production** | Concurrent execution, dependency resolution, circuit breakers | Live system |
| **Sequential** | Single-threaded, deterministic ordering | Integration tests |
| **Test** | Immediate execution, no scheduling, fixed results | Unit tests |

---

## Cross-References

- Algorithm dependency graph: `overview.md`
- Version vector and staleness: `versioning.md`
- Composition root: `infrastructure/composition.md`
- LTR training: `learning/ltr.md`
- Gap Finder integration: `gap-finder/integration.md`
