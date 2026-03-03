# Frontend — Overview

> **Stack note:** likely TypeScript — unconfirmed. This document describes
> the UI structure and interaction design, independent of framework choice.

---

## Role

The frontend provides two primary interfaces: a recruiter search experience
and a candidate intelligence dashboard.

## Recruiter Interface

### Search Input

- Free-text job description input (primary)
- Optional structured filters: location + radius, salary range, notice
  period, employment type, seniority level
- "Search" action triggers query compilation and pipeline execution

### Results Display

- Ranked candidate list with composite scores
- Per-candidate score breakdown visualisation (radar chart or bar chart
  showing f1 through fN normalised values)
- Expandable detail view: matched skills, relevant experience chunks,
  career trajectory summary
- Shortlist actions: add to shortlist, request consent, reject

### Score Explainability

Each score dimension is labelled in plain language:
- "Skill match: 92 %" rather than "f1: 0.92"
- "Experience relevance: 71 %" rather than "f2: 0.71"
- Hover or expand for a brief explanation of what each dimension measures

### Multi-Vacancy Mode

When multiple vacancies are specified, results are grouped by seat.
Each seat shows its own shortlist, with a team-level diversity summary.

## Candidate Interface

### Intelligence Dashboard

Displays the UserIntelligence report (see `features/user-intelligence.md`):
- Match activity count
- Average score breakdown across all matching searches (visualised)
- Market demand: top skills requested in matching roles
- Highest-impact skill gaps with actionable suggestions
- Synergy gaps: skills the candidate has but hasn't combined
- Career trajectory insight

### Profile Management

- View and edit structured profile data
- Upload CV for automatic extraction
- Manage consent settings (opt in/out of search visibility)

## Data Flow

```
Recruiter → Search UI → API (POST /search) → Results UI
Candidate → Dashboard → API (GET /intelligence) → Insights UI
```

## Data Shapes

All request and response types (SearchRequest, SearchResult,
ScoreBreakdownDisplay, UserIntelligenceReport, etc.) are defined in
`core/schema/api-contracts-core.md` and `core/schema/api-contracts-algorithms.md`.
The frontend never manually defines API shapes — typed interfaces are generated
from the shared schema. See `core/schema/overview.md` for the full design.

## Design Principles

- **Latency transparency:** show a loading indicator during query compilation
  (~800 ms cold); results appear near-instantly once cached
- **Progressive disclosure:** show composite score first, breakdown on
  interaction, full detail on expansion
- **Actionable insights:** every number shown should have a plain-language
  explanation and, where possible, a suggested action
- **Engagement-aware design:** the platform has a daily habit loop (feed) and
  a weekly match loop; the UI hierarchy reflects this — feed is the default
  landing view, matches are a distinct premium surface
- **Unified roles:** the same user can act as candidate and hiring manager
  within one session; no hard account-type fork

## Frontend File Index

| File | Description |
|------|-------------|
| [overview.md](overview.md) | UI structure, data flow, design principles |
| [layout.md](layout.md) | Nav structure, design aesthetic, mobile-first layout, micro-interactions |
| [candidate-experience.md](candidate-experience.md) | Feed, match inbox, gap finder, career path, profile, insights |
| [hiring-experience.md](hiring-experience.md) | Post role, applicant view, multi-role, team-fill |
| [profile-builder.md](profile-builder.md) | CV import, progressive completion, AI suggestions, privacy controls |
