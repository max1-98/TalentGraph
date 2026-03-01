# Architecture — SOLID Principles Applied

Every SOLID principle maps to a concrete architectural decision in the matching
engine. This is not abstract — each principle prevents a specific class of
"I changed X and broke Y" failures.

---

## Single Responsibility

Each scorer does exactly one thing. The SkillCoverageScorer computes skill
coverage. It does not know about embeddings, location, or its position in
the pipeline. It receives a tensor and query, returns a float. If the skill
formula changes, one file changes. Nothing else in the system is aware.

**Prevents:** "I improved the skill scorer and accidentally broke career
trajectory scoring because they shared a utility function."

## Open/Closed

New scorers are added without modifying existing code. Implement the Scorer
trait, register it, add to config. Existing scorers are never touched. The
pipeline discovers scorers from a registry. The weight vector grows by one
dimension automatically.

**Prevents:** "Adding a new signal requires changes in 12 files across
4 modules."

## Liskov Substitution

Any implementation of a trait is a drop-in replacement. HNSW and brute-force
both implement VectorIndex. Memory-mapped and in-memory both implement
TensorStore. The pipeline does not know which implementation it is using.
Swap freely between production and test implementations.

**Prevents:** "We can't test the scoring pipeline without loading a 4 GB
index."

## Interface Segregation

Traits are narrow. Stage has one core method (execute). Scorer has one core
method (score). VectorIndex has two methods (search, search_filtered). No
trait has 15 methods. No component implements capabilities it does not need.
The filter stage does not implement scoring. The scorer does not implement
filtering.

**Prevents:** "The filter struct has a score method that panics because it
was forced to implement the FullStage trait."

## Dependency Inversion

Everything depends on abstractions, not concretions. The precision stage
depends on the Scorer trait, not on SkillCoverageScorer. The pipeline depends
on a list of Stage trait objects, not on a hardcoded sequence. The tensor
compiler depends on an EmbeddingModel trait, not on a specific model. Swap
the embedding model and the compiler does not change.

**Prevents:** "Upgrading the embedding model requires changes in the scorer,
the compiler, and the index builder."

## Module Dependency Graph

The core module (traits + types) is the leaf. Everything depends on it, it
depends on nothing.

```
              tg-core (traits + types)
                   |
    +---------+----+----+---------+---------+---------+
    |         |         |         |         |         |
  filter   narrow   scorers    team     storage   compiler
    |         |         |         |         |         |
    |         |     precision     |         |         |
    |         |         |         |         |         |
    +----+----+----+----+---------+---------+---------+
                   |
              tg-pipeline (composition root)
              ONLY place with concrete types
```

Scorers do not depend on storage or filter. Scorers compile and test with
zero infrastructure. The composition root is the only module that imports
concrete implementations.

## Change Impact Summary

| Change | Touches | Everything Else |
|--------|---------|----------------|
| Improve one scorer | One file, bump version | Unchanged |
| Add new scorer | New file + register + config | Unchanged |
| Remove a scorer | Config change | Unchanged |
| Swap vector index | New impl + config | Unchanged |
| Swap embedding model | New impl + tensor rebuild | Scorers unchanged |
| Insert new pipeline stage | New impl + config | Existing stages unchanged |
