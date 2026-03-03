# Algorithm Layer — Overview

Higher-level algorithms that sit on top of the core matching engine. Each
algorithm composes shared building blocks to solve a distinct talent problem —
from parsing CVs to suggesting career paths.

---

## Design Philosophy

- **Compose, don't duplicate.** Every algorithm is assembled from shared traits.
  Logic that appears in two algorithms is extracted into a building block.
- **Trait-bounded.** Each swappable part is defined by an abstract interface
  (trait) with clear inputs, outputs, and invariants. Implementations can be
  swapped via configuration.
- **Extend the pipeline.** Algorithms reuse existing pipeline stages and scoring
  functions wherever possible rather than rebuilding equivalent logic.

## Shared Building Blocks

Five traits that prevent the same logic being reinvented across algorithms.

| Trait | Purpose | Used By |
|-------|---------|---------|
| **SkillResolver** | Map free-text to canonical taxonomy IDs; handle aliases and synonyms | All algorithms |
| **TaxonomyGraph** | Living ontology: skill nodes + relationship edges (parent, alias, related, supersedes) | All (via SkillResolver); directly by Taxonomy, Gap Finder, Career Pathing |
| **RoleContextAnalyser** | Role/company description to structured requirements (skills, seniority, team needs) | Gap Finder, Co-Founder, Role Recommendation, Career Pathing |
| **ProfileComparator** | Two profiles (or profile + role) to structured diff (missing, surplus, deltas) | Gap Finder, Co-Founder, Career Pathing |
| **MarketSignalProvider** | Aggregate demand data: skill frequency, salary ranges, growth trends | Career Pathing, Gap Finder, Taxonomy, Profile Enrichment |

## Relationship to Existing Modules

| Building Block | Existing Module | Relationship |
|----------------|-----------------|--------------|
| SkillResolver | `compiler/overview.md` step 2 | Extracted and generalised from the compiler's skill-mapping stage |
| ProfileComparator | `scoring/` f1–f5 | Generalises scoring sub-functions into structured diffs |
| MarketSignalProvider | `learning/feedback.md` | Sibling projection — same outcome data, different aggregation |
| RoleContextAnalyser | `compiler/overview.md` | Extends the compiler for richer offline (non-latency-bound) analysis |
| TaxonomyGraph | `core/types.md` skill IDs | Elevates the flat skill-ID space into a graph with relationships |

## Algorithm Dependency Graph

```
                     TaxonomyGraph
                          |
                    Skill Taxonomy
                   /    |    |    \
                  /     |    |     \
    CV-to-Profile  Profile   |   Role
          |      Enrichment  |   Recommendation
          |                  |
          |             Gap Finder
          |            /         \
    Co-Founder    Career
     Matching     Pathing
```

Taxonomy is the foundation — every other algorithm depends on it.

### Dependency Table

| Algorithm | Direct Dependencies |
|-----------|-------------------|
| Skill Taxonomy | TaxonomyGraph (trait) |
| CV-to-Profile | Taxonomy |
| Profile Enrichment | Taxonomy, CV-to-Profile (for base tensor) |
| Gap Finder | Taxonomy, ProfileComparator, RoleContextAnalyser, MarketSignalProvider |
| Career Pathing | Taxonomy, Gap Finder, MarketSignalProvider, ProfileComparator |
| Role Recommendation | Taxonomy, Query Compiler (for role tensors), Scoring Pipeline |
| Co-Founder Matching | Taxonomy, RoleContextAnalyser, ProfileComparator, Scoring Pipeline |

## Algorithm Index

| Algorithm | Folder | One-Line Summary |
|-----------|--------|------------------|
| Skill Taxonomy | `taxonomy/` | Self-filling, self-correcting skill ontology |
| CV-to-Profile | `cv-to-profile/` | Unstructured CV to structured CandidateTensor |
| Gap Finder | `gap-finder/` | Profile vs role to actionable skill gaps |
| Co-Founder Matching | `cofounder-matching/` | Complementary team member discovery |
| Career Pathing | `career-pathing/` | Data-driven next-move suggestions |
| Role Recommendation | `role-recommendation/` | Inverse search: candidate to matching roles |
| Profile Enrichment | `profile-enrichment/` | External signal augmentation of profiles |

## Cross-Cutting Concerns

Four topics span all algorithms and are documented separately:

| File | Covers |
|------|--------|
| [versioning.md](versioning.md) | Tensor version vector, staleness detection, update ordering, concurrent safety, audit trail |
| [orchestration.md](orchestration.md) | Execution modes, dependency resolution, call chains, caching, batch scheduling, fault tolerance |
| [component-taxonomy.md](component-taxonomy.md) | Shared vs algorithm-specific components, classification criteria, composition pattern, promotion path |
| [fault-tolerance.md](fault-tolerance.md) | Unified error model, per-building-block and per-algorithm error specs, degradation propagation |
