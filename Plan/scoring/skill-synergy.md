# f3 — Skill Synergy Score

Measures whether the candidate has used the required skills *together* in the
same projects, not merely listed them independently.

---

## Formula

f3(c, q) = cos(c_skill_vec, q_skill_vec) + gamma * PMI(c, q)

**First term:** cosine similarity between the full skill vectors of the
candidate and the query. Captures overall skill profile alignment.

**Second term:** average Pointwise Mutual Information of skill co-occurrence
within the candidate's work chunks.

PMI(c, q) = (1 / |pairs|) * SUM_{(a,b) in query_skill_pairs}(
  max(0, log( P(a,b) / (P(a) * P(b)) ))
)

- P(a, b) = fraction of candidate's chunks containing both skill a and skill b
- P(a) = fraction of candidate's chunks containing skill a
- gamma = learnable scaling factor

## Why This Matters

"Python + Finance together in the same project" is categorically different from
"knows Python from one job and worked in finance at a different job." Every
other matching system treats skills as independent dimensions. The co-occurrence
signal captures the combinatorial value of multi-skill experience.

## Inputs

- **Candidate skill vector** — for the cosine term
- **Candidate skill co-occurrence matrix** — a compact upper-triangular sparse
  matrix (~200 bytes for typical profiles), pre-computed during tensor
  compilation. Entry (a, b) stores how many chunks contain both skill a and
  skill b.
- **Query required skills** — used to select which pairs to evaluate

## Algorithm

1. Compute cosine similarity between candidate and query skill vectors
2. Enumerate all pairs of query skills: |pairs| = n * (n - 1) / 2
3. For each pair, look up co-occurrence count in the candidate's matrix
4. If co-occurrence > 0, compute PMI and clamp to non-negative
5. Average the PMI values across all pairs
6. Combine: cosine_term + gamma * average_pmi

## Edge Cases

- **Candidate has all skills but never together:** cosine term may be high,
  but PMI term will be low — the score correctly reflects the gap
- **Query has only one skill:** no pairs to evaluate, PMI term is zero, score
  reduces to pure cosine similarity
- **Candidate with very few chunks:** PMI can be noisy with small sample
  sizes; the confidence field in the tensor can modulate weight accordingly
- **PMI clamping:** negative PMI (skills that avoid each other) is clamped
  to zero — we only reward positive co-occurrence, not penalise negative

## Synergy Feedback

This sub-score uniquely enables actionable career advice: "You know Python and
you know Kafka, but you've never used them together — that's why you're scoring
low on fintech data pipeline roles." Only possible because co-occurrence is
tracked at the chunk level and scored explicitly.
