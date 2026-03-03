# Career Pathing — Overview

Data-driven career move suggestions: "what should I do next?" answered by
analysing the trajectories of similar professionals and current market demand.

---

## Problem

Career advice is typically intuition-based and generic. With a large corpus of
career trajectories (extracted from CandidateTensors), the system can identify
empirical patterns: what did people like you do next, and which of those moves
led to the best outcomes? Combined with market demand data, this produces
actionable, personalised suggestions.

## Approach

### Step 1 — Find Trajectory Peers

Use ANN search over career trajectory vectors (the same vectors used for f4 in
`scoring/career-trajectory.md`) to find candidates with similar career
histories: comparable seniority progression, overlapping industries, similar
skill sets at a comparable career stage.

### Step 2 — Analyse Peer Destinations

For each trajectory peer, examine where they went next: what role, industry,
seniority level, and new skills did they acquire? Aggregate these destinations
into a distribution of possible next moves.

### Step 3 — Filter by Market Demand

Cross-reference the destination distribution with MarketSignalProvider (shared
building block). Paths leading to high-demand, growing roles are boosted.
Paths toward declining demand are flagged but not hidden — the candidate
decides.

### Step 4 — Compute Gap-to-Target

For each viable destination, run Gap Finder (`gap-finder/overview.md`) to
determine what the candidate would need to acquire. This converts an abstract
"possible next move" into a concrete development plan with prioritised gaps
and bridge suggestions.

### Step 5 — Rank Paths

Rank candidate paths by a composite of: trajectory peer frequency (how common
is this move), market demand score, gap size (smaller gaps are more
accessible), and estimated time to close gaps.

## Swappable Components

| Trait | Purpose |
|-------|---------|
| **TrajectoryPeerFinder** | ANN search over career vectors to find similar histories |
| **DestinationAnalyser** | Aggregate peer next-moves into a destination distribution |
| **PathRanker** | Score and rank paths by feasibility, demand, and peer evidence |
| **TimeEstimator** | Estimate time to close gaps for each path |

## Inputs / Outputs

- **In:** CandidateTensor + optional preferences (target industry, desired
  seniority, geographic constraints)
- **Out:** ranked list of career paths, each with: destination description,
  peer evidence count, market demand score, gap list, estimated time

## Dependencies

- Trajectory vectors from `scoring/career-trajectory.md` — for peer search
- Gap Finder (`gap-finder/overview.md`) — for gap-to-target analysis
- MarketSignalProvider (`algorithm/overview.md`) — for demand filtering
- TaxonomyGraph (`taxonomy/overview.md`) — for skill adjacency in gap bridges
- ProfileComparator (`algorithm/overview.md`) — used within Gap Finder

## Detailed Design

| File | Covers |
|------|--------|
| [peer-similarity.md](peer-similarity.md) | Learned Mahalanobis distance (LMNN), career stage gating, MMR diversity, Fisher-Rao refinement |
| [time-estimation.md](time-estimation.md) | Cox Proportional Hazards, Bayesian confidence intervals, quantile regression forest ensemble, per-gap decomposition |

## Edge Cases

- **Career changers:** few trajectory peers have made a similar cross-industry
  jump; the system broadens peer search to partial overlaps and flags lower
  confidence in the suggestions
- **Very senior candidates:** the peer pool shrinks at senior levels; the
  system relaxes similarity thresholds and may surface fewer but more
  targeted paths
- **Rapidly evolving fields:** trajectory peer data may be stale (peers moved
  into roles that no longer exist in the same form); market demand filtering
  compensates by deprioritising outdated destinations
- **Insufficient data:** when the trajectory corpus is too small for
  meaningful peer analysis, the system falls back to market-demand-only
  suggestions and discloses the reduced confidence
