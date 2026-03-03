# CV-to-Profile — Entity Extraction

Stage 2 of the CV-to-Profile pipeline. Extracts structured entities from
validated, segmented text using pattern matching, statistical methods, and
cheap embedding similarity. No NER models, no LLMs. The enforced CV standard
(`parsing.md`) means each section type has known, constrained structure —
extraction rules are specialised per section.

Depends on: `parsing.md` (ParsedDocument, Section), `taxonomy/overview.md`
(SkillResolver), `taxonomy/bootstrap.md` (ESCO occupation-skill mappings).

---

## Contact Section (pure regex)

Email, phone, location, and URL extracted via standard regex patterns. Name:
first non-empty line lacking email/phone patterns (heuristic).

---

## Skills Section (list parsing + embedding matching)

**Step 1 — List parsing:** split by common delimiters (commas, semicolons,
newlines, bullets, pipes). Each fragment is a skill candidate.

**Step 2 — Cleanup:** strip whitespace, remove empties and noise phrases
("and", "etc"). Reject fragments < 2 or > 100 characters.

**Step 3 — Taxonomy matching:** embed each fragment using the sentence encoder
(likely all-MiniLM-L6-v2, ~22M params — unconfirmed). Cosine similarity
against pre-computed SkillNode embeddings via SkillResolver.

**Step 4 — BM25 fallback:** for fragments below embedding threshold (cosine
< 0.80), BM25 against SkillNode names and aliases. Catches abbreviations
embeddings miss ("JS" for "JavaScript").

Output per skill: SkillMention (raw text, resolved taxonomy ID or null,
match method, confidence = similarity score, origin = "explicit").

---

## Experience Section (positional patterns + regex)

Entries follow a positional template: line 1 = job title (bold/larger font),
line 2 = company + location, line 3 = date range (regex), remaining =
description. Entry boundaries at date-range matches or heading-level lines.

Dates normalised to (year, month, precision) tuples. "Present"/"Current"
map to upload date. Year-only has precision = "year".

---

## Education Section (pattern matching)

Degree detection via regex for known abbreviations and full names ("BSc",
"PhD", "MBA", etc.). Institution: non-degree line within each entry,
optionally validated against a known-institutions list. Dates: same regex
as experience. Field of study: text following degree or "in" keyword.

---

## Narrative Skill Extraction (from experience descriptions)

Skills also appear in experience descriptions. Extracted via n-gram taxonomy
matching — no NER. Steps: (1) extract unigrams, bigrams, trigrams from each
description. (2) TF-IDF filter: keep n-grams above a weight threshold,
eliminating common words without a stopword list. (3) Embed surviving n-grams
and compute cosine similarity against SkillNode embeddings; threshold 0.85
(stricter than Skills section to reduce narrative false positives).
(4) Deduplicate: merge with Skills-section results, keeping higher confidence
when the same taxonomy node is matched from both sources.

---

## Implicit Skill Inference

Three sources, all flagged origin = "inferred" (candidates can override):

**Source 1 — ESCO occupation-skill:** job title matched to ESCO occupation
via embedding similarity. Occupation's essential skills inferred with
confidence = occupation_match_confidence * importance_rating.

**Source 2 — Dependency rules:** curated set (React→JavaScript,
Django→Python). Fixed confidence 0.7–0.9. Seeded from ESCO, refined via
corpus co-occurrence.

**Source 3 — Corpus co-occurrence:** P(B|A) from corpus. If > 0.8, infer B
with confidence = P(B|A) * 0.7 (discounted).

---

## Proficiency Estimation

Four signals, weighted and normalised to [0, 1]:

- **Explicit keywords:** "expert"=1.0, "advanced"=0.8, "intermediate"=0.6,
  "basic"=0.4, "beginner"=0.2
- **Years:** proficiency_signal = sigmoid((N - 2) / 3)
- **Recency:** exp(-lambda * years_since_last_use), lambda = ln(2) / 3
- **Seniority:** Junior=0.3, Mid=0.5, Senior=0.7, Lead/Principal=0.9

Aggregate: P = w1*explicit + w2*years + w3*recency + w4*seniority. Feeds
the P factor in S = alpha * P * exp(-lambda * t) * log(1 + d) * w_ctx
(`core/types.md`).

---

## SkillMatcher Trait Contract

**Input:** raw skill text (string) + SkillProvider (read-only taxonomy access).

**Output:** SkillMatch (taxonomy node ID or null, confidence score, match
method: embedding | bm25 | exact).

**Invariant:** confidence is the raw similarity score — no rescaling. A null
ID means no match above threshold. Does not mutate the taxonomy.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Embedding cosine + BM25 fallback | Normal operation |
| ExactOnly | String-match against canonical names/aliases | Degraded / no-model mode |
| Test | Returns hardcoded SkillMatch | Unit testing |

---

## EntityExtractor Trait Contract

**Input:** ParsedDocument (from SectionSegmenter — validated).

**Output:** ExtractionResult containing: list of SkillMention (raw text,
taxonomy ID, confidence, origin, match method), ExperienceEntry list,
EducationEntry list, ContactInfo, CertificationEntry list.

**Invariant:** every extracted entity traces to a Section and character
offsets within it.

| Variant | Description | Use Case |
|---------|-------------|----------|
| Production | Full pattern + embedding pipeline | Normal operation |
| Lenient | Regex-only, no embedding matching | Fallback / degraded mode |
| Test | Returns hardcoded ExtractionResult | Unit testing |

---

## Cross-References

- Section input: `parsing.md` | Confidence scoring: `confidence.md`
- SkillResolver: `taxonomy/overview.md` | ESCO mappings: `taxonomy/bootstrap.md`
- Skill score formula (P factor): `core/types.md`
