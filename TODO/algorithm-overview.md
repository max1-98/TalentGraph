# Algorithm Layer — Design Gaps

Issues in `algorithm/overview.md` and across the algorithm layer that need
addressing before or during implementation.

---

## Structural

- [x] **Consistency and versioning model.** Multiple algorithms modify
  CandidateTensor (CV-to-Profile creates, Enrichment updates, Taxonomy changes
  may invalidate skill mappings). Document: update ordering, tensor versioning,
  whether taxonomy changes trigger re-tensoring, concurrent safety.
  *Done: `algorithm/versioning.md`.*

- [x] **Algorithm orchestration.** Some algorithms compose others (Career
  Pathing calls Gap Finder, Co-Founder calls the scoring pipeline). Document:
  call sequencing, whether algorithms share cached intermediate results,
  scheduling for batch vs on-demand execution. *(Co-founder composition
  details are backlogged; document the other call chains first.)*
  *Done: `algorithm/orchestration.md`.*

- [x] **Fault tolerance and fallbacks.** Each algorithm doc mentions edge cases
  but there is no unified error model. Document: what happens when
  SkillResolver cannot resolve, when TaxonomyGraph is unavailable, when an
  upstream algorithm fails. Define graceful degradation strategy.
  *Done: `algorithm/fault-tolerance.md`.*

- [x] **Shared vs algorithm-specific components.** The overview lists 5 shared
  building blocks, but each algorithm also defines its own swappable parts
  (DocumentParser, SourceDiscoverer, GapPrioritiser, etc.). Clarify the
  distinction — what makes a component "shared" vs "algorithm-specific"?
  *Done: `algorithm/component-taxonomy.md`.*

## Accuracy

- [x] **Dependency graph verification.** The ASCII graph in `overview.md` shows
  Co-Founder Matching depending on Gap Finder, but the co-founder doc does not
  reference Gap Finder directly. Profile Enrichment is shown connecting to Gap
  Finder and Role Recommendation, but the enrichment doc only describes it as
  input to CV-to-Profile. Verify and correct the graph.
  *Done: corrected graph and added dependency table in `algorithm/overview.md`.*

- [x] **Taxonomy vs TaxonomyGraph terminology.** The dependency graph lists both
  "TaxonomyGraph" (the trait) and "Skill Taxonomy" (the algorithm). Clarify
  the distinction for readers who might confuse them.
  *Done: `algorithm/taxonomy/graph-model/contracts.md` defines TaxonomyGraph
  as the composed trait (SkillProvider + HierarchyReader + GraphMutator +
  ChangeObserver); `taxonomy/overview.md` is the algorithm that operates it.*
