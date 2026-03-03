# Ingestion Pipeline — Overview (Discovery)

How new skills enter the taxonomy. The taxonomy grows automatically from
documents flowing through the system — no manual curation required.

Depends on: `graph-model/contracts.md` (SkillProvider, GraphMutator).

---

## Trigger

The pipeline runs whenever a new CV is parsed or a new JD is compiled. When
the SkillResolver (see `graph-model/contracts.md`) encounters a mention it
cannot confidently map to an existing active node, it emits a "candidate"
signal containing the raw text and its context window. This signal is the
input to ingestion.

---

## Three Stages

### Stage 1: Text Extraction (SkillExtractor trait)

Identifies skill-like mentions in free text. Three complementary methods:

1. **Named Entity Recognition:** a domain-tuned model (likely a fine-tuned
   transformer — unconfirmed) trained to recognise skill-like spans in CVs
   and JDs. Catches novel skills that pattern matching would miss.
2. **Pattern matching:** lexical templates that reliably surround skills —
   "N years of X", "proficient in X", "experience with X", "certified in X".
   High precision for well-structured documents.
3. **Taxonomy-aware checking:** for each candidate span, query SkillProvider
   by name and by embedding similarity. If a close match exists (cosine
   similarity > 0.92), map to it directly rather than creating a candidate.

When multiple mentions of the same skill appear in one document, they are
collapsed to the single richest context window — the one with the most
surrounding text for embedding quality.

**SkillExtractor trait contract:**
- Input: document text (string), document type (CV | JD)
- Output: list of ExtractionResult (span text, context window, confidence,
  extraction method)
- Invariant: no duplicate spans per document; context windows do not exceed
  a configurable token limit

### Stage 2: Embedding Generation

Each extracted mention is embedded using the same model as CandidateTensor
embeddings (likely all-MiniLM-L6-v2 — unconfirmed). The embedding is computed
from the skill name concatenated with its surrounding context, not the name
alone.

This **context-conditioning** is critical for disambiguation: "Rust" next to
"memory safety" and "cargo" produces a very different vector from "rust" next
to "corrosion" and "oxidation". Same surface form, different meanings,
different embeddings.

The context window is truncated to the model's token limit (typically 512
tokens). If multiple context windows exist for the same mention, the longest
(richest) one is used.

### Stage 3: Candidate Classification

The mention's embedding is compared against all existing active SkillNodes
by cosine similarity (via SkillProvider's search_by_embedding). The maximum
similarity score determines the classification:

| Similarity Range | Classification | Action |
|-----------------|---------------|--------|
| max sim > 0.92 | **Known skill** | Map to existing node; no further action |
| 0.80 < sim <= 0.92 | **Potential alias** | Forward to deduplication pipeline |
| sim <= 0.80 | **Potential new skill** | Forward to admission stage |

These thresholds are calibrated on a labelled synonym dataset (precision
target: > 95% for auto-match, > 90% for alias detection). They are
configurable per deployment to accommodate domain-specific vocabularies.

---

## SkillExtractor — Implementations

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | NER + patterns + taxonomy check, full pipeline | Normal operation |
| PatternOnly | Lexical patterns only, no ML model | Fallback when NER model unavailable |
| Passthrough | Accepts all input spans as skills | Testing and benchmarking |

All three are drop-in replacements (LSP compliance). The production variant
is the default; fallback selection is configuration-driven.

---

## Data Flow Summary

```
Document (CV/JD)
  |
  v
SkillExtractor --- (known skill) ---> SkillProvider.get_by_name
  |
  v
Embedding Generation
  |
  v
Candidate Classification
  |--- sim > 0.92 ---> Map to existing node (done)
  |--- 0.80 < sim <= 0.92 ---> Deduplication pipeline
  |--- sim <= 0.80 ---> Admission stage (admission.md)
```

---

## Cross-References

- Admission gating for new candidates: `ingestion/admission.md`
- Deduplication for alias detection: `deduplication/overview.md`
- SkillResolver and SkillProvider contracts: `graph-model/contracts.md`
- Embedding model and CandidateTensor: `core/types.md`
