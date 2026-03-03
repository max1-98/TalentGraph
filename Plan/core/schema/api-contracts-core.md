# API Contracts — Core Types

Field-level prose definitions for Tier 2 types in the profile, search, and
feedback domains. These are the data shapes shared between the API and
frontend. For algorithm result types, see `api-contracts-algorithms.md`.

> See `boundary-types.md` for the full inventory and tier classification.

---

## ProfilePayload

The display-safe representation of a candidate profile. Used for profile
viewing, editing, and search result detail expansion.

- **user_id** — unique identifier for the candidate
- **display_name** — full name (or pseudonym if consent restricts it)
- **headline** — short professional summary (one line)
- **location** — city and country (not raw coordinates)
- **open_to_work** — whether the candidate is actively seeking
- **experience** — ordered list of positions, each with: title, company name,
  start date, end date (or "present"), and a prose description
- **skills** — list of skills, each with: display name, proficiency level
  (beginner / intermediate / advanced / expert), and years of experience
- **education** — list of entries, each with: institution, qualification, field
  of study, and graduation year
- **preferences** — employment types accepted, salary expectations (range),
  maximum notice period, location flexibility (remote / hybrid / on-site),
  preferred industries
- **consent_status** — whether the candidate has opted in to search visibility
- **profile_completeness** — 0–100 score indicating how filled-out the profile
  is (used to prompt candidates to add missing information)

Derived from CandidateTensor identity and skill fields (`core/types.md`), but
omits all internal numerical representations (embeddings, career vector,
filter-pack fields). The API maps between ProfilePayload and the database
model; the engine maps between ProfilePayload and CandidateTensor where needed.

## CVUploadRequest

Submitted when a candidate uploads a document for automatic extraction.

- **document** — the uploaded file (PDF, DOCX, or plain text)
- **extraction_preferences** — optional hints: target language, whether to
  overwrite existing profile fields or merge
- **user_id** — the candidate uploading the document

## CVUploadResponse

Returned after CV processing completes.

- **extracted_profile** — a partial ProfilePayload containing the fields
  successfully extracted from the document
- **confidence** — 0–1 overall extraction confidence
- **warnings** — list of extraction issues (unrecognised sections, ambiguous
  dates, skills not in taxonomy)
- **requires_review** — list of fields that need manual confirmation

---

## SearchRequest

Submitted by a recruiter to initiate a talent search.

- **job_description** — free-text JD (primary input; compiled into a
  QueryTensor by the engine)
- **filters** — optional structured constraints:
  - location + maximum radius
  - salary budget range
  - maximum notice period in days
  - acceptable employment types (permanent, contract, freelance, part-time)
  - minimum seniority level
  - required industries
- **result_limit** — maximum number of candidates to return (default 50)
- **vacancies** — number of positions to fill (triggers multi-vacancy
  optimisation when greater than one; see `features/team-selection.md`)
- **search_id** — client-generated idempotency key (optional)

## SearchResult

Returned for a completed search.

- **search_id** — identifier linking results to the original request
- **candidates** — ordered list of matched candidates, each containing:
  - **candidate_id** — unique identifier
  - **display_name** and **headline** — for the results list
  - **composite_score** — 0–100 overall match score
  - **score_breakdown** — a ScoreBreakdownDisplay (see below)
  - **matched_skills** — skills the candidate has that the role requires
  - **experience_highlights** — relevant experience excerpts
- **total_evaluated** — how many candidates entered the pipeline
- **query_compilation_cached** — whether the JD compilation was a cache hit
- **diagnostics_summary** — coarse latency and funnel counts (detailed
  diagnostics are Tier 3; see `engine-transport.md`)

## ScoreBreakdownDisplay

The external projection of the internal ScoreBreakdown (`core/types.md`).
Raw scorer floats are mapped to named, human-readable dimensions.

- **dimensions** — ordered list, each containing:
  - **label** — plain-language name (e.g. "Skill match", "Experience relevance")
  - **score** — integer 0–100 (mapped from the sigmoid-normalised internal float)
  - **explanation** — brief prose describing what this dimension measures
- **composite** — the weighted total, also 0–100

The mapping from internal scorer IDs (f1, f2, ...) to display labels is
maintained by the API layer. When new scorers are registered, corresponding
display metadata must be added.

---

## FeedbackSignal

Submitted by a recruiter to record an outcome for a candidate in a search.

- **search_id** — the search this feedback relates to
- **candidate_id** — the candidate the feedback is about
- **signal_type** — one of: hired, shortlisted, profile_viewed, consent_given,
  skipped, rejected (see signal hierarchy in `learning/feedback.md`)
- **timestamp** — when the signal was recorded
- **notes** — optional free-text context from the recruiter
