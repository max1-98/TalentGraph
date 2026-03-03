# Co-Founder Matching — Overview

Finds complementary team members rather than similar ones — optimising for
collective coverage across a founding team's skill and experience gaps.

---

## Problem

Standard talent search finds candidates similar to a job description. Co-founder
matching inverts this: given an existing founder (or team), find people whose
skills, experience, and network fill the team's gaps. A technical founder needs
a commercial counterpart, not another engineer.

## Approach

### Step 1 — Define Team Context

Gather the existing team's profiles (CandidateTensors) and the venture's needs.
Venture needs are expressed as a role-like description: target market, required
capabilities (technical, commercial, operational), stage, and sector.

### Step 2 — Aggregate Team Tensor

Combine the existing team's individual tensors into a single "team tensor" that
represents collective coverage. For skill dimensions, use element-wise max
(the team has a skill if any member has it). For embeddings, compute the
centroid. For career trajectory, retain the spread (min/max seniority, set of
industries).

This reuses the team-aggregation pattern from `features/team-selection.md`,
adapted for offline analysis rather than real-time search.

### Step 3 — Diff Against Venture Needs

Pass the team tensor and the venture requirements through RoleContextAnalyser
and ProfileComparator (shared building blocks). The result is a structured gap
profile: what the team collectively lacks relative to what the venture needs.

### Step 4 — Search for Complements

Convert the gap profile into a search query — essentially a job description for
the missing capabilities. Run this through the existing scoring pipeline, but
in "gap-emphasis mode": the scoring weights are shifted so that f1 (skill
coverage) and f3 (skill synergy) focus on gaps, not overall similarity.

### Step 5 — Submodular Selection

When filling multiple co-founder seats, use submodular selection
(`features/team-selection.md`) to ensure each new member covers distinct gaps
rather than duplicating each other.

## Swappable Components

| Trait | Purpose |
|-------|---------|
| **TeamTensorAggregator** | Combine individual tensors into a team tensor |
| **RoleContextAnalyser** | Venture description to structured requirements |
| **ProfileComparator** | Team tensor + requirements to gap profile |
| **ComplementarityScorer** | Score candidates by how well they fill team gaps |
| **SelectionOptimiser** | Submodular selection for multi-seat filling |

## Inputs / Outputs

- **In:** existing team members (list of CandidateTensors) + venture description
- **Out:** ranked list of complementary candidates, each annotated with which
  team gaps they address

## Dependencies

- RoleContextAnalyser and ProfileComparator (`algorithm/overview.md`)
- Team selection pattern (`features/team-selection.md`)
- Scoring pipeline (`pipeline/overview.md`) — reused in gap-emphasis mode
- TaxonomyGraph (`taxonomy/overview.md`) — for skill relationship awareness

## Detailed Design

| File | Covers |
|------|--------|
| [team-tensor.md](team-tensor.md) | Choquet integral for skills, multi-head attention pooling for embeddings, career trajectory dual representation, size normalisation |
| [gap-emphasis.md](gap-emphasis.md) | Bayesian weight posterior adjustment, Pareto-optimal weight perturbation, WeightProvider decorator pattern |

## Overlap Notes

Shares RoleContextAnalyser and ProfileComparator with Gap Finder
(`gap-finder/overview.md`). The team tensor aggregation pattern is adapted from
`features/team-selection.md`.

## Edge Cases

- **Solo founder:** the "team tensor" is just their individual tensor; the
  algorithm reduces to a specialised gap-finder search
- **Heavily overlapping team:** if all current members have similar profiles,
  gaps are large — the system should flag this as a team diversity concern
- **Non-technical ventures:** the taxonomy may have sparse coverage for
  non-technical skills (sales, fundraising, operations); enrichment from
  market signals helps fill vocabulary gaps
- **Founder with no CV:** minimal profile produces a low-confidence team
  tensor; the system recommends enrichment before matching
