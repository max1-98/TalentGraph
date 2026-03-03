# TalentGraph — Project Conventions

## What This Repo Is

A design documentation repository for TalentGraph, a talent matching engine.
The `Plan/` folder mirrors the intended code/module structure — each folder maps
to a future component, each file describes the design of an algorithm, feature,
or interface.

## File Rules

- **150-line limit** per Markdown file. If a topic outgrows this, split it.
- **Stack-agnostic language** in all design docs. Describe algorithms, data
  flows, and interfaces abstractly — not in any specific programming language.
- When a likely implementation technology is mentioned, keep it **parenthetical
  and hedged**: e.g. "the matching engine (likely Rust)", "the API layer
  (likely Django REST + Postgres — unconfirmed)", "the frontend (likely
  TypeScript — unconfirmed)".
- **No raw code snippets.** Express formulas in plain math notation. Describe
  interfaces with prose: inputs, outputs, invariants.
- `Plan/` folder structure mirrors the future code structure. Each subfolder
  corresponds to a module or crate. Each file describes one design concern.

## Folder Map

| Folder               | Maps to                          |
|----------------------|----------------------------------|
| `Plan/core/`        | Core traits and shared types     |
| `Plan/core/schema/` | Shared schema: cross-boundary type definitions |
| `Plan/scoring/`     | Scoring algorithms               |
| `Plan/pipeline/`    | Search pipeline stages           |
| `Plan/compiler/`    | Query compilation                |
| `Plan/storage/`     | Data layer (tensors + indexes)   |
| `Plan/learning/`    | Learning-to-rank subsystem       |
| `Plan/features/`    | Feature modules                  |
| `Plan/api/`         | Backend API                      |
| `Plan/frontend/`    | Frontend                         |
| `Plan/algorithm/`    | Higher-level algorithm skeletons |
| `Plan/infrastructure/` | Architecture, testing, deploy |
| `Plan/product/`     | Product strategy, engagement, onboarding |

## Repo Structure — Git Submodules

This is a **meta-repo**. Each app lives in its own GitHub repo and is linked here
as a git submodule. The root repo tracks docs (`Plan/`, `TODO/`) and submodule
pointers.

| Path | Repo | Status |
|------|------|--------|
| `prototype/` | `talentgraph-prototype` | visual prototype (Vite + React + TS) |
| `lambda/` (future) | `talentgraph-lambda` | serverless functions |
| `api/` (future) | `talentgraph-api` | backend API |

**Working with submodules:**
- Clone with `git clone --recurse-submodules <root-url>`
- Update all submodules: `git submodule update --remote`
- Commit changes to an app inside its own folder; then update the pointer in root
- Add a new app: `git submodule add <repo-url> <folder>`

## Contribution Workflow

1. Read `Plan/README.md` for the full index and conventions.
2. Use the GitHub issue templates (design-proposal, architecture-decision).
3. PRs must pass the checklist in `.github/PULL_REQUEST_TEMPLATE.md`.
