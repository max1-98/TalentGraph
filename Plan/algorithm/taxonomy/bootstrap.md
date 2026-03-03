# Taxonomy Bootstrap — ESCO Seed Import

The self-filling taxonomy starts empty. Without a seed, early CVs have poor
skill resolution — every mention is "unrecognised." ESCO provides a
machine-readable, actively maintained skill ontology to bootstrap from,
giving the system immediate coverage from day one.

Depends on: `graph-model/schema.md` (SkillNode, edge types),
`graph-model/contracts.md` (GraphMutator). Consumed by: SkillResolver,
`cv-to-profile/extraction.md` (implicit inference rules).

---

## ESCO as Seed Source

ESCO v1.2 provides approximately 13,890 skills/competences and 3,008
occupations, hierarchically organised. Available as RDF/SKOS bulk download
and REST API. No licensing cost. Actively maintained by the European
Commission.

Occupation-to-skill relationships (with importance ratings) provide the
foundation for implicit inference rules used in entity extraction.

---

## Import Pipeline

### Step 1 — Download

Fetch the ESCO SKOS dataset as a one-time bulk import. Periodic refresh for
new ESCO versions (typically annual). Downloaded data is stored as a
versioned snapshot for reproducibility.

### Step 2 — Map to SkillNode Schema

Each ESCO skill concept becomes a SkillNode:

- canonical_name = preferred English label
- embedding = generated from concatenation of (English name + ESCO
  description text) using the standard sentence encoder (likely
  all-MiniLM-L6-v2, ~22M parameters — unconfirmed)
- status = active (bypasses the normal admission gate — these are
  pre-validated by ESCO's editorial process)
- confidence = 0.8 (high but not 1.0 — corpus evidence may refine)
- source = "esco_bootstrap"
- usage_count = 0 (increments naturally as CVs reference them)

### Step 3 — Build Hierarchy

ESCO's broaderTransitive relations map to parent edges in the taxonomy
graph. Edge confidence = 0.7 (provisional). Corpus co-occurrence statistics
will validate, strengthen, or override these edges over time via the
hierarchy inference pipeline (`hierarchy/inference.md`).

### Step 4 — Generate Aliases

ESCO's altLabels (English) become alias edges via the union-find structure.
The ESCO preferred label is the canonical representative. Non-English labels
are stored as metadata but not indexed (English-only system for now).

### Step 5 — Compute Related Edges

ESCO's "related skill" relationships map to related edges. Initial NPMI
weight is estimated from ESCO's relatedness metadata, normalised to the
0–1 range expected by the schema. Only stored when the normalised weight
exceeds the minimum threshold defined in `graph-model/schema.md`.

---

## O*NET Crosswalk

The official ESCO-O*NET crosswalk supplements the ESCO seed:

- **Occupation mapping:** maps ESCO occupations to O*NET occupations,
  enabling cross-referencing of skill importance ratings.
- **Skill importance ratings:** O*NET provides finer-grained skill-to-
  occupation importance on a 1–5 scale. These seed the implicit inference
  confidence levels used in `cv-to-profile/extraction.md`: when a job title
  matches an occupation, that occupation's skills are inferred with
  confidence proportional to the O*NET importance rating (normalised to
  0–1).
- **Title enrichment:** O*NET occupation titles enrich the title-to-
  occupation matching dictionary, improving job title resolution accuracy.

---

## Reconciliation with Self-Filling Pipeline

Bootstrapped nodes are treated identically to any active node after import:

- **usage_count** increments normally as CVs reference them. Nodes that no
  CVs reference will eventually be reviewed for archival per the standard
  lifecycle (`graph-model/schema.md`).
- **New skills** discovered by the ingestion pipeline (emerging
  technologies, domain jargon not in ESCO) are added via the normal
  admission gate (`ingestion/admission.md`). ESCO coverage is a floor, not
  a ceiling.
- **Deduplication** may merge ESCO nodes if corpus evidence reveals
  redundancy — this is expected and correct. The deduplication pipeline
  (`deduplication/overview.md`) treats all nodes equally regardless of
  source.
- **Hierarchy refinement:** bootstrapped parent edges carry confidence 0.7
  and decay with a half-life of approximately 12 months. Corpus-validated
  edges replace them as co-occurrence data accumulates. The hierarchy
  inference pipeline (`hierarchy/inference.md`) does not distinguish
  bootstrapped from corpus-derived edges.

---

## Cold-Start Benefit

- **Day 1:** SkillResolver resolves approximately 13K skills immediately.
  No degraded onboarding experience for early adopters.
- **Implicit inference:** approximately 3K ESCO occupation-skill mappings
  seed the inference rules, enabling proficiency estimation from job titles
  before any corpus co-occurrence data exists.
- **Admission gate:** the Poisson anomaly gating (`ingestion/admission.md`)
  benefits from a populated taxonomy — the 0.80 similarity threshold for
  "known skill" classification has abundant reference nodes to compare
  against, reducing false promotions of near-duplicates.

---

## Cross-References

- SkillNode schema and edge types: `graph-model/schema.md`
- Graph mutation contracts: `graph-model/contracts.md`
- Admission gate for new skills: `ingestion/admission.md`
- Hierarchy inference: `hierarchy/inference.md`
- Deduplication: `deduplication/overview.md`
- Implicit inference using ESCO mappings: `cv-to-profile/extraction.md`
