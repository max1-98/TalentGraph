# Hierarchy Inference — Overview (Co-occurrence Analysis)

How the taxonomy discovers its own structure from data. This process analyses
how skills co-occur across the document corpus to infer parent-child, peer,
and unrelated relationships. The taxonomy self-corrects as data evolves.

Depends on: `graph-model/contracts.md` (SkillProvider, HierarchyReader,
GraphMutator).

---

## Co-occurrence Matrix

The foundation is a sparse co-occurrence matrix: for every pair of active
skills, how many documents contain both?

- **Dimensions:** N_active x N_active (typically 5K–10K active nodes).
- **Storage:** sparse format (typically 50K–200K non-zero entries). Only pairs
  with at least 5 co-occurrences are stored.
- **Update modes:**
  - *Incremental:* as each new document is processed, update the relevant
    cells. Efficient for steady-state operation.
  - *Full recomputation:* periodically (e.g. quarterly) recompute the entire
    matrix from the corpus to correct drift from incremental updates,
    document deletions, and skill merges.

---

## Four Statistical Measures

### 1. PMI / PPMI / NPMI

Pointwise Mutual Information measures how much more often two skills co-occur
than independence would predict:

  PMI(A, B) = log2( P(A,B) / (P(A) * P(B)) )

Where P(A,B) is the fraction of documents containing both, P(A) is the
fraction containing A alone.

- **PPMI** (Positive PMI) = max(0, PMI). Negative PMI is unreliable with
  limited data, so clamped to zero.
- **NPMI** (Normalised PMI) = PMI / (-log2(P(A,B))). Normalises to [-1, 1]
  regardless of skill frequency. NPMI = 1 means perfect co-occurrence.
  This normalisation makes scores comparable across skill pairs of different
  frequencies, which raw PMI does not.

**Minimum frequency filters:** count(A) >= 20, count(B) >= 20,
count(A,B) >= 5. Below these thresholds, PMI estimates are too noisy.

### 2. Asymmetric Conditional Probability

Conditional probability reveals directionality:

  P(A|B) = count(A,B) / count(B)

The asymmetry ratio AR(A,B) = P(A|B) / P(B|A) reveals which skill is broader:

- AR >> 1: A is likely a specialisation (child) of B
- AR approximately equal to 1: symmetric (peers)
- AR << 1: B is likely a specialisation of A

Example: "React Hooks" almost always appears with "React", but "React" often
appears without "Hooks" — so AR(Hooks, React) is high, suggesting Hooks is
a child of React.

### 3. Distributional Inclusion (WeedsPrec)

A more robust test for parent-child relationships. For each skill, build a
context profile: the set of other skills it co-occurs with, weighted by PPMI.
Then measure how much of A's profile is included in B's:

  WeedsPrec(A->B) = SUM_f( min(PPMI(A,f), PPMI(B,f)) ) / SUM_f( PPMI(A,f) )

If A is a specialisation of B, then everything A co-occurs with, B also
co-occurs with (the parent's world subsumes the child's). So WeedsPrec(A->B)
is high, but WeedsPrec(B->A) is low.

This is more robust than simple conditional probability because it considers
the full distributional profile, not just direct co-occurrence between A and B.

### 4. Hearst Patterns

Lexico-syntactic templates that directly state hypernym relationships in text:

- "X such as Y" implies Y is a child of X
- "Y is a type of X" implies Y is a child of X
- "X including Y" implies Y falls under X
- "Y and other X" implies Y is a child of X

Hearst patterns have **very high precision** (almost always correct when
found) but **very low recall** (most relationships are never stated this way).
They supplement the statistical measures with direct textual evidence.

Extraction runs during document ingestion and results are stored as
(parent_text, child_text, pattern_type, source_document) tuples for use
during inference.

---

## CooccurrenceAnalyser Trait Contract

- **Input:** the document corpus (or incremental update), current active
  SkillNode set (via SkillProvider)
- **Output:** updated co-occurrence matrix and all four derived measures
  (NPMI, AR, WeedsPrec, Hearst evidence) for each skill pair
- **Invariant:** frequency filters are applied before computing derived
  measures. Pairs below minimum thresholds return no signal (not zero).

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Incremental + periodic full recompute, all 4 measures | Normal operation |
| BatchOnly | Full recompute only, no incremental updates | Initial bootstrap, testing |
| Precomputed | Loads from a static file, no computation | Unit testing, benchmarking |

---

## Cross-References

- Edge typing and DAG enforcement: `hierarchy/inference.md`
- Graph mutation and traversal: `graph-model/contracts.md`
- Initial placement of new skills: `ingestion/admission.md`
- NPMI used for related-edge weights: `graph-model/schema.md`
