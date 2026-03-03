# TalentGraph — Design Plan

A talent matching engine: candidate profiles compiled into dense tensors, scored
against job queries via multi-signal weighted functions, ranked with per-dimension
breakdowns. Sub-50ms latency at one million profiles.

Each subfolder maps to a future code module; each file describes one design
concern. Reference docs: `_archive/` (git-ignored). Outstanding work: `TODO/`.
Conventions: 150-line limit, stack-agnostic language, no raw code (see
`.claude/CLAUDE.md`).

---

## Folder Index

### `core/` — Core Types, Interfaces, and Shared Schema

| File | Description |
|------|-------------|
| [traits.md](core/traits.md) | Abstract interface contracts: Scorer, Stage, TensorStore, VectorIndex, QueryCompiler, TeamSelector, WeightProvider |
| [types.md](core/types.md) | Candidate tensor schema, query tensor model, score breakdown structure (engine-internal, Tier 1) |
| [schema/overview.md](core/schema/overview.md) | Shared schema layer: why it exists, three-tier model, file index |
| [schema/boundary-types.md](core/schema/boundary-types.md) | Complete inventory of every cross-boundary type with tier assignments |
| [schema/api-contracts-core.md](core/schema/api-contracts-core.md) | Tier 2 field-level definitions: profile, search, feedback types |
| [schema/api-contracts-algorithms.md](core/schema/api-contracts-algorithms.md) | Tier 2 field-level definitions: algorithm result types |
| [schema/engine-transport.md](core/schema/engine-transport.md) | Tier 3 field-level definitions: API-to-engine internal protocol |
| [schema/serialization.md](core/schema/serialization.md) | IDL options, per-layer compilation strategy, maintenance tooling |
| [schema/versioning.md](core/schema/versioning.md) | Compatibility rules, migration process, schema registry |

### `scoring/` — Scoring Algorithms

| File | Description |
|------|-------------|
| [overview.md](scoring/overview.md) | Composite scoring formula, weight system, score assembly |
| [skill-coverage.md](scoring/skill-coverage.md) | f1: weighted skill hit rate with importance tiers |
| [semantic-relevance.md](scoring/semantic-relevance.md) | f2: multi-perspective embedding cosine similarity |
| [skill-synergy.md](scoring/skill-synergy.md) | f3: skill co-occurrence via PMI + vector similarity |
| [career-trajectory.md](scoring/career-trajectory.md) | f4: seniority Gaussian, industry fit, stability |
| [practical-fit.md](scoring/practical-fit.md) | f5: location, salary, notice period, employment type |

### `pipeline/` — Search Pipeline Stages

| File | Description |
|------|-------------|
| [overview.md](pipeline/overview.md) | Three-stage funnel architecture and latency budget |
| [filter.md](pipeline/filter.md) | Stage 1: bitwise pre-filter on packed integers (~1 ms) |
| [narrow.md](pipeline/narrow.md) | Stage 2: ANN retrieval across four embeddings (~5 ms) |
| [precision.md](pipeline/precision.md) | Stage 3: exact multi-signal scoring (~10 ms) |

### `compiler/` — Query Compilation

| File | Description |
|------|-------------|
| [overview.md](compiler/overview.md) | JD-to-query-tensor compilation, caching, fallback strategy |

### `storage/` — Data Layer

| File | Description |
|------|-------------|
| [tensor-store.md](storage/tensor-store.md) | Candidate tensor persistence, memory-mapped access |
| [vector-index.md](storage/vector-index.md) | ANN index structure, filtered search, configuration |

### `learning/` — Learning Subsystem

| File | Description |
|------|-------------|
| [ltr.md](learning/ltr.md) | Learning-to-rank framework (pairwise LambdaRank) |
| [feedback.md](learning/feedback.md) | Outcome signals, preference pairs, feedback loop |
| [cold-start.md](learning/cold-start.md) | Day-one priors through full LTR ramp strategy |

### `features/` — Feature Modules

| File | Description |
|------|-------------|
| [team-selection.md](features/team-selection.md) | Multi-vacancy optimisation via submodular selection |
| [user-intelligence.md](features/user-intelligence.md) | Aggregated score breakdowns as user-facing insights |

### `api/` — Backend API (likely Django REST + Postgres — unconfirmed)

| File | Description |
|------|-------------|
| [overview.md](api/overview.md) | Endpoints, data flow, authentication sketch |

### `frontend/` — Frontend (likely TypeScript — unconfirmed)

| File | Description |
|------|-------------|
| [overview.md](frontend/overview.md) | UI structure, data flow, design principles |
| [layout.md](frontend/layout.md) | Nav structure, design aesthetic, mobile-first layout, micro-interactions |
| [candidate-experience.md](frontend/candidate-experience.md) | Feed, match inbox, gap finder, career path, profile, insights |
| [hiring-experience.md](frontend/hiring-experience.md) | Post role, applicant view, multi-role, team-fill |
| [profile-builder.md](frontend/profile-builder.md) | CV import, progressive completion, AI suggestions, privacy controls |

### `product/` — Product Strategy Layer

| File | Description |
|------|-------------|
| [overview.md](product/overview.md) | Product vision, core mechanics, competitive positioning, success metrics |
| [go-to-market.md](product/go-to-market.md) | Cold-start strategy, atomic network model, expansion playbook |
| [engagement.md](product/engagement.md) | Habit loops, feed design, streak mechanics, badge system, notification philosophy |
| [onboarding.md](product/onboarding.md) | Talent and employer onboarding flows, activation events, progressive disclosure |

