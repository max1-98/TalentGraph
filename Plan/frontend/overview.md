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

## Design Principles

- **Latency transparency:** show a loading indicator during query compilation
  (~800 ms cold); results appear near-instantly once cached
- **Progressive disclosure:** show composite score first, breakdown on
  interaction, full detail on expansion
- **Actionable insights:** every number shown should have a plain-language
  explanation and, where possible, a suggested action
