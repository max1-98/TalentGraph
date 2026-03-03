# API Contracts — Algorithm Result Types

Field-level prose definitions for Tier 2 types produced by higher-level
algorithms and returned to the frontend. For core types (profile, search,
feedback), see `api-contracts-core.md`.

> See `boundary-types.md` for the full inventory and tier classification.

---

## UserIntelligenceReport

Aggregated career insights for a candidate, computed from historical score
breakdowns across all searches where the candidate was evaluated. Described
in detail in `features/user-intelligence.md`.

- **candidate_id** — the candidate this report belongs to
- **period** — date range covered by the aggregation
- **match_count** — number of searches that included this candidate
- **average_breakdown** — a ScoreBreakdownDisplay (see `api-contracts-core.md`)
  averaged across all matching searches
- **market_demand** — list of skills most frequently required in matching roles,
  each with: skill name and frequency count
- **highest_impact_gaps** — skills the candidate lacks that would most improve
  match rates, each with: skill name, estimated score improvement, and
  percentage of matching roles requiring it
- **synergy_gaps** — skill pairs the candidate possesses independently but has
  not demonstrated together, each with: skill pair and the role categories
  where the combination is valued
- **trajectory_insight** — prose summary of career trajectory relative to
  matching roles (e.g. seniority band alignment)

## TaxonomyNode

The external representation of a single skill in the ontology. Used by admin
CRUD interfaces. See `algorithm/taxonomy/overview.md` for the taxonomy design.

- **node_id** — unique identifier
- **canonical_name** — the primary display name
- **aliases** — alternative names that resolve to this node
- **parent_id** — the parent node in the hierarchy (null for root nodes)
- **relationship_type** — edge type to parent: parent, alias, related,
  supersedes
- **status** — active, candidate (below frequency threshold), or archived
- **created_at** — when the node was first added
- **sighting_count** — number of independent mentions across CVs and JDs

---

## GapAnalysisResult

Output of the Gap Finder algorithm (`algorithm/gap-finder/overview.md`).
Prioritised skill and experience gaps between a candidate and a target role.

- **candidate_id** — the candidate analysed
- **target_description** — summary of the target role or requirements
- **gaps** — prioritised list, each containing:
  - **skill_name** — the skill or experience area
  - **gap_type** — missing (not present), weak (below expected proficiency),
    or experience (seniority/industry mismatch)
  - **priority_score** — composite of role importance and market demand
  - **bridges** — suggested paths to close the gap (adjacent skills the
    candidate has, certifications, quick-to-acquire alternatives)
- **surplus_skills** — skills the candidate has beyond role requirements
  (potential differentiators)
- **overall_fit** — 0–100 summary score of profile-to-role alignment

## CareerPathSuggestion

Output of the Career Pathing algorithm (`algorithm/career-pathing/overview.md`).
A ranked next-move option with supporting evidence.

- **destination** — description of the suggested role or career move (title,
  industry, seniority)
- **peer_evidence_count** — how many trajectory peers made this move
- **market_demand_score** — 0–100 reflecting current demand for this
  destination
- **gap_summary** — condensed GapAnalysisResult for the candidate relative to
  this destination
- **estimated_time** — rough estimate to close the identified gaps
- **confidence** — low / medium / high, reflecting peer pool size and data
  recency

## RoleRecommendation

Output of the Role Recommendation algorithm
(`algorithm/role-recommendation/overview.md`). A matching role surfaced for a
candidate.

- **role_id** — identifier of the matching role
- **role_title** — display title
- **employer** — company or organisation name
- **composite_score** — 0–100 overall match
- **score_breakdown** — a ScoreBreakdownDisplay showing per-dimension fit
- **matched_skills** — skills the candidate has that the role requires
- **gap_skills** — skills the role requires that the candidate lacks

## CofounderMatchResult

Output of the Co-Founder Matching algorithm
(`algorithm/cofounder-matching/overview.md`). A complementary candidate
annotated with which team gaps they address.

- **candidate_id** — the suggested co-founder
- **display_name** and **headline** — for the results list
- **complementarity_score** — 0–100 reflecting how well they fill team gaps
- **gaps_addressed** — list of team gaps this candidate covers (skill names
  and experience areas)
- **overlap_areas** — areas where this candidate duplicates existing team
  strengths (flagged to aid decision-making)

## TeamAssignment

Output of multi-vacancy search (`features/team-selection.md`). Maps each
vacancy to a shortlist optimised for team-level coverage.

- **vacancy_index** — which vacancy (seat) this assignment is for
- **shortlist** — ordered list of candidates for this seat, each with:
  - **candidate_id**, **display_name**, **headline**
  - **composite_score** — 0–100 match for this vacancy
  - **marginal_contribution** — what this candidate uniquely adds to the team
- **team_coverage** — percentage of required skills covered by the full team
  assignment
