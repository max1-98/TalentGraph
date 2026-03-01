# Learning to Rank

A pairwise learning-to-rank framework that trains scoring weights from
real hiring outcomes.

---

## Objective

For each search with a known outcome, we know that the hired candidate should
rank above all rejected ones. The system learns weight parameters that make
this ordering as correct as possible across the entire dataset.

## Loss Function (LambdaRank)

L = - SUM_{(i,j) in P} log( sigma( Score(c_i, q) - Score(c_j, q) ) )

- P = set of preference pairs: candidate i should rank above candidate j
- sigma = sigmoid function
- This is the standard pairwise LambdaRank loss, which optimises NDCG of
  the ranking

## Preference Pairs

Pairs are extracted from outcome data using a strict ordering:

Hired > Shortlisted > ProfileViewed > ConsentGiven > Skipped > Rejected

Each adjacent pair becomes a training example:
- (hired_candidate, shortlisted_candidate) — hired should rank higher
- (shortlisted_candidate, skipped_candidate) — shortlisted should rank higher
- And so on through the hierarchy

## What Gets Learned

| Parameter | Description |
|-----------|-------------|
| w_1 through w_N | Per-scorer weights (how much each sub-score matters) |
| beta_1 through beta_4 | Embedding perspective scaling factors |
| lambda | Recency decay rate in the skill vector |
| sigma | Seniority Gaussian width |
| Penalty curves | Location decay, notice sigmoid steepness |
| Per-vertical specialisation | Separate weight sets by industry x seniority |

## Training Process

- Runs as a batch job (nightly or weekly — unconfirmed)
- Gradient descent on all learnable parameters
- Requires ~500 searches with outcomes before weights diverge meaningfully
  from hand-tuned priors
- Falls back to priors when insufficient data for a specific vertical

## Per-Vertical Weights

Different job types weight signals differently. A fintech search may weight
industry match (f4) higher than a startup search. Per-vertical weight
specialisation begins once enough outcome data accumulates for each
industry-by-seniority bucket.

When a vertical has insufficient data, it inherits the global weight set
with Bayesian regularisation to prevent overfitting.

## Guardrails

- High regularisation during early training prevents wild swings
- Minimum data thresholds per vertical before specialisation activates
- A/B testing infrastructure (see `infrastructure/testing.md`) validates
  new weights against incumbent weights before full rollout
- Learned weights are versioned; instant rollback via config change

## Relationship to Scoring

The LTR system outputs weight sets consumed by the WeightProvider trait.
The scoring pipeline itself is unaware of how weights were produced — it
simply applies them. This separation means the learning subsystem can
evolve independently of the scoring logic.
