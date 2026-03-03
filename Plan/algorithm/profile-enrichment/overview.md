# Profile Enrichment — Overview

Augments incomplete CandidateTensors with signals from external sources,
filling gaps that CVs leave behind.

---

## Problem

CVs are self-reported and incomplete. Candidates omit skills they consider
obvious, understate experience, or simply have outdated documents. Public
signals — open-source contributions, publications, professional profiles,
certifications — can fill these gaps and improve match quality.

## Approach

### Step 1 — Discover External Profiles

Given a candidate's identity fields (name, email, known handles), locate
matching profiles on external platforms. Matching is probabilistic — name
alone is insufficient, so multiple signals (email, employer overlap, skill
overlap) are combined to score match confidence.

### Step 2 — Extract Signals

For each confirmed external profile, extract structured signals: additional
skills, project descriptions, endorsements, publication topics, contribution
patterns. Each signal carries a source tag and a freshness timestamp.

### Step 3 — Resolve to Taxonomy

Pass extracted skill mentions through SkillResolver, just as in the
CV-to-Profile pipeline. Newly discovered skills that are not yet in the
taxonomy become ingestion candidates.

### Step 4 — Merge into CandidateTensor

Combine external signals with the existing tensor. The merge strategy must
handle conflicts (e.g. CV says "intermediate Python", GitHub activity suggests
"advanced Python") and avoid overwriting explicit candidate-provided data
without justification.

### Step 5 — Recompute Embeddings

After merging, regenerate the four embeddings to reflect the enriched profile.
The confidence score is updated — enriched profiles typically move from low to
moderate confidence.

### Step 6 — Prioritise by Demand

Use MarketSignalProvider (shared building block) to prioritise enrichment
effort. Skills in high demand but missing from the profile receive more
enrichment attention than niche or declining skills.

## Detailed Design Files

| File | Covers |
|------|--------|
| [merge-strategy.md](merge-strategy.md) | Conflict resolution hierarchy, field-level merge rules, confidence updating |
| [privacy.md](privacy.md) | Opt-out schema, enrichment log, notification policy, data retention, GDPR |

## Swappable Components

| Trait | Purpose | Day-1 Impl | Upgrade Path |
|-------|---------|------------|--------------|
| **SourceDiscoverer** | Candidate identity to list of candidate external profiles | Email + name matching against known platforms | Graph-based identity resolution |
| **SignalExtractor** | Per-source extraction of structured signals | Platform-specific scrapers with rate limiting | Unified extraction model |
| **MergeStrategy** | Rules for combining external signals with existing tensor | Additive (fill gaps only) | Full production merge with conflict detection |
| **SkillResolver** | Free-text skills to canonical taxonomy IDs | Taxonomy lookup + alias table | Embedding-based fuzzy resolution |

## Inputs / Outputs

- **In:** existing CandidateTensor + candidate identity fields
- **Out:** enriched CandidateTensor + enrichment log (what changed, from which
  source, at what confidence)

## Dependencies

- Skill Taxonomy (`taxonomy/overview.md`) — for skill resolution
- MarketSignalProvider (`algorithm/overview.md`) — for demand-based prioritisation
- CV-to-Profile (`cv-to-profile/overview.md`) — provides the base tensor
- Embedding model — shared across the system

## Edge Cases

- **Name collisions:** "John Smith" matches thousands of profiles; discovery
  must require multiple corroborating signals before confirming a match
- **Source unavailable:** external APIs may be rate-limited or down; enrichment
  is best-effort and never blocks tensor creation
- **Conflicting signals:** when external data contradicts the CV, the conflict
  is logged and the higher-confidence signal wins, but the candidate's
  explicit data is never silently overwritten
- **Privacy:** enrichment only uses publicly available data; candidates can
  opt out of enrichment entirely; all sources are disclosed in the
  enrichment log
