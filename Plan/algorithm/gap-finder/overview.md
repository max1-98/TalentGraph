# Gap Finder — Overview

Given a candidate profile and a target role (or company), identifies actionable
skill and experience gaps, prioritised by impact and market demand.

---

## Problem

Candidates and career coaches need to know: "What am I missing for this role?"
A simple list of absent skills is not enough — gaps must be prioritised by how
much they affect match quality and how valuable they are in the market. Ideally,
the system also suggests how to bridge each gap.

## Approach

### Step 1 — Analyse Role Context

Pass the target role description (or company profile) through
RoleContextAnalyser (shared building block). This produces a structured
requirements object: required skills with importance weights, target seniority
range, industry context, team composition needs.

### Step 2 — Compare Profile to Requirements

Feed the candidate's CandidateTensor and the structured requirements into
ProfileComparator (shared building block). The comparator produces a structured
diff:

- **Missing skills** — required by the role, absent from the profile
- **Weak skills** — present but below the expected proficiency level
- **Surplus skills** — candidate has them, role does not require them
- **Experience deltas** — seniority gap, industry mismatch magnitude

### Step 3 — Prioritise Gaps

Rank each gap by: importance weight (from role requirements) multiplied by
market demand signal (from MarketSignalProvider). High-importance, high-demand
gaps surface first. Low-importance or declining-demand gaps are deprioritised.

### Step 4 — Suggest Bridges

For each top-priority gap, use TaxonomyGraph adjacency to suggest bridge
paths: related skills the candidate already has that could serve as stepping
stones, certifications that cover the gap, or adjacent skills that are
quicker to acquire.

## Swappable Components

| Trait | Purpose |
|-------|---------|
| **RoleContextAnalyser** | Role description to structured requirements |
| **ProfileComparator** | Profile + requirements to structured diff |
| **GapPrioritiser** | Rank gaps by impact and market demand |
| **BridgeSuggester** | Propose paths to close each gap via taxonomy adjacency |

## Inputs / Outputs

- **In:** CandidateTensor + target role description (or structured requirements)
- **Out:** prioritised list of gaps, each with: skill ID, gap type (missing /
  weak / experience), priority score, and suggested bridges

## Dependencies

- RoleContextAnalyser and ProfileComparator (`algorithm/overview.md`)
- TaxonomyGraph (`taxonomy/overview.md`) — for bridge suggestions
- MarketSignalProvider (`algorithm/overview.md`) — for demand-based prioritisation
- Scoring sub-functions (`scoring/`) — ProfileComparator generalises f1–f5

## Detailed Design

- [contracts.md](contracts.md) — trait interfaces: ProficiencyGapScorer,
  SemanticGapDetector, GapPrioritiser, BridgeSuggester
- [proficiency-scoring.md](proficiency-scoring.md) — decay-aware continuous
  deltas, context thresholds, gap type classification
- [semantic-detection.md](semantic-detection.md) — soft matching, latent skill
  inference, synergy gaps, cluster-level analysis
- [prioritisation.md](prioritisation.md) — JSD, EMD, market signals, graph
  centrality, Pareto frontier
- [bridge-suggestions.md](bridge-suggestions.md) — graph traversal,
  transferability scoring, collaborative filtering, learning path ordering
- [integration.md](integration.md) — building block wiring, Career Pathing
  callsite, User Intelligence feed, schema extensions

## Overlap Notes

Shares RoleContextAnalyser and ProfileComparator with Co-Founder Matching
(`cofounder-matching/overview.md`) and Career Pathing
(`career-pathing/overview.md`). These are shared building blocks, not
duplicated logic.

## Edge Cases

- **Broad generalist roles:** many skills at low importance — gaps are shallow
  but numerous; the prioritiser should surface the few that matter most
- **Thin profiles:** low-confidence tensors produce unreliable diffs; the
  system flags this and recommends enrichment before gap analysis
- **Multiple target roles:** when a candidate targets several roles, run gap
  analysis per role and optionally merge into a unified development plan
- **Exact match:** no gaps found — the system confirms fit and highlights
  surplus skills as potential differentiators
