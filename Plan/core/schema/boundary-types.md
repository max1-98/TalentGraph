# Shared Schema — Boundary Type Inventory

Every type that crosses a service boundary, classified by tier, with pointers
to existing documentation and field-level definitions.

---

## Tier 2 — API Contracts (frontend ↔ API)

Types visible to external consumers. Full field-level definitions are in
`api-contracts-core.md` and `api-contracts-algorithms.md`.

| Type | Purpose | Boundaries | Defined In |
|------|---------|------------|------------|
| **ProfilePayload** | Structured candidate profile for display and editing | Frontend ↔ API | `api-contracts.md` |
| **CVUploadRequest** | Document upload with extraction preferences | Frontend → API | `api-contracts.md` |
| **CVUploadResponse** | Extracted profile data returned after CV processing | API → Frontend | `api-contracts.md` |
| **SearchRequest** | Job description + filters submitted by recruiter | Frontend → API | `api-contracts.md` |
| **SearchResult** | Ranked candidate list with scores and breakdowns | API → Frontend | `api-contracts.md` |
| **ScoreBreakdownDisplay** | Per-candidate score dimensions with display labels | API → Frontend | `api-contracts.md` |
| **FeedbackSignal** | Outcome signal tied to a search and candidate | Frontend → API | `api-contracts.md` |
| **UserIntelligenceReport** | Aggregated career insights for a candidate | API → Frontend | `api-contracts.md` |
| **TaxonomyNode** | Single skill in the ontology (admin CRUD shape) | Frontend ↔ API | `api-contracts.md` |
| **GapAnalysisResult** | Prioritised skill gaps with bridge suggestions | API → Frontend | `api-contracts.md` |
| **CareerPathSuggestion** | Ranked next-move option with evidence and gaps | API → Frontend | `api-contracts.md` |
| **RoleRecommendation** | Matching role with score breakdown | API → Frontend | `api-contracts.md` |
| **CofounderMatchResult** | Complementary candidate with gap annotations | API → Frontend | `api-contracts.md` |
| **TeamAssignment** | Per-vacancy shortlist from multi-vacancy search | API → Frontend | `api-contracts.md` |

### Existing documentation cross-references

- ProfilePayload fields derive from the identity and skill sections of
  CandidateTensor (`core/types.md`), projected into a display-safe shape.
- SearchRequest maps to the inputs described in `pipeline/overview.md` and
  `compiler/overview.md`.
- ScoreBreakdownDisplay maps from the internal ScoreBreakdown
  (`core/types.md`) — raw floats become named, 0–100 integer dimensions.
- FeedbackSignal corresponds to the signal hierarchy in `learning/feedback.md`.
- UserIntelligenceReport corresponds to the aggregated data described in
  `features/user-intelligence.md`.
- GapAnalysisResult corresponds to the output of `algorithm/gap-finder/overview.md`.
- CareerPathSuggestion corresponds to the output of `algorithm/career-pathing/overview.md`.
- RoleRecommendation corresponds to the output of `algorithm/role-recommendation/overview.md`.
- CofounderMatchResult corresponds to the output of `algorithm/cofounder-matching/overview.md`.
- TeamAssignment corresponds to the output of `features/team-selection.md`.

---

## Tier 3 — Engine Transport (API ↔ engine)

Internal protocol types. Not exposed to the frontend. Full field-level
definitions are in `engine-transport.md`.

| Type | Purpose | Boundaries | Defined In |
|------|---------|------------|------------|
| **EngineSearchRequest** | JD + filters + internal metadata for the engine | API → Engine | `engine-transport.md` |
| **EngineSearchResponse** | Raw scored candidates + compilation metadata | Engine → API | `engine-transport.md` |
| **PipelineDiagnostics** | Per-stage counts and durations for one search | Engine → API | `engine-transport.md` |
| **ProfileChangeEvent** | Notification that a candidate profile was updated | API → Engine (via message stream) | `engine-transport.md` |
| **WeightSnapshot** | Current weight set state for admin inspection | Engine → API | `engine-transport.md` |
| **AlgorithmInvocation** | Request envelope wrapping algorithm-specific inputs | API → Engine | `engine-transport.md` |

---

## Tier 1 — Engine-Internal (excluded from shared schema)

These types are optimised for compute performance and memory layout. They are
**not** part of the shared schema — changes to them do not require
cross-boundary coordination.

| Type | Reason for Exclusion | Defined In |
|------|---------------------|------------|
| CandidateTensor | Dense numerical layout, ~4 KB, SIMD-optimised | `core/types.md` |
| FilterPack | 32-byte packed struct, cache-line aligned | `core/types.md` |
| QueryTensor | Mirrors CandidateTensor dimensions for pairwise ops | `core/types.md` |
| WeightSet | Learned parameters scoped to engine internals | `core/types.md` |
| ScoreBreakdown (raw) | Per-scorer floats; projected to ScoreBreakdownDisplay for API | `core/types.md` |
| HNSW graph internals | Index structure, never leaves the vector search layer | `storage/vector-index.md` |

The engine implements mapping functions between Tier 1 and Tier 2/3 types.
For example, a raw ScoreBreakdown (floats, internal scorer IDs) is mapped to
a ScoreBreakdownDisplay (named dimensions, 0–100 range, display labels) before
leaving the engine.
