# TalentGraph — Design Plan

A talent matching engine that compiles candidate profiles into dense numerical
tensors, scores them against job queries using a multi-signal weighted function,
and returns ranked results with per-dimension score breakdowns. Designed for
sub-50ms latency at one million profiles.

## How This Folder Works

Each subfolder in `Plan/` maps to a future code module. Each Markdown file
describes the design of one algorithm, interface, or feature. Together they form
a complete blueprint of the system before implementation begins.

The original HTML reference documents live in `_archive/` (git-ignored).

## Conventions

- **150-line limit** per file — split if a topic outgrows this
- **Stack-agnostic language** — describe designs abstractly; mention likely
  technologies only in parentheses and hedged ("likely", "unconfirmed")
- **No raw code** — express formulas in plain math, interfaces as prose
- See `.claude/CLAUDE.md` for the full contributor guide

---

## Folder Index

### `core/` — Core Types and Interfaces

| File | Description |
|------|-------------|
| [traits.md](core/traits.md) | Abstract interface contracts: Scorer, Stage, TensorStore, VectorIndex, QueryCompiler, TeamSelector, WeightProvider |
| [types.md](core/types.md) | Candidate tensor schema, query tensor model, score breakdown structure |

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
| [overview.md](frontend/overview.md) | UI structure, search UX, result display |

### `infrastructure/` — Architecture, Testing, and Deployment

| File | Description |
|------|-------------|
| [architecture.md](infrastructure/architecture.md) | SOLID principles applied, module dependency graph |
| [composition.md](infrastructure/composition.md) | Config-driven pipeline assembly, scorer registry |
| [testing.md](infrastructure/testing.md) | Unit / integration / regression strategy, shadow mode |
| [deployment.md](infrastructure/deployment.md) | Canary rollout, instant rollback, cost projections |
