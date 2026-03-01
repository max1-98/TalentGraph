# User Intelligence

Aggregated score breakdowns transformed into actionable career insights for
candidates.

---

## Purpose

Every time a candidate appears in a search result, their ScoreBreakdown
(f1 through fN) is recorded. Over time, these breakdowns accumulate into a
per-candidate intelligence profile that reveals patterns invisible from a
single search.

## What Users Learn

### Match Activity

"You were considered for 47 roles this quarter."

The count of searches where the candidate appeared in the scored result set.

### Average Score Breakdown

Aggregated across all matching searches:

| Dimension | Example Insight |
|-----------|-----------------|
| Skill coverage (f1) | "82 % — your skills match most roles well" |
| Semantic relevance (f2) | "71 % — your experience is broadly relevant" |
| Skill synergy (f3) | "44 % — this is your weakest area" |
| Career trajectory (f4) | "68 % — solid career progression signal" |
| Practical fit (f5) | "90 % — logistics are rarely an issue" |

### Market Demand

The skills most frequently required in roles that matched the candidate,
sorted by frequency. Shows the candidate what the market wants from someone
with their profile.

### Highest-Impact Gaps

Skills the candidate is missing that would have the greatest impact on their
scores. Calculated as: frequency_in_matching_roles * average_weight_in_scoring.

"Adding Kubernetes experience would improve your match rate for 34 % of
the roles you're being considered for."

### Synergy Gaps

Specific to f3 — identifies skill pairs the candidate possesses independently
but has never used together in the same project.

"You know Python and you know Kafka, but you've never used them together.
That's why you're scoring low on fintech data pipeline roles. Consider a
project combining them."

This insight is unique to this system — only possible because skill
co-occurrence is tracked at the chunk level and scored explicitly in f3.

### Trajectory Insight

Derived from the career trajectory vector and the seniority Gaussian:

"Your career trajectory suggests you're ready for Staff level, but most
roles matching you are Senior level."

## Data Flow

1. Every search produces ScoreBreakdowns for all scored candidates
2. A background aggregator (nightly batch) groups breakdowns by candidate
3. Computes running averages, demand frequencies, and gap analysis
4. Writes the resulting UserIntelligence record to the database

## Privacy

- Candidates see only their own aggregate data
- No individual search or recruiter is identifiable from the aggregate
- Insights are computed from anonymised score data, not from raw queries

## Relationship to Feedback

User intelligence and the feedback loop (see `learning/feedback.md`) consume
the same raw data — ScoreBreakdowns. The feedback system uses them to train
weights; user intelligence uses them to inform candidates. The aggregation
runs in the same batch pipeline.