### `algorithm/` — Higher-Level Algorithm Skeletons

| File | Description |
|------|-------------|
| [overview.md](algorithm/overview.md) | Shared building blocks, dependency graph, algorithm index |
| [taxonomy/overview.md](algorithm/taxonomy/overview.md) | Self-filling, self-correcting skill ontology |
| [taxonomy/graph-model/schema.md](algorithm/taxonomy/graph-model/schema.md) | Node/edge types, invariants, growth bounds |
| [taxonomy/graph-model/contracts.md](algorithm/taxonomy/graph-model/contracts.md) | ISP sub-traits, SkillResolver contract |
| [taxonomy/ingestion/overview.md](algorithm/taxonomy/ingestion/overview.md) | Skill discovery: extraction, embedding, classification |
| [taxonomy/ingestion/admission.md](algorithm/taxonomy/ingestion/admission.md) | Poisson frequency gating, initial placement |
| [taxonomy/deduplication/overview.md](algorithm/taxonomy/deduplication/overview.md) | Four-signal duplicate detection |
| [taxonomy/deduplication/merging.md](algorithm/taxonomy/deduplication/merging.md) | Golden record merge, rollback |
| [taxonomy/hierarchy/overview.md](algorithm/taxonomy/hierarchy/overview.md) | Co-occurrence analysis: PMI, WeedsPrec, Hearst |
| [taxonomy/hierarchy/inference.md](algorithm/taxonomy/hierarchy/inference.md) | Edge typing, DAG enforcement, confidence decay |
| [cv-to-profile/overview.md](algorithm/cv-to-profile/overview.md) | Unstructured CV to structured CandidateTensor |
| [cv-to-profile/parsing.md](algorithm/cv-to-profile/parsing.md) | Document parsing: formats, section labelling, OCR, DocumentParser trait |
| [cv-to-profile/extraction.md](algorithm/cv-to-profile/extraction.md) | Skill and entity extraction pipeline |
| [cv-to-profile/confidence.md](algorithm/cv-to-profile/confidence.md) | Bayesian confidence model: field-level and aggregate scoring |
| [cv-to-profile/cost-model.md](algorithm/cv-to-profile/cost-model.md) | LLM call cost estimation and optimisation |
| [cv-to-profile/evaluation.md](algorithm/cv-to-profile/evaluation.md) | Accuracy metrics, ground-truth dataset, regression testing |
| [gap-finder/overview.md](algorithm/gap-finder/overview.md) | Profile vs role to actionable skill gaps |
| [gap-finder/contracts.md](algorithm/gap-finder/contracts.md) | ISP trait interfaces: ProficiencyGapScorer, SemanticGapDetector, GapPrioritiser, BridgeSuggester |
| [gap-finder/proficiency-scoring.md](algorithm/gap-finder/proficiency-scoring.md) | Decay-aware continuous deltas, context thresholds, gap type classification |
| [gap-finder/semantic-detection.md](algorithm/gap-finder/semantic-detection.md) | Soft matching, latent skill inference, synergy gaps, cluster analysis |
| [gap-finder/prioritisation.md](algorithm/gap-finder/prioritisation.md) | JSD, EMD, market signals, graph centrality, Pareto frontier |
| [gap-finder/bridge-suggestions.md](algorithm/gap-finder/bridge-suggestions.md) | Graph traversal, transferability scoring, collaborative filtering, learning paths |
| [gap-finder/integration.md](algorithm/gap-finder/integration.md) | Building block wiring, Career Pathing callsite, User Intelligence feed, schema extensions |
| [cofounder-matching/overview.md](algorithm/cofounder-matching/overview.md) | Complementary team member discovery |
| [career-pathing/overview.md](algorithm/career-pathing/overview.md) | Data-driven next-move suggestions |
| [role-recommendation/overview.md](algorithm/role-recommendation/overview.md) | Inverse search: candidate to matching roles |
| [profile-enrichment/overview.md](algorithm/profile-enrichment/overview.md) | External signal augmentation of profiles |
| [profile-enrichment/merge-strategy.md](algorithm/profile-enrichment/merge-strategy.md) | Conflict resolution, field-level merge rules, confidence updating |
| [profile-enrichment/privacy.md](algorithm/profile-enrichment/privacy.md) | Opt-out, enrichment log schema, notification, retention, regulatory |
| [taxonomy/bootstrap.md](algorithm/taxonomy/bootstrap.md) | Initial taxonomy seeding from external sources |
| [versioning.md](algorithm/versioning.md) | Tensor version vector, staleness, update ordering, concurrent safety |
| [orchestration.md](algorithm/orchestration.md) | Algorithm scheduling, call chains, caching, fault tolerance |

### `infrastructure/` — Architecture, Testing, and Deployment

| File | Description |
|------|-------------|
| [architecture.md](infrastructure/architecture.md) | SOLID principles applied, module dependency graph |
| [composition.md](infrastructure/composition.md) | Config-driven pipeline assembly, scorer registry |
| [testing.md](infrastructure/testing.md) | Unit / integration / regression strategy, shadow mode |
| [deployment.md](infrastructure/deployment.md) | Canary rollout, instant rollback, cost projections |
