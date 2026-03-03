# Algorithm Layer — Fault Tolerance

Per-component error specifications and degradation rules. Complements the
generic patterns (retry, circuit breaker, isolation) in `orchestration.md`.

---

## Unified Error Model

Every algorithm result carries a structured envelope: **payload** (the normal
output), **severity** (one of three levels below), and **degradation_notices**
(list of components that were unavailable, with fallback descriptions).

### Severity Levels

| Level | Meaning | Caller Action |
|-------|---------|---------------|
| **Nominal** | All components responded normally | Use result as-is |
| **Degraded** | One or more components fell back to an approximation | Use result but surface the degradation notice to the consumer (UI badge, API field) |
| **Fatal** | A critical component is unavailable and no meaningful approximation exists | Reject the request with a descriptive error; do not return partial results that could mislead |

Severity is the *maximum* across all component statuses: if any component is
Fatal the whole result is Fatal; if the worst is Degraded the result is
Degraded.

---

## Building Block Error Specifications

What happens when each shared building block is unavailable or returning
errors.

| Building Block | Fallback | Severity | Key Impact |
|----------------|----------|----------|------------|
| **SkillResolver** | Hash-based pseudo-IDs (exact-match only, no taxonomy relationships) | Degraded | f1/f3 lose synonym matching; bridge suggestions disabled; pseudo-IDs re-resolved lazily on recovery |
| **TaxonomyGraph** | Cached snapshot (read-only); new mentions queued | Degraded for non-Taxonomy; **Fatal for Taxonomy** (it *is* the graph) | Frozen skill relationships; ingestion/deduplication/hierarchy halt |
| **RoleContextAnalyser** | Rule-based regex + keyword extraction | Degraded | Gap Finder, Co-Founder, Career Pathing receive less nuanced role context |
| **ProfileComparator** | None — no meaningful approximation | **Fatal** for Gap Finder, Co-Founder, Career Pathing | These algorithms reject requests; Role Recommendation and Enrichment unaffected |
| **MarketSignalProvider** | Graph-centrality demand proxy (node degree + PageRank) | Degraded | Gap prioritisation and Career Pathing lose real demand data |

---

## Per-Algorithm Error Specifications

How each algorithm behaves when its upstream dependencies fail.

### Taxonomy

| Upstream Failure | Behaviour |
|-----------------|-----------|
| TaxonomyGraph unavailable | Fatal — cannot operate |
| External ingestion source down | Degraded — skips source, logs, continues with others |

### CV-to-Profile

| Upstream Failure | Behaviour |
|-----------------|-----------|
| SkillResolver unavailable | Degraded — pseudo-IDs for skills, lower f1/f3 quality |
| LLM compiler unavailable | Degraded — falls back to rule-based compiler |
| Document format unsupported | Fatal for that document — returns error with format details |

### Profile Enrichment

| Upstream Failure | Behaviour |
|-----------------|-----------|
| External source unavailable | Degraded — skips source, logs, enriches from remaining |
| CV-to-Profile base tensor missing | Fatal — no tensor to enrich |
| SkillResolver unavailable | Degraded — pseudo-IDs for new skills |

### Gap Finder

| Upstream Failure | Behaviour |
|-----------------|-----------|
| ProfileComparator unavailable | Fatal — cannot produce diffs |
| MarketSignalProvider unavailable | Degraded — prioritisation uses centrality only |
| TaxonomyGraph unavailable | Degraded — bridge suggestions disabled, exact-match gaps only |

### Career Pathing

| Upstream Failure | Behaviour |
|-----------------|-----------|
| Gap Finder fatal | Fatal — cannot compute gap-to-target |
| Gap Finder degraded | Degraded — paths returned with degraded gap data |
| TimeEstimator unavailable | Degraded — paths without duration forecasts |
| MarketSignalProvider unavailable | Degraded — no demand filtering, peer frequency only |

### Role Recommendation

| Upstream Failure | Behaviour |
|-----------------|-----------|
| Compiled role tensors stale | Degraded — serves stale tensors with notice, queues recompilation |
| Scoring pipeline unavailable | Fatal — cannot score candidates against roles |
| ResultDiversifier unavailable | Degraded — returns undiversified ranked list |

### Co-Founder Matching

| Upstream Failure | Behaviour |
|-----------------|-----------|
| ProfileComparator unavailable | Fatal — cannot produce team gap profile |
| RoleContextAnalyser unavailable | Degraded — rule-based extraction for venture needs |
| Scoring pipeline unavailable | Fatal — cannot score complement candidates |

---

## Error Propagation Rules

1. **Degradation notices accumulate.** When algorithm A calls algorithm B and
   B returns Degraded, A appends B's notices to its own and sets its own
   severity to at least Degraded.

2. **Fatal is terminal.** When a dependency returns Fatal, the calling
   algorithm does not attempt to produce partial results — it returns Fatal
   with the upstream error attached.

3. **Composed algorithms inherit the worst severity.** Career Pathing calls
   Gap Finder iteratively (5-15 times). If any call is Fatal, that path is
   excluded. If all are Degraded, the overall result is Degraded.

4. **Batch mode collects per-item results.** One candidate's Fatal does not
   stop the batch. Completion reports include Nominal/Degraded/Fatal counts.

5. **Degradation notices are surfaced to consumers.** API responses include a
   `degradation_notices` array rendered as contextual UI badges.

---

## Cross-References

- Generic fault tolerance patterns: `orchestration.md` § Fault Tolerance
- Version-driven cache invalidation: `versioning.md`
- Component classification: `component-taxonomy.md`
