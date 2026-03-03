# Feedback Loop

Outcome signals collected from recruiter and candidate interactions that feed
the learning-to-rank system and the user intelligence module.

---

## Signal Hierarchy

Signals are ordered by strength — higher-strength signals provide more
reliable training data.

| Signal | Source | Strength | Description |
|--------|--------|----------|-------------|
| Hired | Recruiter confirms placement | Strongest | This candidate got the job |
| Shortlisted | Recruiter selects for interview | Strong | Explicit positive decision |
| ProfileViewed | Recruiter clicks to see full profile | Moderate | Implicit interest |
| ConsentGiven | Candidate approves data sharing | Moderate | Candidate engaged |
| Skipped | In results but not interacted with | Weak negative | Implicitly passed |
| Rejected | Recruiter explicitly passes | Negative | Explicit rejection |

## How Signals Become Training Data

Each search that produces an outcome generates preference pairs:

1. Identify the highest-signal candidate in the result set
2. Pair them against all lower-signal candidates from the same search
3. Each pair becomes one training example for the LTR loss function

Example: in a search where candidate A was hired, candidate B was shortlisted,
and candidate C was skipped, we generate pairs:
- (A, B) — A should rank above B
- (A, C) — A should rank above C
- (B, C) — B should rank above C

## Feedback Aggregation

A background aggregator (likely a batch job — unconfirmed) processes raw
signals and:

1. Groups them by search ID
2. Extracts preference pairs
3. Writes training examples to the LTR training dataset
4. Aggregates per-candidate score breakdowns for user intelligence

## Score Breakdowns in Feedback

Every scored candidate in every search retains a full ScoreBreakdown (f1
through fN). When feedback arrives, the breakdown is linked to the outcome.
This lets the LTR system learn not just "this candidate should rank higher"
but "this candidate's skill coverage score was 0.92 and their synergy was
0.31, and they were still hired" — informing which sub-scores actually
predict success.

## Implicit vs. Explicit Signals

- **Explicit:** Hired, Shortlisted, Rejected — high confidence
- **Implicit:** ProfileViewed, Skipped — inferred from behaviour
- Implicit signals are noisier. The LTR system weights them lower
  (smaller gradient contribution) to prevent noisy signals from
  overwhelming reliable ones.

## Privacy and Consent

Feedback data is aggregated and anonymised before use in training. Individual
candidate identities are not exposed to the learning system — it operates on
tensor-level features and outcome labels only.

## Feedback Latency

There is an inherent delay between a search and its outcome (hiring decisions
take days to weeks). The system operates on a nightly or weekly batch cycle,
not real-time. This is acceptable because weight drift is gradual.

## Relationship to Market Signals

Aggregated outcome data serves a dual purpose. Beyond feeding the LTR weight
system, it also supplies the **MarketSignalProvider** building block (see
`algorithm/overview.md`) — a sibling projection of the same data that computes
skill demand frequencies, salary range distributions, and growth trends. These
market signals drive gap prioritisation, career path feasibility scoring, and
demand-based enrichment targeting.
