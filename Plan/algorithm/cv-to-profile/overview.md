# CV-to-Profile — Overview

Transforms an unstructured CV document into a structured CandidateTensor
suitable for scoring, searching, and comparison.

---

## Problem

Candidates submit CVs in wildly different formats — PDF, Word, plain text,
LinkedIn exports — with inconsistent structure, varying levels of detail, and
no standard vocabulary. The matching engine requires a fixed-format
CandidateTensor (see `core/types.md`).

## Approach: Multi-Stage Pipeline

### Stage 1 — Parse Document

Accept the raw document and produce clean, segmented text. Handle format
variations (PDF extraction, DOCX parsing, HTML stripping). Output: a sequence
of text chunks with rough section labels (experience, education, skills, etc.).

### Stage 2 — Extract Entities

From each chunk, extract structured entities: job titles, company names, dates,
skill mentions, certifications, education, location. This stage operates on
text, not taxonomy — it produces raw strings.

### Stage 3 — Resolve Skills

Pass raw skill mentions through SkillResolver (shared building block) to map
them to canonical taxonomy IDs. Unknown skills are flagged as taxonomy
ingestion candidates (see `taxonomy/overview.md`).

### Stage 4 — Generate Embeddings

Produce the four 384-dimensional embeddings that populate the CandidateTensor:
skills embedding, experience embedding, education embedding, and full-profile
embedding. Uses the same embedding model as query compilation.

### Stage 5 — Assemble Tensor

Populate all CandidateTensor fields: skill bit vector, embeddings, career
trajectory features (seniority estimate, industry codes, tenure statistics),
identity fields, and practical-fit fields (location, salary expectation if
stated, availability).

### Stage 6 — Confidence Score

Assign an overall confidence score (0 to 1) reflecting how much of the tensor
was populated from explicit data vs inferred or defaulted. Low-confidence
tensors are flagged for enrichment (see `profile-enrichment/overview.md`).

## Swappable Components (7 Traits)

| Trait | Responsibility | Day-1 Impl | Upgrade Path |
|-------|---------------|------------|--------------|
| **DocumentValidator** | Enforce CV standard, reject non-compliant | Rule-based: character analysis, heading dictionary, Jaro-Winkler | ML-based language detection, layout classification |
| **TextExtractor** | Raw bytes to ordered text with metadata | Lightweight PDF/DOCX library | Layout-aware parser, OCR engine |
| **SectionSegmenter** | Text blocks to labelled sections | Heading score function + dictionary | Trained classifier (BiLSTM-CRF, transformer) |
| **EntityExtractor** | Sections to structured entities | Regex + positional heuristics + n-grams | Fine-tuned NER model, LLM-backed extractor |
| **SkillMatcher** | Raw skill text to taxonomy node | Embedding cosine similarity + BM25 | Re-ranking model, cross-encoder |
| **ProficiencyEstimator** | Skill context to proficiency (0–1) | Heuristic formula (keywords, years, recency) | Trained regression model |
| **ConfidenceScorer** | Extraction results to per-field confidence | Bayesian update with method-based priors | Calibrated ML confidence model |

Every trait has production, lenient, and test implementations. Any can be
swapped without modifying the pipeline orchestration or other components
(see `infrastructure/architecture.md`).

## Detailed Design Files

| File | Covers |
|------|--------|
| `parsing.md` | Stage 1: format validation, rule-based section segmentation, rejection mechanics |
| `extraction.md` | Stage 2: pattern matching, n-gram embedding matching, implicit inference, proficiency |
| `confidence.md` | Stage 6: Bayesian per-field confidence, aggregation formula, enrichment thresholds |
| `cost-model.md` | Cross-cutting: per-CV cost breakdown, caching, startup-to-enterprise scaling |
| `evaluation.md` | Cross-cutting: metrics, Hungarian alignment, monitoring, regression testing |

## Inputs / Outputs

- **In:** raw CV document (any supported format) + optional metadata (source,
  upload date)
- **Out:** populated CandidateTensor + confidence score + list of unresolved
  skill mentions

## Dependencies

- Skill Taxonomy (`taxonomy/overview.md`) — for skill resolution
- Embedding model — shared with query compiler (`compiler/overview.md`)
- Tensor schema — defined in `core/types.md`

## Edge Cases

- **Non-English CVs:** rejected with guidance to submit in English
- **Minimal content:** very short CVs produce low-confidence tensors; the
  system proceeds but flags for enrichment
- **Non-standard formats:** scanned images are rejected with guidance to
  upload a text-based PDF or DOCX
- **Multiple CVs per candidate:** if a candidate uploads updated versions, the
  newest CV is re-processed and the tensor is overwritten; the previous
  version is retained for audit
- **Conflicting information:** when entity extraction produces contradictory
  signals (e.g. two different current employers), the most recent entry wins
  and the conflict is logged
