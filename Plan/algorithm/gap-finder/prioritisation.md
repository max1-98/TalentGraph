# Gap Finder — Prioritisation

Ranks gaps by a composite multi-criteria score that balances role importance,
information-theoretic gap magnitude, market demand, strategic graph position,
and estimation uncertainty. Produces both a scalar ranking and a Pareto
frontier for multi-objective analysis.

Depends on: `contracts.md` (GapPrioritiser trait), `proficiency-scoring.md`
(gap deltas, types, confidence), `semantic-detection.md` (adjusted deltas,
inference flags), `taxonomy/graph-model/contracts.md` (HierarchyReader for
graph distances).

---

## Composite Priority Score

  Priority(g) = W_imp * Importance(g)
              + W_info * InfoGain(g)
              + W_mkt * MarketSignal(g)
              + W_graph * GraphCentrality(g)
              - W_unc * Uncertainty(g)

Weights (W_imp, W_info, W_mkt, W_graph, W_unc) are configurable. Initial
values: (0.30, 0.25, 0.20, 0.15, 0.10). All component scores are normalised
to 0–1 before weighting.

---

## Per-Skill Information Gain (JSD Decomposition)

Measures each gap's contribution to the overall divergence between candidate
and role profiles. Uses the per-skill decomposition of Jensen-Shannon
Divergence:

  InfoGain(g) = 0.5 * (p_g * log(p_g / m_g) + q_g * log(q_g / m_g))

Where P is the normalised candidate proficiency distribution (effective
proficiency per skill, normalised to sum to 1), Q is the normalised role
importance distribution, and m_g = 0.5 * (p_g + q_g).

Skills where the role demands more than the candidate provides receive high
InfoGain — they are the skills whose absence most distorts the profile-to-role
fit. This is mathematically grounded, symmetric, and bounded (0 to log 2),
making it a stable ranking signal.

---

## Earth Mover's Distance (Global Fit Measure)

EMD captures how much "work" is needed to transform the candidate's skill
distribution into the role's requirement distribution, accounting for skill
similarity:

  EMD(P, Q) = min over all flows of
    SUM over i, j of (flow(i, j) * d_graph(skill_i, skill_j))

The ground metric d_graph is the TaxonomyGraph shortest-path distance via
HierarchyReader.shortest_path, with edge weights: parent = 1, related = 2,
supersedes = 0.5 (superseded skills are close to their successors).

EMD is computed once per gap analysis (not per gap) and maps to the
emd_distance field in the response. It serves as a holistic fit measure for
programmatic consumers (e.g. Career Pathing uses it as path difficulty).

**Efficiency:** for typical gap analyses (20–50 skills), exact transport is
feasible. For the precomputed pairwise distance matrix, Sinkhorn
regularisation provides an efficient approximation.

---

## Market Signal Weighting

Integrates real-time demand data from MarketSignalProvider:

  MarketSignal(g) = alpha_d * demand_percentile(g)
                  + alpha_g * growth_rate(g)
                  + alpha_s * salary_premium(g)

Where demand_percentile is the skill's rank in current job postings (0–1),
growth_rate is year-over-year demand change (normalised), and salary_premium
is the salary uplift associated with the skill (normalised).

Weights (alpha_d=0.5, alpha_g=0.3, alpha_s=0.2) reflect that current demand
is the strongest signal, but growth trajectory and salary impact also matter.

**Demand velocity:** growth_rate captures acceleration — a skill growing at
40% year-over-year is prioritised even if its absolute demand is modest. This
prevents the system from only recommending already-saturated skills.

---

## Graph Centrality

Captures the strategic value of closing a gap — central skills unlock more of
the taxonomy than peripheral ones:

  GraphCentrality(g) = beta_deg * normalised_degree(g)
                     + beta_bet * betweenness(g)
                     + beta_pr * pagerank(g)

Where degree counts direct connections (parent + related + supersedes edges),
betweenness measures how often the skill lies on shortest paths between other
skills, and pagerank reflects recursive importance in the graph.

All three metrics are precomputed quarterly, coinciding with the taxonomy
hierarchy recompute cycle. Stored as SkillNode metadata. Weights
(beta_deg=0.3, beta_bet=0.4, beta_pr=0.3) emphasise betweenness — skills
that bridge clusters are strategically most valuable.

---

## Uncertainty Penalty

High-uncertainty gaps are deprioritised and flagged, preventing the system
from confidently recommending action on unreliable data:

  Uncertainty(g) = 1 - (confidence(g) * market_freshness(g) * edge_confidence(g))

Where confidence(g) is from `proficiency-scoring.md`, market_freshness is a
decay factor on market signal age (1.0 for data < 30 days old, decaying
toward 0), and edge_confidence is the minimum confidence of TaxonomyGraph
edges traversed to reach the skill.

Gaps flagged as "possibly unstated" by SemanticGapDetector receive an
additional uncertainty boost (additive 0.2), further deprioritising them.

---

## Pareto Frontier

The scalar Priority score compresses multiple objectives into one number.
For strategic analysis, the system also identifies the Pareto frontier across
three dimensions: (Importance, MarketSignal, GraphCentrality).

A gap is on the frontier if no other gap dominates it across all three. These
are flagged as "strategic priorities" regardless of scalar ranking — a gap
might rank 8th but be non-dominated due to uniquely high graph centrality.
Computed via pairwise dominance comparison (feasible for 10–30 gaps).

---

## Cross-References

- Trait contract: `contracts.md` (GapPrioritiser)
- Gap deltas and confidence: `proficiency-scoring.md`
- Semantic adjustments: `semantic-detection.md`
- Graph distances and centrality: `taxonomy/graph-model/contracts.md`
- Market data: `algorithm/overview.md` (MarketSignalProvider)
- Career Pathing consumer: `career-pathing/overview.md` (step 4)
- Response schema: `integration.md`
