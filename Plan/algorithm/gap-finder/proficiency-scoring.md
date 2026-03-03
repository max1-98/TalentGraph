# Gap Finder — Proficiency Scoring

Continuous gap measurement that replaces binary has/hasn't with nuanced deltas,
accounting for skill decay, context-dependent thresholds, and gap types.

Depends on: `contracts.md` (ProficiencyGapScorer trait), `scoring/
skill-coverage.md` (f1 skill model), `taxonomy/graph-model/schema.md`
(SkillNode attributes).

---

## Continuous Delta

For each required skill g, the gap delta is:

  Delta(g) = max(0, RequiredProficiency(g) - EffectiveProficiency(g))

When Delta(g) = 0, the candidate meets or exceeds the requirement and the
skill is excluded from the gap list. This formula replaces the binary
missing/present classification from ProfileComparator with a continuous
measure suitable for prioritisation and bridge scoring.

---

## Decay-Aware Proficiency

Candidate proficiency degrades over time when a skill is not actively used.
The effective proficiency at time t is:

  EffectiveProficiency(g, t) = raw_proficiency(g) * exp(-lambda(g) * age(g))

Where age(g) is the time since the skill was last demonstrated (last job using
it, last project, or last certification). The decay rate lambda varies by
skill type, derived from TaxonomyGraph node properties:

**Leaf / specific skills** (frameworks, specific tools): higher lambda.
These evolve rapidly; a 3-year-old framework version is substantially less
relevant than a current one.

**Foundational skills** (algorithms, design patterns, mathematics): lower
lambda. These remain relevant over long periods.

**Skills with successors** (supersedes edges in TaxonomyGraph): accelerated
decay. If a skill has been superseded, its effective proficiency decays faster,
reflecting reduced market relevance.

Lambda derivation: base_lambda is a configurable constant (initial value
tuned so that half-life is approximately 3 years for foundational skills).
Adjusted by: depth_factor * node_depth (deeper/more specific nodes decay
faster) + successor_penalty if a supersedes edge exists pointing away.

---

## Context-Dependent Thresholds

The required proficiency for each gap is not fixed — it scales with role
context:

  RequiredProficiency(g) = base_threshold(importance) * seniority_factor * depth_factor

**Base threshold by importance tier** (from RoleContextAnalyser output):
- Required skills: 0.7
- Preferred skills: 0.4
- Nice-to-have skills: 0.2

**Seniority scaling factor** (from target role seniority):
- Junior: 0.7x — lower bar acknowledges growth potential
- Mid: 1.0x — baseline
- Senior: 1.2x — higher expectations
- Principal / Staff: 1.4x — near-expert proficiency expected

**Depth scaling factor** (from TaxonomyGraph node depth):
- Root-level / broad skills: 0.8x — broad skills are harder to master fully
- Leaf-level / specific skills: 1.0x — specific skills have clearer mastery

The product is clamped to the range 0.1–1.0.

---

## Gap Type Classification

Each gap is classified into one of five types based on the proficiency delta
and its cause:

**Missing** — EffectiveProficiency = 0. The candidate has no evidence of this
skill. Delta equals the full RequiredProficiency.

**Weak-slight** — Delta > 0 and Delta < 0.2 * RequiredProficiency. The
candidate has the skill but falls slightly short. Often closeable through
practice rather than formal training.

**Weak-significant** — Delta >= 0.2 * RequiredProficiency and the skill was
not previously strong. A meaningful proficiency shortfall requiring deliberate
development.

**Decayed** — The candidate previously demonstrated strong proficiency
(raw_proficiency >= RequiredProficiency) but time-based decay has reduced
EffectiveProficiency below the threshold. Identified by comparing raw vs
effective proficiency. These gaps are typically easier to close than missing
skills — the candidate is re-learning, not learning from scratch.

**Experience** — Career trajectory mismatch rather than skill deficiency. The
candidate has the skill but lacks the context: wrong seniority level, wrong
industry application, or insufficient duration. Derived from the f4 career
trajectory components rather than the skill vector.

---

## Confidence Propagation

Each gap carries a confidence score reflecting how reliable the delta is:

  Confidence(g) = tensor_confidence * skill_evidence_factor

Where tensor_confidence is the overall CandidateTensor confidence (from
`cv-to-profile/confidence.md`) and skill_evidence_factor reflects how well the
specific skill is attested:

  skill_evidence_factor = min(1, chunk_mentions(g) / 3)

A skill mentioned in 3 or more independent CV chunks (different jobs,
projects, or sections) receives full evidence weight. Skills mentioned only
once receive 0.33 — the system is less certain about the proficiency estimate.

Confidence flows downstream: the prioritiser uses it as an uncertainty penalty
(see `prioritisation.md`), and the API response surfaces it so consumers can
distinguish high-confidence gaps from speculative ones.

---

## Cross-References

- Trait contract and variants: `contracts.md` (ProficiencyGapScorer)
- Candidate proficiency source: `scoring/skill-coverage.md` (f1)
- Skill decay rates derived from: `taxonomy/graph-model/schema.md` (node
  depth, supersedes edges)
- Tensor confidence model: `cv-to-profile/confidence.md`
- Career trajectory components for experience gaps: `scoring/
  career-trajectory.md` (f4)
- Consumed by: `semantic-detection.md`, `prioritisation.md`
