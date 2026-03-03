# Algorithm Layer — Component Taxonomy

Classifies every swappable component in the algorithm layer as either shared
(cross-algorithm building block) or algorithm-specific. This drives composition
strategy and prevents both duplication and premature generalisation.

---

## Classification Criteria

A component is **shared** when all three conditions hold:

1. **Multiple consumers.** Three or more algorithms depend on it.
2. **Algorithm-agnostic interface.** Its contract makes no assumption about
   which algorithm is calling it — inputs and outputs are domain-general.
3. **Duplication risk.** Without extraction, two or more algorithms would
   independently implement equivalent logic.

A component is **algorithm-specific** when:

1. **Single consumer.** Only one algorithm uses it (or two, but with
   fundamentally different contracts).
2. **Domain-specific interface.** Its inputs, outputs, or invariants are
   meaningful only within the context of one algorithm.

Grey area: a component used by exactly two algorithms with similar contracts.
Leave it algorithm-specific until a third consumer emerges (see Promotion Path
below).

---

## Shared Building Blocks

Five traits that satisfy all three shared-component conditions.

| Trait | Consumers | Contract Location |
|-------|-----------|-------------------|
| **SkillResolver** | All 7 algorithms | `compiler/overview.md` step 2 (origin); `algorithm/overview.md` (generalised) |
| **TaxonomyGraph** | All (via SkillResolver); directly by Taxonomy, Gap Finder, Career Pathing | `taxonomy/graph-model/contracts.md` |
| **RoleContextAnalyser** | Gap Finder, Co-Founder, Role Recommendation, Career Pathing | `compiler/overview.md` (origin); `algorithm/overview.md` (generalised) |
| **ProfileComparator** | Gap Finder, Co-Founder, Career Pathing | `scoring/overview.md` (origin); `algorithm/overview.md` (generalised) |
| **MarketSignalProvider** | Career Pathing, Gap Finder, Taxonomy, Profile Enrichment | `learning/feedback.md` (origin); `algorithm/overview.md` (generalised) |

Each shared trait is defined once, with multiple implementations swappable via
configuration (production, fallback, test).

---

## Algorithm-Specific Components

Grouped by owning algorithm. Each component has a single consumer and a
domain-specific contract.

### Taxonomy

| Component | Purpose |
|-----------|---------|
| IngestionSource | Stream raw skill mentions from external feeds |
| DeduplicationStrategy | Detect and merge duplicate skill nodes |
| HierarchyInferencer | Infer parent-child relationships from usage patterns |

### CV-to-Profile

| Component | Purpose |
|-----------|---------|
| DocumentParser | Convert PDF/DOCX/HTML/plain-text to labelled sections |
| FieldExtractor | Map labelled sections to CandidateTensor fields |
| ConfidenceModel | Compute per-field and aggregate confidence scores |

### Profile Enrichment

| Component | Purpose |
|-----------|---------|
| SourceDiscoverer | Find external data sources for a given candidate |
| MergeStrategy | Resolve conflicts between CV data and external signals |
| PrivacyGatekeeper | Enforce opt-out and data-retention policies |

### Gap Finder

| Component | Purpose |
|-----------|---------|
| GapPrioritiser | Rank gaps by importance, market demand, and closability |
| BridgeSuggester | Propose adjacent skills that close or reduce gaps |

### Career Pathing

| Component | Purpose |
|-----------|---------|
| TrajectoryPeerFinder | ANN search over career vectors for similar histories |
| DestinationAnalyser | Aggregate peer next-moves into a destination distribution |
| PathRanker | Score and rank paths by feasibility, demand, and peer evidence |
| TimeEstimator | Estimate time to close gaps for each path |

### Role Recommendation

| Component | Purpose |
|-----------|---------|
| RoleSourcing | Provide the corpus of active roles to search against |
| ScoringMode | Configure the scoring pipeline for candidate-fixed mode |
| CandidatePreFilter | Reversed Stage 1: filter roles by candidate attributes |
| ResultDiversifier | Ensure variety across the top-N results |

### Co-Founder Matching

| Component | Purpose |
|-----------|---------|
| TeamTensorAggregator | Combine individual tensors into a team tensor |
| ComplementarityScorer | Score candidates by how well they fill team gaps |
| SelectionOptimiser | Submodular selection for multi-seat filling |

---

## Composition Pattern

**Shared building blocks** are injected via the composition root
(`infrastructure/composition.md`). A single instance of each shared trait is
constructed at startup and passed to every algorithm that needs it. This
guarantees all algorithms see the same taxonomy snapshot and skill resolution
logic.

**Algorithm-specific components** are assembled by per-algorithm builder
functions. Each algorithm's builder accepts the shared traits plus its own
specific components, returning a fully configured algorithm instance. Builders
are the only code that knows which specific implementations to wire together.

---

## Promotion Path

When an algorithm-specific component is needed by a second algorithm:

1. **Assess contract compatibility.** If both consumers need the same inputs,
   outputs, and invariants, the component is a promotion candidate.
2. **Extract the trait.** Move the interface definition to the shared building
   blocks. The original implementation remains; the second consumer provides
   its own or reuses the existing one.
3. **Update the composition root.** The promoted trait is now injected
   centrally rather than constructed per-algorithm.
4. **Update this file.** Move the component from the algorithm-specific table
   to the shared table and record the new consumers.

If the contracts differ, keep both algorithm-specific and document the
divergence — premature unification is worse than controlled duplication.

---

## Cross-References

- Shared trait definitions: `algorithm/overview.md`
- Composition root: `infrastructure/composition.md`
