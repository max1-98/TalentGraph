# Shared Schema — Serialization and Compilation

How authoritative prose definitions become typed structures in each runtime
layer.

---

## The Compilation Problem

Three layers, potentially three languages, one source of truth. The prose
definitions in `api-contracts-core.md`, `api-contracts-algorithms.md`, and
`engine-transport.md` are authoritative. From these, an interface definition
language (IDL) file is written. From the IDL, per-layer typed definitions are
generated mechanically.

## IDL Options

| IDL | Strengths | Weaknesses |
|-----|-----------|------------|
| Protocol Buffers | Strong typing, compact binary wire format, broad language support, backward-compatibility tooling | Requires a build step; less natural for REST/JSON APIs |
| JSON Schema | Native to HTTP/JSON APIs, validator libraries in every language, human-readable | No binary format; weaker type system; no built-in RPC |
| GraphQL SDL | Self-documenting, flexible querying, strong frontend tooling | Heavier runtime; less suited for internal engine transport |

The choice depends on implementation-time constraints. Protocol Buffers are
the likely default for engine transport (likely gRPC — unconfirmed) while
JSON Schema may suit the API-to-frontend boundary. A hybrid approach — Proto
for Tier 3, JSON Schema for Tier 2 — is viable since the authoritative
definitions are prose and both IDLs are derived artifacts.

## Per-Layer Compilation Strategy

### Engine (likely Rust — unconfirmed)

- **Tier 2 and 3 types:** IDL compiles to native structs. These are the wire
  types for communication with the API layer.
- **Tier 1 types:** remain hand-written in `core/types.md`. Memory layout,
  SIMD alignment, and packed representations require manual control that no
  IDL generator provides.
- **Mapping layer:** the engine implements explicit conversion functions
  between Tier 1 and Tier 2/3 types. For example, CandidateTensor fields
  are projected into a ProfilePayload; raw ScoreBreakdown floats are mapped
  to ScoreBreakdownDisplay integers with display labels.

### API Layer (likely Django REST + Postgres — unconfirmed)

- **Tier 2 types:** IDL compiles to serializer/validator schemas. Every API
  endpoint validates request and response payloads against the compiled schema.
- **Tier 3 types:** IDL compiles to client stubs for engine communication.
- **Database models are separate.** The shared schema defines wire types, not
  storage models. Database tables may store data in a different shape —
  normalised, denormalised, or partitioned. The API layer maps between
  database models and schema types. This separation prevents schema changes
  from forcing database migrations and vice versa.

### Frontend (likely TypeScript — unconfirmed)

- **Tier 2 types:** IDL compiles to typed interfaces. The frontend never
  manually defines API shapes — all request and response types are generated.
- **Tier 3 types:** not generated for the frontend (it never interacts with
  the engine directly).
- **Display logic is separate.** The schema defines data shapes, not how they
  are rendered. Component props may wrap or extend schema types with
  UI-specific fields (loading state, selection state, etc.).

## Build Pipeline

1. Prose definitions are written and reviewed (this directory)
2. IDL files are authored to match the prose (mechanically verifiable)
3. Per-language generation runs, producing typed files for each layer
4. CI validates that generated files match the checked-in copies (staleness
   check — any drift fails the build)

## Maintenance Scripts and Tooling

Implementation-time scripts that keep the schema healthy. These live in a
future `scripts/schema/` (or similar) directory. Their design is described
here; implementation is deferred until the first layer is built.

### Code Generation Script

Reads the IDL files and emits typed source files for each target layer. Invoked
whenever the IDL changes. Produces native structs for the engine, serializer
schemas for the API, and typed interfaces for the frontend.

### Staleness Checker

A CI step that re-runs code generation and compares the output against the
files checked into the repository. If they differ, the build fails. This
prevents the generated files from drifting out of sync with the IDL.

### Compatibility Validator

Compares a new IDL version against the previous version. Flags breaking changes
(field removals, type changes, renamed fields) and enforces the backward-
compatibility rules described in `versioning.md`. Runs as part of the PR
review pipeline.

### Migration Scaffold Generator

Given a breaking schema change, generates skeleton migration code for each
affected layer: database migration stubs for the API, type adapter stubs for
the engine, and updated interface stubs for the frontend. Reduces the manual
effort of coordinating breaking changes across layers.

---

## What the Schema Does NOT Cover

- **Tier 1 types** — engine-internal, hand-written for performance
- **Database models** — storage concerns are separate from wire types
- **Frontend display logic** — rendering, state management, component props
- **RPC protocol details** — whether the engine uses gRPC, HTTP, or message
  queues is an infrastructure concern, not a schema concern
