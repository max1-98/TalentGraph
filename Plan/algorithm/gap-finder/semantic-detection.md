# Gap Finder — Semantic Detection

Extends proficiency-based gap analysis beyond exact skill matching. Uses
embedding similarity, co-occurrence intelligence, and synergy analysis to find
partial coverage, infer unstated skills, and detect co-occurrence gaps.

Depends on: `contracts.md` (SemanticGapDetector trait), `proficiency-scoring.md`
(ProficiencyGap inputs), `taxonomy/graph-model/contracts.md` (SkillProvider,
HierarchyReader), `taxonomy/hierarchy/overview.md` (NPMI co-occurrence matrix).

---

## Soft Skill Matching

For each skill r classified as "missing" by ProficiencyGapScorer, search for
the closest candidate skill using embedding similarity:

  SemanticProximity(r) = max over all c in CandidateSkills of
    cosine(embedding(c), embedding(r))

If SemanticProximity(r) exceeds theta_soft (configurable, initial value 0.75),
the gap is reclassified from "missing" to "partial coverage" and the delta is
reduced:

  AdjustedDelta(r) = Delta(r) * (1 - gamma_soft * SemanticProximity(r))

Where gamma_soft controls the maximum reduction (initial value 0.6). At
proximity = 1.0, the delta shrinks by 60% — close but not identical skills
meaningfully reduce the gap.

The matching candidate skill and its proximity score are recorded in the
semantic_notes field for transparency ("partially covered by React experience,
proximity 0.82").

Uses SkillProvider.search_by_embedding internally — no new index needed.

---

## Latent Skill Inference

Some skills a candidate possesses imply proficiency in related skills they
have not explicitly listed. The NPMI co-occurrence matrix (from
`taxonomy/hierarchy/overview.md`) captures these statistical relationships.

For each gap skill g, compute an inferred proficiency from the candidate's
existing skills:

  InferredProficiency(g) = SUM over c of (NPMI(c, g) * proficiency(c))
                           / SUM over c of (max(0, NPMI(c, g)))

Where c ranges over candidate skills with positive NPMI against g. Only
positive NPMI values contribute — negatively correlated skills are excluded.

If InferredProficiency(g) exceeds theta_infer (configurable, initial value
0.5), the gap is annotated as "possibly unstated" — the candidate likely has
this skill but did not list it. This reduces the gap's priority (see
`prioritisation.md`) but never eliminates it entirely, since inference is
probabilistic.

**Evidence transparency:** the response always lists which candidate skills
contributed to the inference and their individual NPMI scores. This lets
consumers (and candidates) understand and challenge the inference.

---

## Synergy Gap Detection

Reuses the f3 skill synergy logic from `scoring/skill-synergy.md`. Where f3
scores overall co-occurrence alignment, synergy gap detection identifies
specific skill pairs where the candidate's co-occurrence falls short:

  SynergyGap(a, b) = max(0,
    target_cooccurrence(a, b) - candidate_cooccurrence(a, b))

Target co-occurrence is derived from the role's typical skill pairings (from
the NPMI matrix filtered to role-relevant skills). Candidate co-occurrence
comes from chunk-level co-occurrence in the CandidateTensor.

Non-zero synergy gaps indicate: "You know A and B separately, but the role
expects them used together." These are distinct from proficiency gaps — the
candidate has both skills but lacks demonstrated combined experience.

Synergy gaps feed directly into UserIntelligenceReport.synergy_gaps (see
`features/user-intelligence.md`) and receive their own section in the gap
analysis response.

---

## Cluster-Level Gap Analysis

Individual skill gaps may miss thematic patterns. Cluster-level analysis
identifies broad capability areas where the candidate falls short.

**Method:** compute the residual vector between the role's query embedding
(embed_domain or embed_role from the QueryTensor) and the candidate's
embed_skills projection. Search this residual vector against TaxonomyGraph
ClusterNode embeddings using SkillProvider.search_by_embedding.

The top matching clusters represent thematic gaps — broad areas like "cloud
infrastructure" or "data engineering" that the candidate lacks. These are
reported as high-level annotations alongside the individual skill gaps,
giving career coaches a strategic view ("lacks cloud infrastructure
experience") beyond the tactical skill-by-skill list.

Cluster-level gaps do not replace individual gaps — they augment them. A
candidate might have no individual "missing" skills in a cluster but still
show a thematic gap due to shallow coverage across many related skills.

---

## Configuration

All thresholds are exposed via the system's configuration layer:

- theta_soft = 0.75 — minimum embedding proximity for soft match
- gamma_soft = 0.6 — maximum delta reduction from soft match
- theta_infer = 0.5 — minimum inferred proficiency to flag as possibly unstated
- Minimum NPMI for inference contribution = 0.1

Initial values are hand-tuned. Refined via A/B testing on gap report
engagement metrics (click-through on bridge suggestions, gap report
completion rate).

---

## Cross-References

- Trait contract and variants: `contracts.md` (SemanticGapDetector)
- Input proficiency gaps: `proficiency-scoring.md`
- NPMI co-occurrence matrix: `taxonomy/hierarchy/overview.md`
- f3 synergy scoring logic: `scoring/skill-synergy.md`
- Synergy gap consumer: `features/user-intelligence.md`
- Embedding search: `taxonomy/graph-model/contracts.md` (SkillProvider)
- Consumed by: `prioritisation.md` (adjusted deltas and inference flags)
