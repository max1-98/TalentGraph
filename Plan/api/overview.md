# Backend API — Overview

> **Stack note:** likely Django REST + Postgres — unconfirmed. This document
> describes the design at the interface level, independent of framework choice.

---

## Role

The API layer sits between clients (web app, external consumers) and the
matching engine. It handles authentication, request validation, profile CRUD,
and search orchestration.

## Core Endpoints (sketch)

### Profile Management

- **Create / update profile** — accepts structured profile data (positions,
  chunks, skills). Writes to the database. Triggers async tensor
  recompilation via message queue.
- **Upload CV** — accepts a document, extracts structured data (positions,
  chunks), and creates/updates the profile. Extraction likely uses an
  LLM or parsing service (unconfirmed).
- **Get profile** — returns the candidate's profile data. Does not expose
  the raw tensor.

### Search

- **Submit search** — accepts a job description (free text) and optional
  constraints (location, salary, etc.). Forwards to the matching engine
  (likely via gRPC — unconfirmed). Returns ranked results with score
  breakdowns.
- **Get search results** — retrieves previously computed results by search ID.

### Feedback

- **Record outcome** — accepts outcome signals (hired, shortlisted, rejected,
  etc.) linked to a search and candidate. Writes to the feedback store.
- **Get user intelligence** — returns the aggregated intelligence report for
  the authenticated candidate.

### Administration

- **Taxonomy management** — CRUD for the skill taxonomy
- **Weight management** — view active weights, trigger retraining
- **Health / metrics** — system status, latency percentiles, pipeline
  diagnostics

## Data Flow

```
Client → API Gateway → [Auth + Validation]
  ├── Profile writes → Database → Message Queue → Tensor Compiler
  ├── Search queries → Matching Engine (gRPC) → Ranked Results
  └── Feedback signals → Database → Nightly Aggregation
```

## Authentication and Authorisation

- Recruiter and candidate roles with different access levels
- Candidates can only access their own profiles and intelligence reports
- Recruiters can search and record outcomes within their organisation
- Specific mechanism (JWT, session, OAuth) is unconfirmed

## Consent

Candidate data is only included in search results if the candidate has
granted consent. The API enforces this at the query level — the matching
engine never sees non-consented profiles.

## Rate Limiting

Search queries are rate-limited per organisation to prevent abuse and
control LLM compilation costs. Profile updates are rate-limited per user.
