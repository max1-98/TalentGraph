# Role Recommendation — Overview

The inverse of recruiter search: given a candidate, find roles that match their
profile — surfacing opportunities the candidate might not have considered.

---

## Problem

Recruiter search starts with a role and finds candidates. Role recommendation
starts with a candidate and finds roles. This "inverse search" helps candidates
discover opportunities, powers job alerts, and enables proactive outreach by
matching candidates to open positions they have not applied for.

## Approach

### Step 1 — Pre-Filter Roles

Apply reversed Stage 1 logic (see `pipeline/filter.md`). Instead of filtering
candidates by role constraints, filter roles by candidate attributes: location
compatibility, salary range overlap, employment type match, seniority band.
This narrows the role corpus to a manageable set.

### Step 2 — Score with Fixed Candidate

Reuse the existing scoring pipeline (`scoring/overview.md`) with one
structural change: the candidate tensor is fixed and the query tensor varies
across roles. Each role's compiled QueryTensor (from `compiler/overview.md`)
is scored against the candidate.

The composite score formula is identical — f1 through f5 are computed per
role — but the interpretation shifts: high scores indicate roles the
candidate is well-suited for, not candidates well-suited for a role.

### Step 3 — Rank and Diversify

Sort roles by composite score. Apply light diversification to avoid surfacing
ten nearly identical positions: cluster roles by employer, industry, and
seniority, then ensure the top-N results span multiple clusters.

## Swappable Components

| Trait | Purpose |
|-------|---------|
| **RoleSourcing** | Provide the corpus of active roles to search against |
| **ScoringMode** | Configure the scoring pipeline for candidate-fixed mode |
| **CandidatePreFilter** | Reversed Stage 1: filter roles by candidate attributes |
| **ResultDiversifier** | Ensure variety across the top-N results |

## Inputs / Outputs

- **In:** CandidateTensor + optional preferences (target industry, desired
  seniority, location flexibility)
- **Out:** ranked list of matching roles, each with composite score and
  per-dimension breakdown

## Dependencies

- Scoring pipeline (`scoring/overview.md`) — reused with fixed candidate
- Query compiler (`compiler/overview.md`) — roles must have compiled tensors
- Pipeline filter logic (`pipeline/filter.md`) — reversed for role filtering

## Detailed Design

| File | Covers |
|------|--------|
| [compilation.md](compilation.md) | Pre-compilation strategy, corpus sizing, staleness management, query-time latency budget |
| [diversification.md](diversification.md) | Determinantal Point Process, cluster hard constraints, MMR fallback, evaluation metrics |

## Edge Cases

- **No matching roles:** the candidate's profile does not overlap with any
  active role — surface the closest near-misses with gap annotations
- **Hundreds of equal matches:** diversification prevents a wall of identical
  results; the system can also group results by employer or category
- **Stale postings:** roles that have been open too long or filled but not
  removed — RoleSourcing should expose a freshness signal for filtering
- **Career changers:** a candidate's current profile may not match their
  desired direction; optional preference overrides let them bias results
  toward a target industry or seniority
