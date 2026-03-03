# CV-to-Profile — Quality Assurance & Evaluation

Cross-cutting concern. Defines how parsing quality is measured, monitored,
and continuously improved without relying on expensive human review at scale.
Every change to extraction patterns, embedding models, or taxonomy thresholds
must pass automated regression testing.

Depends on: `extraction.md` (ExtractionResult), `confidence.md`
(ConfidenceReport, field weights).

---

## Evaluation Dataset

A labelled set of CVs with ground-truth extractions:

- **Seed:** 100–200 CVs manually labelled across industries (technology,
  finance, healthcare, manufacturing) and seniority levels (junior to
  executive), all compliant with the enforced CV standard (`parsing.md`).
- **Ground truth format:** for each CV, the exact list of entities that
  should be extracted — skills (with taxonomy IDs), experience entries
  (title, company, dates), education entries, contact info, certifications.
  Each entity includes its source Section and character offsets.
- **Growth mechanism:** when candidates correct auto-extracted data via the
  UI, the correction is logged as a (predicted, actual) pair. Accepted
  corrections become new ground-truth examples after human review of a
  random sample (to filter erroneous corrections).
- **Versioning:** the evaluation dataset is versioned alongside the
  extraction pipeline. Each pipeline version is evaluated against the
  dataset version it was tested with.

---

## Automated Evaluation Pipeline

### Entity Alignment

Predicted and ground-truth entities are matched using the Hungarian
algorithm (optimal one-to-one assignment minimising total cost). This
handles order mismatches and quantity discrepancies — predicted entities
are aligned to their best matching ground-truth counterparts.

Cost function for alignment: 1 - field_similarity, where field_similarity
is computed per entity type (see matching rules below).

### Field-Specific Matching Rules

- **Dates:** normalised comparison. Ignore day when precision is month-only.
  Two dates match if they refer to the same (year, month) tuple or, for
  year-only precision, the same year.
- **Names and titles:** Jaro-Winkler similarity with threshold 0.90.
  Accounts for minor spelling variations and abbreviations.
- **Taxonomy IDs:** exact match. A skill resolved to the wrong taxonomy
  node is a miss even if the raw text is close.
- **Free text fields** (company name, institution): Jaro-Winkler similarity
  with threshold 0.85.

### Per-Field Metrics

For each entity type (skills, experience, education, contact, certification):

- **Precision:** correctly extracted entities / total predicted entities
- **Recall:** correctly extracted entities / total ground-truth entities
- **F1:** harmonic mean of precision and recall

### Aggregate Accuracy

Weighted F1 using the same field importance weights as the confidence model
(`confidence.md`): Skills 0.30, Experience 0.25, Education 0.10, Job titles
0.10, Dates 0.10, Location 0.05, Certifications 0.05, Contact 0.03,
Other 0.02.

---

## Continuous Monitoring (production metrics)

Four signals tracked in production, with alerts on significant shifts:

**Extraction confidence distribution:** track mean and variance of
overall_confidence over rolling windows (daily, weekly). A downward shift
indicates data distribution drift or extraction degradation.

**SkillResolver hit rate:** percentage of skill mentions that resolve to an
existing taxonomy node (vs becoming ingestion candidates). Declining hit
rate suggests taxonomy gaps — may trigger a review of emerging skill
categories or a taxonomy refresh.

**Rejection rate:** percentage of uploaded CVs failing validation. Rising
rate may indicate user confusion about the enforced standard — improve
guidance messaging or review whether a requirement is too strict.

**Candidate correction rate:** how often candidates modify auto-extracted
data in the UI. High correction rate for a specific field type indicates
extraction weakness. Per-field correction rates are tracked separately to
pinpoint which extraction rules need refinement.

---

## Regression Testing

Every change to extraction patterns, the embedding model, taxonomy
thresholds, or confidence weights must pass the evaluation dataset:

- **Automated comparison:** run both old and new extraction on the full
  evaluation set. Compute per-field metrics for both. Flag any metric that
  regresses by more than 1 percentage point.
- **Approval gate:** regressions in any field's F1 score block deployment
  unless explicitly overridden with justification (e.g. intentional
  trade-off documented in the change proposal).
- **A/B validation:** for changes expected to improve one metric at the
  cost of another (e.g. higher skill recall at slightly lower precision),
  the net effect on aggregate weighted F1 must be positive.

---

## Cross-References

- Extraction output format: `extraction.md` (ExtractionResult)
- Confidence model and field weights: `confidence.md`
- Document validation standard: `parsing.md`
- Pipeline overview: `overview.md`
