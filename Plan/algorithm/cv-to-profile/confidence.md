# CV-to-Profile — Bayesian Confidence Model

Stage 6 of the CV-to-Profile pipeline. Assigns per-field and aggregate
confidence scores (0–1) reflecting how much of the CandidateTensor is
grounded in explicit, reliable data vs heuristically inferred or defaulted.
Drives enrichment prioritisation and candidate messaging.

Depends on: `extraction.md` (ExtractionResult, SkillMention).

---

## Per-Field Confidence (source-based)

Each extracted field carries a base confidence from its extraction method:

- **Regex match** (dates, emails, phones): 0.95
- **Structured list parse** (skills section): taxonomy_match_similarity_score
  (typically 0.80–1.0)
- **Positional heuristic** (job title, company): 0.75
- **Narrative n-gram match** (skills from descriptions):
  embedding_similarity * 0.85 (discounted for narrative noise)
- **Implicit inference** (ESCO, dependency rules, co-occurrence):
  rule_confidence * source_confidence * 0.7 (double-discounted)

---

## Bayesian Confidence Update

When multiple independent extraction methods produce the same result,
confidence is updated using Bayes' rule:

- Prior: first method's confidence (p1)
- Likelihood ratio: p2 / (1 - p2), where p2 is the second method's
  confidence
- Posterior = (p1 * likelihood_ratio) / (p1 * likelihood_ratio + (1 - p1))

**Example:** a skill found in both the Skills section (confidence 0.90) and
narrative text (confidence 0.80) yields a Bayesian posterior of approximately
0.97.

This rewards convergent evidence without requiring ML. Subsequent agreeing
sources feed further Bayesian updates, with diminishing returns as confidence
approaches 1.0.

---

## Field Importance Weights

Weights reflect each field category's contribution to matching quality.
Sum to 1.0:

| Field Category | Weight |
|---------------|--------|
| Skills | 0.30 |
| Experience entries | 0.25 |
| Education | 0.10 |
| Job titles | 0.10 |
| Dates | 0.10 |
| Location | 0.05 |
| Certifications | 0.05 |
| Contact info | 0.03 |
| Other | 0.02 |

---

## Aggregate Confidence Formula

overall_confidence = SUM over all field categories i of:
  field_weight_i * mean_field_confidence_i * coverage_i

Where coverage_i = 1 if at least one value was extracted for field category
i, and 0 if entirely missing. A CV with zero extracted skills has
coverage_skills = 0, losing 30% of the maximum possible confidence
regardless of other fields.

---

## Enrichment Trigger Thresholds

| Range | Action |
|-------|--------|
| overall_confidence >= 0.70 | Tensor accepted, normal processing |
| 0.40 <= overall_confidence < 0.70 | Accepted, queued for background enrichment (see `profile-enrichment/overview.md`) |
| overall_confidence < 0.40 | Accepted with "low quality" flag; candidate prompted to review/update their CV |

---

## Schema Validation Checks

Before finalising the confidence score, sanity checks catch logical
inconsistencies:

- **Date sanity:** experience dates must not be in the future; end date must
  follow start date. Violations reduce the affected entry's confidence to
  0.3 and log a warning.
- **Salary range:** if stated, min must not exceed max. Violation flags the
  field as suspect (confidence = 0.2).
- **Skills count:** zero extracted skills triggers a warning regardless of
  overall confidence. The candidate tensor is still created but flagged for
  enrichment.
- **Duplicate detection:** same skill appearing multiple times with different
  proficiencies — highest proficiency is kept, duplicates are merged, and
  the Bayesian update combines their confidence values.

---

## ConfidenceScorer Trait Contract

**Input:** ExtractionResult (from EntityExtractor).

**Output:** ConfidenceReport containing: per-field confidence values
(field type, field value, confidence, extraction method), aggregate
confidence score (0–1), enrichment recommendation (none, background,
prompt_candidate), list of validation warnings.

**Invariant:** aggregate confidence is always in [0, 1] and equals the
weighted sum formula above. Field weights always sum to 1.0.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Bayesian update + validation checks | Normal operation |
| Flat | Fixed confidence (e.g. 0.5) for all fields | Baseline comparison |
| Test | Returns hardcoded ConfidenceReport | Unit testing |

---

## ProficiencyEstimator Trait Contract

Separated from ConfidenceScorer (ISP) — estimates skill-level proficiency,
not extraction reliability. Input: SkillMention + context (experience
entries, years spanned, recency, seniority). Output: proficiency score
(0–1) per skill, feeding the P factor in `core/types.md`. Invariant:
output always in [0, 1]; missing context defaults to 0.5.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Heuristic formula (see `extraction.md`) | Normal operation |
| Uniform | Returns 0.5 for all skills | Baseline / A-B testing |
| Test | Returns hardcoded proficiencies | Unit testing |

---

## Cross-References

- Extraction outputs and proficiency formula: `extraction.md`
- Skill score formula (P factor): `core/types.md`
- Profile enrichment triggers: `profile-enrichment/overview.md`
