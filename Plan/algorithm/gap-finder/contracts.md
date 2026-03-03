# Gap Finder — Contracts

Abstract interfaces for the Gap Finder's four internal stages. Follows
Interface Segregation — each consumer depends only on the slice it needs.

Depends on: `algorithm/overview.md` (building blocks), `taxonomy/graph-model/
contracts.md` (SkillProvider, HierarchyReader).

---

## Composition Order

The Gap Finder pipeline chains six components in sequence:

RoleContextAnalyser -> ProfileComparator -> ProficiencyGapScorer ->
SemanticGapDetector -> GapPrioritiser -> BridgeSuggester -> assemble result

The first two are shared building blocks (see `algorithm/overview.md`). The
four traits below are Gap Finder-specific.

---

## ProficiencyGapScorer

**Purpose:** extend the raw ProfileComparator diff into continuous, typed gaps
with confidence scores.

- **score(candidate_tensor, required_skills, role_context) -> list of
  ProficiencyGap:** each gap carries effective_proficiency, required_threshold,
  delta, gap_type, and confidence. See `proficiency-scoring.md` for the
  delta formula, decay model, and threshold logic.
- **Invariant:** delta is always non-negative. Gaps with delta = 0 are
  excluded from the output. gap_type is one of: missing, weak_slight,
  weak_significant, decayed, experience.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Decay-aware proficiency, context-dependent thresholds | Normal operation |
| SimpleDelta | Raw (required minus current), no decay, fixed thresholds | Reduced data environments |
| Fixed | Returns a predetermined gap list | Unit testing |

---

## SemanticGapDetector

**Purpose:** find partial coverage and inferred skills beyond exact matching,
using embedding similarity and co-occurrence intelligence.

- **detect(candidate_tensor, required_skills, taxonomy_graph) ->
  SemanticGapAnalysis:** returns adjusted deltas (reduced where soft matches
  exist), inferred skills with supporting evidence, and synergy gaps. Consumes
  SkillProvider (embedding search) and HierarchyReader (related edges) only.
  See `semantic-detection.md` for formulas.
- **Invariant:** inferred skills always include evidence (which candidate
  skills support the inference and their co-occurrence strength). Adjusted
  deltas never exceed the original ProficiencyGapScorer deltas.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Embedding soft match + NPMI inference + synergy detection | Normal operation |
| EmbeddingOnly | Soft match only, no co-occurrence or synergy analysis | When co-occurrence matrix unavailable |
| Disabled | Pass-through, returns gaps unchanged | Baseline comparisons |

---

## GapPrioritiser

**Purpose:** rank gaps by a composite multi-criteria score that balances
importance, information gain, market demand, strategic value, and uncertainty.

- **prioritise(gaps, role_context, market_signals, taxonomy_graph) ->
  PrioritisedGapList:** each gap receives a priority score, Pareto frontier
  flag, and confidence interval. Consumes SkillProvider (for centrality
  lookup) and HierarchyReader (for graph distances used in EMD). See
  `prioritisation.md` for the composite formula.
- **Invariant:** the output is sorted by descending priority score. Pareto
  frontier flags are consistent with the non-dominance definition across
  (Importance, MarketSignal, GraphCentrality).

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Full composite: JSD + EMD + market + centrality + uncertainty | Normal operation |
| SimpleWeighted | importance_weight * demand_percentile only | Minimal data, early launch |
| Fixed | Returns a predetermined priority ordering | Unit testing |

---

## BridgeSuggester

**Purpose:** propose actionable bridge paths for closing each gap, using graph
traversal, embedding similarity, and collaborative filtering.

- **suggest(gap, candidate_skills, taxonomy_graph, acquisition_history) ->
  list of BridgeSuggestion:** returns up to 3 bridge paths per gap, each with
  source skill, path through graph, transferability score, and optional
  certification recommendation. Consumes SkillProvider + HierarchyReader. See
  `bridge-suggestions.md` for traversal and scoring details.
- **suggest_batch(gaps, candidate_skills, taxonomy_graph,
  acquisition_history) -> map of gap to list of BridgeSuggestion:** batch
  variant that shares graph traversal state across gaps for efficiency.
- **Invariant:** all suggested bridge paths have length <= 3 hops.
  Transferability scores are in the range 0–1. Results are sorted by
  descending transferability within each gap.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Graph + embedding + collaborative filtering | Normal operation |
| GraphOnly | Graph traversal + embedding, no acquisition history | Cold-start, no history data |
| Fixed | Returns predetermined bridge suggestions | Unit testing |

---

## Output Assembly

The four traits feed into the final GapAnalysisResult. The assembler is not a
trait — it is deterministic glue that merges ProficiencyGap entries, semantic
adjustments, priority scores, and bridge suggestions into the response schema
defined in `core/schema/api-contracts-algorithms.md`. Schema extensions are
documented in `integration.md`.

---

## Cross-References

- Proficiency delta and decay model: `proficiency-scoring.md`
- Embedding and co-occurrence detection: `semantic-detection.md`
- Priority formula and Pareto analysis: `prioritisation.md`
- Bridge traversal and scoring: `bridge-suggestions.md`
- System integration and schema: `integration.md`
- Shared building blocks: `algorithm/overview.md`
- TaxonomyGraph sub-traits consumed: `taxonomy/graph-model/contracts.md`
