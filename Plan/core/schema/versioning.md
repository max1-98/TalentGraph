# Shared Schema — Versioning and Evolution

Rules for evolving shared types without breaking consumers.

---

## Compatibility Rules

### Backward-Compatible Changes (non-breaking)

These changes can be deployed without coordinating across all layers:

- **Add an optional field** — existing consumers ignore it; new consumers use it
- **Add a new enum value** — existing consumers fall through to a default handler
- **Widen a numeric range** — e.g. increase a score from 0–100 to 0–200 (only
  if consumers treat the value as continuous, not categorical)
- **Add a new type** — no existing consumer references it yet

### Breaking Changes

These require a coordinated migration (see Migration Process below):

- **Remove a field** — consumers that read it will break
- **Rename a field** — semantically equivalent to remove + add
- **Change a field's type** — e.g. string to integer
- **Remove an enum value** — consumers matching on it will break
- **Change a field from optional to required** — existing producers may not
  supply it
- **Narrow a constraint** — e.g. reduce maximum string length

## Versioning Strategy

Each type carries a version identifier. When a type changes, its version
increments. API endpoints include a version prefix (e.g. `/v1/search`,
`/v2/search`) so that multiple contract versions can coexist.

- **Type version** — embedded in the IDL. Incremented on every change
  (patch for compatible, minor for additive, major for breaking).
- **API version** — the endpoint prefix. Incremented only when a Tier 2
  breaking change is introduced. Multiple API versions can coexist behind
  the same deployment.
- **Engine transport version** — since Tier 3 types are internal, both sides
  (API and engine) are deployed together. A simpler protocol version counter
  suffices; no multi-version coexistence is required.

## Migration Process

When a breaking change is necessary:

1. **Add new version** — define the new type version alongside the old one in
   the IDL. Both versions are generated.
2. **Engine supports both** — the engine accepts requests in either version and
   returns responses in the requested version. Mapping between old and new
   happens in the engine's conversion layer.
3. **API serves both** — the API exposes both `/vN` and `/vN+1` endpoints.
   Both hit the same engine, translating as needed.
4. **Frontend upgrades** — the frontend switches to the new API version.
   During the transition, the old version remains available.
5. **Deprecate old** — after a defined sunset period (communicated via API
   deprecation headers), the old version is removed. The engine drops its
   backward-compatibility mapping. The old IDL version is archived.

## Relationship to Internal Versioning

The shared schema versions are distinct from engine-internal versions:

| Internal Version | What It Tracks | Schema Impact |
|-----------------|----------------|---------------|
| Scorer version | Scoring logic changes (`core/traits.md`) | None — internal |
| Weight set version | Learned parameter updates (`scoring/overview.md`) | Exposed in WeightSnapshot (Tier 3) |
| Embedding model version | Embedding generation changes (`compiler/overview.md`) | Exposed in compilation metadata (Tier 3) |
| Taxonomy version | Skill ontology changes (`algorithm/taxonomy/overview.md`) | May affect TaxonomyNode (Tier 2) |

Internal version changes do not require schema version bumps unless they alter
the shape of a Tier 2 or Tier 3 type.

## Schema Registry

A directory of versioned IDL files, one per version of each type. The registry
serves as the historical record:

- Current active version for each type
- Previous versions (for backward-compatibility mapping)
- Sunset dates for deprecated versions
- Change log entries linking each version to the PR that introduced it

The registry lives alongside the IDL files in the repository (likely in a
`schema/versions/` directory — unconfirmed). CI tooling reads the registry
to determine which versions to generate and which compatibility checks to run.

## Deployment Coordination

Schema version rollouts follow the canary process described in
`infrastructure/deployment.md`:

1. Shadow mode — new schema version runs alongside old; responses are logged
   but not served
2. Canary — small traffic fraction uses the new version
3. Gradual ramp — increase traffic, gated on error rates and latency
4. Full rollout — old version enters sunset period
