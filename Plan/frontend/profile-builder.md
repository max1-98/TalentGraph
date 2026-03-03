# Frontend — Profile Builder

> Describes the profile creation and enrichment experience for candidates.
> Covers CV import, progressive completion, AI-assisted suggestions, and
> privacy controls.

---

## Design Goal

A candidate should have a useful, match-ready profile within 3 minutes of
arriving on the platform. The profile builder is the primary path to that
outcome.

---

## Entry Paths

1. **CV upload** — fastest path; AI parses the document and populates a draft
2. **LinkedIn import** — convenience path via OAuth (likely — unconfirmed)
3. **Manual entry** — fallback; the builder presents the profile in micro-steps

All three paths converge at the same profile structure. The difference is
how quickly that structure is pre-populated.

---

## CV Import Flow

### Step 1 — Upload
- Drag-and-drop or file picker; accepts PDF and common document formats
- Framing: "We'll parse your CV and build a draft profile in seconds"
- Progress indicator during parsing (likely under 5 seconds)

### Step 2 — Parse Preview
- The extracted fields are shown for confirmation:
  - Current and past roles (titles, employers, date ranges)
  - Skills extracted (with confidence labels — "detected" vs "inferred")
  - Education
  - Summary (if detected)
- Candidate can accept all, edit individual fields, or remove false extractions
- Unrecognised or ambiguous skills are flagged: "We weren't sure about these
  — confirm or remove"

### Step 3 — Confirm and Save
- One-click confirmation saves the draft as the active profile
- Profile completeness ring updates immediately to reflect the parsed data
- Candidate lands on the Gap Finder or Match Preview (see onboarding flow)

---

## Profile Sections

Each section is a distinct unit with its own completeness contribution:

| Section | Completeness weight | Notes |
|---|---|---|
| Current role | High | Required for meaningful matches |
| Skills | High | Primary match signal |
| Experience history | Medium | Influences career trajectory score |
| Education | Low | Parsed from CV; rarely entered manually |
| Career summary | Low | Auto-generated; editable |
| Preferences | Medium | Location, salary, employment type — affects practical fit score |

---

## Progress Ring

A circular completeness indicator is always visible on the Profile view.

- Fills proportionally as sections are completed
- Each section has a distinct arc segment (colour-coded by weight)
- Animates on update — positive reinforcement
- Tooltip on each segment: "Adding skills improves your match quality by ~X%"
  (estimated, based on weight in the scoring formula)

---

## AI-Assisted Suggestions

The profile builder surfaces contextual improvement prompts:

- "Based on your experience at [Company X], consider adding [Skill Y] — it
  appears frequently in roles similar to yours"
- "Adding a seniority level for your skills could improve your match quality
  by ~15%"
- "Your profile is missing a career summary — this helps the algorithm
  contextualise your experience"

Suggestions are non-blocking: candidates can dismiss or defer them. They
surface as inline prompts within the relevant section, not as modal
interruptions.

---

## Privacy Controls

Privacy settings are **inline within each section**, not buried in a settings
page.

Per-section visibility options:
- **Public** — visible on the public profile
- **Employers only** — visible only to verified employers during match review
- **Private** — used by the algorithm; never shown to any external party

Sensible defaults: skills and current role are "employers only" by default;
contact information is "private" by default. Candidates can change any setting
at any time.

---

## Skill Input

The skills section uses taxonomy autocomplete:
- As the candidate types, the taxonomy graph (see `algorithm/taxonomy/`)
  suggests matching canonical skill names
- Confidence levels can be set per skill (e.g., beginner / intermediate /
  proficient / expert)
- Related skills are suggested: "People with [Skill X] often also list [Skill Y]"
- Skill recency is tracked (last used / currently using)
