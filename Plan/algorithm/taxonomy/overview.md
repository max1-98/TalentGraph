# Skill Taxonomy — Overview

A living, self-filling, self-correcting ontology of skills that underpins every
algorithm in the system. No manual curation — the taxonomy grows,
deduplicates, and reorganises itself from data.

---

## Problem

The matching engine operates over a ~5K-dimension skill space. Manually
curating thousands of skills, their aliases, hierarchies, and relationships
does not scale. Skills emerge constantly (new frameworks, methodologies,
certifications), existing names drift, and duplicates accumulate from
parallel ingestion sources.

## Architecture: Four Processes

The taxonomy is organised into four processes, each in its own subfolder.
The graph data model is foundational; the three pipelines operate over it
independently.

### Process 1: Graph Data Model (`graph-model/`)

The foundational data structure and its access interfaces. Every other process
depends on this; it depends on nothing outside `core/`.

| File | Description |
|------|-------------|
| [schema.md](graph-model/schema.md) | Node types (SkillNode, ClusterNode), four edge types (parent, alias, related, supersedes), invariants, growth bounds |
| [contracts.md](graph-model/contracts.md) | Four ISP sub-traits (SkillProvider, HierarchyReader, GraphMutator, ChangeObserver), SkillResolver contract, implementations |

### Process 2: Ingestion Pipeline (`ingestion/`)

How new skills enter the taxonomy. Triggered when the SkillResolver cannot
match a mention to an existing node.

| File | Description |
|------|-------------|
| [overview.md](ingestion/overview.md) | Three-stage discovery: text extraction, embedding generation, candidate classification |
| [admission.md](ingestion/admission.md) | Poisson anomaly gating, quality checks, initial placement, MarketSignalProvider integration |

### Process 3: Deduplication Pipeline (`deduplication/`)

How redundant nodes are detected and resolved. Combines four independent
signals for robust duplicate detection.

| File | Description |
|------|-------------|
| [overview.md](deduplication/overview.md) | Four-signal detection: embedding similarity, string similarity, context overlap, usage patterns |
| [merging.md](deduplication/merging.md) | Golden record merge (7 steps), downstream propagation, atomic rollback |

### Process 4: Hierarchy Inference (`hierarchy/`)

How the taxonomy discovers its own structure. Analyses co-occurrence patterns
to infer parent-child and peer relationships.

| File | Description |
|------|-------------|
| [overview.md](hierarchy/overview.md) | Co-occurrence matrix, four statistical measures: PMI/NPMI, asymmetric conditional probability, WeedsPrec, Hearst patterns |
| [inference.md](hierarchy/inference.md) | Edge classification, DAG enforcement, confidence decay, transitive reduction |

---

## Swappable Components (6 Traits)

Each pipeline stage is governed by a trait with production, fallback, and test
implementations — all drop-in replaceable (LSP).

| Trait | Process | Purpose | Defined In |
|-------|---------|---------|-----------|
| SkillExtractor | Ingestion | Extract skill mentions from text | `ingestion/overview.md` |
| AdmissionGate | Ingestion | Evaluate and gate candidate skills | `ingestion/admission.md` |
| DuplicationDetector | Dedup | Identify merge candidates via multi-signal scoring | `deduplication/overview.md` |
| MergeExecutor | Dedup | Execute golden-record merges with rollback | `deduplication/merging.md` |
| CooccurrenceAnalyser | Hierarchy | Compute PMI, WeedsPrec, and other statistical measures | `hierarchy/overview.md` |
| RelationshipInferrer | Hierarchy | Classify, validate, and commit typed edges | `hierarchy/inference.md` |

---

## SOLID Compliance

| Principle | Mechanism |
|-----------|-----------|
| **SRP** | Each file covers one concern: schema vs contracts, discovery vs admission, detection vs merging, analysis vs inference |
| **OCP** | New strategies via new trait implementations; no modification to pipeline logic |
| **LSP** | Every trait has production, fallback, and test variants; any is a drop-in replacement |
| **ISP** | TaxonomyGraph split into four sub-traits; consumers depend only on what they need |
| **DIP** | All pipelines depend on trait abstractions from `contracts.md`, never on concrete storage |

---

## Dependencies

- SkillResolver, MarketSignalProvider (shared building blocks — see
  `algorithm/overview.md`)
- Embedding model (same model used for CandidateTensor — see `core/types.md`)
- Corpus of parsed CVs and JDs (from `cv-to-profile/` and `compiler/`)

## Self-Correction Mechanisms

- **Low-usage archival:** nodes with zero references after a configurable
  window are archived (soft-deleted, recoverable)
- **Confidence decay:** hierarchy edges decay exponentially (half-life ~12
  months) and are pruned below 0.20 (see `hierarchy/inference.md`)
- **Merge rollback:** if a merge degrades downstream metrics, it is reversed
  within the 30-day retention window (see `deduplication/merging.md`)
- **Periodic re-evaluation:** the co-occurrence matrix is fully recomputed
  quarterly, correcting drift in hierarchy edges
