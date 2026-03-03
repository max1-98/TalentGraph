# Cross-Reference Updates

The algorithm layer introduced 5 shared building blocks (SkillResolver,
TaxonomyGraph, RoleContextAnalyser, ProfileComparator, MarketSignalProvider)
and 7 algorithms. Existing docs need updating to acknowledge these.

---

## Core Module

- [x] **`core/traits.md` — add algorithm building blocks.** The 5 shared traits
  are not defined in the core traits file where all other traits live. Either
  add them to `core/traits.md` or create `algorithm/traits.md` and
  cross-reference from core.
  *Done: added "Algorithm Layer Traits" section listing all 5 building blocks.*

- [x] **`core/types.md` — add data origin section.** CandidateTensor is defined
  but its lifecycle is not explained: CV-to-Profile creates it, Profile
  Enrichment augments it, confidence reflects enrichment state. Add a short
  "Data Origin and Evolution" section.
  *Done: added "Data Origin and Evolution" section with lifecycle and
  staleness detection.*

## Compiler

- [x] **`compiler/overview.md` — acknowledge SkillResolver.** Step 2 (map
  skills to taxonomy) is an instance of the SkillResolver trait. Note that
  this logic is shared with CV-to-Profile and Profile Enrichment. Add
  cross-reference to `algorithm/overview.md`.
  *Done: added "Relationship to Algorithm Layer" section.*

## Scoring

- [x] **`scoring/overview.md` — note ProfileComparator generalisation.** The
  sub-scores (f1–f5) can be generalised into structured profile diffs for use
  in gap analysis and co-founder matching. Add a brief note and cross-reference.
  *Done: added "Relationship to Algorithm Layer" section referencing Gap Finder
  and ProfileComparator.*

## Learning

- [x] **`learning/feedback.md` — acknowledge MarketSignalProvider.** Aggregated
  outcome data feeds both the LTR weight system and the MarketSignalProvider
  building block (skill demand, salary ranges, growth trends). Document this
  dual use of feedback data.
  *Done: added "Relationship to Market Signals" section.*

## Infrastructure

- [x] **`infrastructure/architecture.md` — add algorithm layer to module graph.**
  The dependency graph currently shows only the core engine modules. Expand it
  to show how algorithm modules (taxonomy, cv-to-profile, etc.) depend on
  core, compiler, scoring, and storage.
  *Done: added paragraph after Module Dependency Graph describing the algorithm
  layer's position and its 5 building-block traits.*

## Features

- [x] **`features/user-intelligence.md` — connect to Gap Finder / Career
  Pathing.** Gap recommendations shown to users should be informed by both
  historical matching data and MarketSignalProvider signals. Note the
  connection.
  *Done: added explicit cross-references to `gap-finder/integration.md` and
  `career-pathing/overview.md`.*
