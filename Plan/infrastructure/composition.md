# Pipeline Composition

The pipeline is assembled at startup from configuration. The composition root
is the only place in the system that knows about concrete types — everything
else talks to traits.

---

## The Pipeline Engine

The pipeline itself is simple: a list of Stage trait objects executed in
sequence. It compiles the query, then passes a candidate set through each
stage. Each stage narrows the set. Diagnostics are recorded at every step.

- If any stage produces zero survivors, the pipeline returns early with an
  empty result and full diagnostics
- The pipeline does not know what its stages do — it only knows that they
  implement the Stage trait

## The Composition Root

A single builder function wires concrete implementations to trait boundaries.
This is the only code that imports concrete types.

Decisions made at assembly time (all driven by config):

| Component | Config Key | Options |
|-----------|-----------|---------|
| Tensor store | storage.backend | memory-mapped, in-memory |
| Vector indexes | index.backend | HNSW (with M, ef params), brute-force |
| Query compiler | query_compiler.backend | LLM-backed, rule-based, fixed |
| Weight provider | weights.backend | fixed, learned, A/B test |
| Team selector | team_selection.backend | greedy submodular, top-N |
| Stage order | stages.pipeline | ordered list of stage names |
| Active scorers | scorers.enabled | ordered list of scorer IDs |

## Configuration Format

A single configuration file (likely TOML — unconfirmed) describes the entire
pipeline. Example structure:

- `[storage]` — backend type and path
- `[index]` — backend type, M, ef_construction
- `[query_compiler]` — backend, model, fallback
- `[weights]` — backend, model path (or A/B test variants + traffic split)
- `[team_selection]` — backend
- `[stages]` — ordered list of stage names
- `[scorers]` — ordered list of enabled scorer IDs

Commenting out a scorer disables it. Uncommenting enables it. No code change.

## Scorer Registry

All available scorers are registered at startup, even if not currently enabled.
The registry maps scorer IDs to Scorer trait objects.

- `active_scorers()` returns only the scorers listed in the enabled config,
  in config order
- `composite_version()` hashes the versions of all active scorers — changes
  when any scorer's logic changes, used for cache invalidation

The precision stage retrieves active scorers from the registry and runs them
generically. It does not know their names, count, or internal logic.

## Hot Reload

Configuration changes (enabling/disabling scorers, swapping weight sets,
adjusting parameters) can take effect without restarting the engine. A file
watcher detects config changes and rebuilds the affected components.

This enables:
- Instant scorer rollback (swap v3 back to v2 via config)
- A/B test traffic changes without deployment
- Parameter tuning in production

## Compilation Impact

| Change | Recompiles | Safe From |
|--------|-----------|-----------|
| One scorer file | scorers → precision → pipeline | filter, narrow, team, storage |
| Core trait change | Everything | Nothing |
| Storage module | storage → pipeline | All scorers, all stages |
| Team module | team → pipeline | Everything else |
| Config only | Nothing | Everything (hot reload) |

The critical invariant: the core module changes rarely. It defines contracts.
Everything else evolves at the edges without touching the centre.
