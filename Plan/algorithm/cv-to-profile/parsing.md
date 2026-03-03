# CV-to-Profile — Document Validation & Parsing

Stages 1a–1c of the CV-to-Profile pipeline. Enforces a structural standard on
uploaded CVs, extracts raw text, and segments it into labelled sections. Rejects
non-compliant documents with actionable guidance rather than attempting heroic
inference.

Depends on: nothing external. Produces input for `extraction.md`.

---

## Enforced CV Standard

Rather than parsing arbitrary layouts (which demands heavy models), the system
enforces a structural contract. Non-compliant CVs are rejected with specific
guidance for the candidate to fix and re-upload.

**Requirements:**

- **Language:** English only. Detected via character-set analysis (Latin script
  ratio above 0.90) and common-word frequency (top-100 English words must
  account for a minimum share of total tokens). No ML language detection model.
- **Format:** PDF or DOCX only. All other MIME types rejected immediately.
- **Required sections:** the CV must contain identifiable headings for at least:
  Contact Information, Work Experience, Education, Skills. Missing any required
  section triggers rejection with a message naming the missing section and
  listing acceptable heading variations.
- **Maximum length:** 10 pages. Prevents abuse and bounds processing time.
- **Text extractability:** extracted character count per page must exceed a
  minimum threshold (e.g. 50 characters). Below this, the document is likely a
  scanned image — rejected with guidance to upload a text-based document.

---

## Format-Specific Text Extraction

**PDF:** lightweight text extraction library (likely pdfminer or similar —
unconfirmed). Extracts text blocks with position coordinates and font metadata
(size, weight, name). No layout model — raw extraction only.

**DOCX:** XML structure parser (likely python-docx or similar — unconfirmed).
Extracts paragraphs with style metadata (heading level, bold, font size). DOCX
structure is inherently ordered — no layout reconstruction needed.

Output: ordered list of TextBlock (text content, page number, font metadata).

---

## Section Segmentation

Rule-based, no ML. Four steps:

**Step 1 — Heading detection:** score each line as a potential heading:
heading_score = w1 * font_size_ratio + w2 * is_bold + w3 * is_uppercase
+ w4 * is_short_line + w5 * dictionary_match. The dictionary contains ~50
common English CV section headings and variations ("Work Experience",
"Employment History", "Professional Experience", etc.).

**Step 2 — Section labelling:** match detected headings to canonical section
types via Jaro-Winkler distance (lowercased, stripped). Closest match above
threshold (0.85) wins. Below threshold: labelled "other".

**Step 3 — Validation:** verify all required sections were found. If not,
generate a rejection response listing missing sections with examples of
acceptable headings.

**Step 4 — Segmentation:** text between consecutive headings is assigned to
the preceding heading's section. Output: ordered list of Section (canonical
label, raw text, heading text, page number, heading confidence score).

---

## Rejection Response Contract

Rejection is not an error — it is a structured response. Contains: list of
specific violations, each carrying a type (wrong_format, missing_section,
wrong_language, scanned_image, too_long), a human-readable explanation, and a
suggested fix. Overall verdict: "accepted" or "rejected". The API returns this
as a normal CVUploadResponse with extracted_profile = null and requires_review
populated with the violation list.

---

## Trait Contracts

Three narrow traits (ISP-compliant split of the former DocumentParser):

**DocumentValidator** — Input: raw document bytes + MIME type. Output:
ValidationVerdict (pass, or fail with list of Violation). Each Violation
carries: type, human-readable message, suggested fix. Invariant: validation is
pure — no side effects, no state mutation.

**TextExtractor** — Input: raw document bytes + MIME type (already validated).
Output: ordered list of TextBlock (text content, page number, font metadata).
For DOCX: heading level and style name replace position/font data. Invariant:
every character of extractable text appears in exactly one TextBlock.

**SectionSegmenter** — Input: list of TextBlock. Output: ParsedDocument
containing ordered list of Section (canonical label, raw text, heading text,
page, heading_confidence), total page count, character count. Invariant: every
TextBlock is assigned to exactly one Section.

---

## Implementations

| Trait | Production | Lenient | Test |
|-------|-----------|---------|------|
| DocumentValidator | Full standard enforcement | Accept everything (bypass) | Always-pass stub |
| TextExtractor | PDF + DOCX with metadata | Plain text fallback (no metadata) | Returns hardcoded text |
| SectionSegmenter | Heading scoring + Jaro-Winkler | Assign all text to "unknown" | Returns single section |

Any implementation is a drop-in replacement — the pipeline does not know or
care which is active (see `infrastructure/architecture.md`).

---

## Cross-References

- Entity extraction from parsed sections: `extraction.md`
- Confidence scoring of extraction results: `confidence.md`
- Pipeline overview and stage map: `overview.md`
- SOLID trait architecture: `infrastructure/architecture.md`
