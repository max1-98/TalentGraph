# Shared Schema — Overview

A single source of truth for every data type that crosses a service boundary.

---

## Why This Layer Exists

TalentGraph has three runtime layers: an IO-bound API, compute-heavy algorithm
services, and a frontend. Each layer may be implemented in a different language.
Without a shared schema, type definitions drift independently and mismatches
surface only at runtime.

The schema layer solves this by formally defining every cross-boundary type in
one place. Layer-specific type definitions are mechanically generated from
these authoritative definitions — never hand-written.

## Three-Tier Type Model

| Tier | Scope | Where Defined | In Shared Schema? |
|------|-------|---------------|-------------------|
| 1 — Engine-internal | Never leaves the compute layer | `core/types.md` | No |
| 2 — API contracts | Crosses the API-to-frontend boundary | `schema/api-contracts.md` | **Yes** |
| 3 — Engine transport | Crosses the API-to-engine boundary only | `schema/engine-transport.md` | **Yes** |

**Tier 1** types (CandidateTensor, FilterPack, QueryTensor, WeightSet) are
optimised for memory layout and compute speed. They remain in `core/types.md`
and are never exposed beyond the engine.

**Tier 2** types are the public API surface — what the frontend sends and
receives. These must be stable, versioned, and well-documented.

**Tier 3** types are internal transport between the API and the engine. They
carry richer metadata than Tier 2 (A/B variant IDs, weight version overrides,
pipeline diagnostics) but are not visible to external consumers.

## Relationship to Other Core Files

- **`core/types.md`** — defines Tier 1 engine-internal types. The schema layer
  does not duplicate or replace these; it defines the *projections* of internal
  state that are safe to expose across boundaries.
- **`core/traits.md`** — defines behavioural contracts (Scorer, Stage, etc.).
  Traits describe what components *do*; the schema describes what data *looks
  like* when it crosses a boundary.

## Schema Compilation Philosophy

The prose definitions in this directory are the authoritative source. From
these, an interface definition language file (likely Protocol Buffers or JSON
Schema — unconfirmed) is generated. From the IDL, per-layer typed definitions
are compiled mechanically. See `serialization.md` for the full compilation
strategy.

## File Index

| File | Contents |
|------|----------|
| [boundary-types.md](boundary-types.md) | Complete inventory of every cross-boundary type, tier assignments |
| [api-contracts-core.md](api-contracts-core.md) | Field-level definitions for Tier 2 core types (profile, search, feedback) |
| [api-contracts-algorithms.md](api-contracts-algorithms.md) | Field-level definitions for Tier 2 algorithm result types |
| [engine-transport.md](engine-transport.md) | Field-level definitions for Tier 3 (API-to-engine) types |
| [serialization.md](serialization.md) | IDL options, per-layer compilation strategy, maintenance tooling |
| [versioning.md](versioning.md) | Compatibility rules, migration process, schema registry |
