# Hierarchy Inference — Edge Typing and DAG Enforcement

Combines the statistical signals from co-occurrence analysis into typed edge
proposals, validates them against the DAG constraint, and maintains the
hierarchy over time.

Depends on: `hierarchy/overview.md` (co-occurrence measures),
`graph-model/contracts.md` (HierarchyReader, GraphMutator).

---

## Edge Classification

The four signals from co-occurrence analysis are combined to propose typed
edges between skill pairs.

### Parent Edge (A is child of B)

All three conditions must hold:

1. **Strong asymmetry:** AR(A, B) > 2.0 — A appears with B far more often
   than B appears with A.
2. **Meaningful co-occurrence:** NPMI(A, B) > 0.3 — the association is
   substantially above chance.
3. **Distributional inclusion:** WeedsPrec(A->B) > 0.5 — A's co-occurrence
   profile is largely contained within B's.

**Confidence scoring:** a weighted combination of the four signals:

  raw_confidence = w_ar * norm(AR) + w_npmi * NPMI + w_wp * WeedsPrec + w_hearst * hearst_indicator

Where hearst_indicator is 1.0 if a Hearst pattern confirms this relationship,
0.0 otherwise. Weights (w_ar=0.30, w_npmi=0.20, w_wp=0.30, w_hearst=0.20)
are initial values, tunable on a validation set.

The raw confidence is then calibrated via **Platt scaling**: a logistic
function fitted on labelled correct/incorrect edge proposals to transform
raw scores into true probabilities. This ensures that a reported confidence
of 0.90 means approximately 90% of such proposals are correct.

### Related Edge (symmetric association)

- NPMI(A, B) > 0.3 — meaningful co-occurrence
- AR between 0.5 and 2.0 — roughly symmetric (neither subsumes the other)
- Confidence equals the NPMI score directly

### No Edge

Insufficient evidence: NPMI <= 0.3, or co-occurrence below minimum frequency
thresholds. No proposal is generated.

---

## DAG Enforcement

Before committing any parent edge, the system validates that it will not
create a cycle in the directed parent graph.

**Cycle detection:** before adding edge A->B (A is child of B), traverse all
descendants of A via HierarchyReader.descendants. If B appears among A's
descendants, the edge would create a cycle and is rejected.

**Conflict resolution:** when two conflicting proposals exist (A child of B
vs B child of A), the higher-confidence proposal wins. The weaker proposal
is downgraded to a "related" edge if its NPMI exceeds 0.3, or discarded
otherwise.

**Transitive reduction:** after each batch of edge additions, redundant
shortcut edges are removed. If A->B->C and A->C all exist as parent edges,
the direct A->C is redundant (the relationship is captured through B) and
is removed. This keeps the hierarchy clean and minimal.

---

## Confidence Decay and Re-evaluation

Edges are not permanent. The corpus evolves — skills shift in usage, new
relationships emerge, old ones weaken.

**Exponential decay:** between re-evaluation cycles, edge confidence decays:

  confidence(t) = confidence(t0) * exp(-lambda * (t - t0))

With a half-life of approximately 12 months (lambda = ln(2) / 365 days).

**Periodic re-scoring:** quarterly, all edges are re-evaluated against the
current corpus. The re-scored confidence replaces the decayed value. Edges
whose confidence has been reinforced by fresh data are strengthened; edges
no longer supported by the data continue to decay.

**Pruning:** edges falling below 0.20 confidence are removed via
GraphMutator.remove_edge. This ensures the taxonomy corrects itself over
time rather than accumulating stale relationships.

---

## RelationshipInferrer Trait Contract

- **Input:** co-occurrence measures for all skill pairs (from
  CooccurrenceAnalyser), current graph state (via HierarchyReader)
- **Output:** list of EdgeProposal (source_id, target_id, edge_type,
  confidence, supporting_evidence)
- **Side effects:** commits approved proposals via GraphMutator after
  DAG validation
- **Invariant:** no committed edge violates the DAG constraint. All
  confidence values are Platt-calibrated.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Full 4-signal inference, Platt calibration, decay | Normal operation |
| StatisticalOnly | AR + NPMI only, no WeedsPrec or Hearst | Reduced-data environments |
| Manual | Accepts human-provided edges, validates DAG only | Manual curation, testing |

---

## Batch Processing Order

Edge proposals are processed in descending confidence order. After each
committed edge, the DAG state is updated before evaluating the next proposal.
This ensures high-confidence edges take precedence and lower-confidence
proposals that would conflict with them are correctly rejected or downgraded.

---

## Future Enhancement: Poincare Embeddings

Hyperbolic (Poincare) embeddings naturally encode hierarchy: distance from
the origin represents depth, and distance between points represents
relatedness. Skill pairs with a parent-child relationship have a
characteristic signature in hyperbolic space.

The RelationshipInferrer trait is designed to accommodate this as a new
implementation without modifying existing pipeline logic — a new variant
would add Poincare distance as a fifth signal alongside AR, NPMI, WeedsPrec,
and Hearst patterns.

---

## Cross-References

- Co-occurrence analysis and signal computation: `hierarchy/overview.md`
- Graph structure and DAG invariant: `graph-model/schema.md`
- Mutation and traversal contracts: `graph-model/contracts.md`
- Confidence used downstream: `deduplication/overview.md` (edge transfer)
