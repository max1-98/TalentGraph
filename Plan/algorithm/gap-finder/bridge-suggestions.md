# Gap Finder — Bridge Suggestions

Proposes actionable bridge paths for closing each prioritised gap. Combines
graph traversal, embedding similarity, collaborative filtering from skill
acquisition history, and learning path ordering.

Depends on: `contracts.md` (BridgeSuggester trait), `prioritisation.md`
(prioritised gaps), `taxonomy/graph-model/contracts.md` (SkillProvider,
HierarchyReader), `learning/cold-start.md` (cold-start pattern).

---

## Graph Traversal

For each gap skill, identify candidate skills that could serve as stepping
stones:

**Step 1 — Seed selection:** find the top-10 candidate skills nearest to the
gap skill by TaxonomyGraph distance (via HierarchyReader.shortest_path).
These are the candidate's most relevant existing strengths.

**Step 2 — Hop-limited BFS:** from the gap skill, perform breadth-first
search up to H = 3 hops (configurable). Traversal follows these edge types
in both directions: parent (up and down), related, and supersedes (old to
new). Alias edges are traversed transparently per the TaxonomyGraph contract.

**Step 3 — Intersection:** the bridge set is the intersection of BFS-reachable
skills and skills the candidate either has or is 1-hop from. Each bridge path
records the sequence of skills and edge types traversed.

The 3-hop limit balances reach against relevance — at 4+ hops, connections
become too tenuous to be actionable.

---

## Transferability Scoring

Each bridge path is scored by how effectively the candidate can transfer
existing knowledge to close the gap:

  Transferability(path) = exp(-alpha * path_length)
                        * cosine(embedding(source), embedding(target))
                        * min_edge_confidence(path)
                        * proficiency(source_skill)

**Components:**

- **Distance decay:** exp(-alpha * path_length) penalises longer paths. Alpha
  is configurable (initial value 0.5), giving a 1-hop path approximately 1.6x
  the score of a 2-hop path.
- **Embedding similarity:** cosine similarity between the source skill (what
  the candidate has) and the target skill (the gap). Captures semantic
  relatedness beyond graph structure.
- **Edge confidence:** the minimum confidence among all edges in the path.
  A path through a low-confidence edge is less reliable as a bridge.
- **Source proficiency:** stronger existing skills make better stepping stones.
  Uses EffectiveProficiency from `proficiency-scoring.md` (decay-adjusted).

The final score is in the range 0–1.

---

## Collaborative Filtering

When skill acquisition history is available (which candidates acquired which
skills over time), collaborative filtering adds an empirical signal: "people
with your skills successfully acquired this target skill."

  AcquisitionProbability(target | candidate_skills) =
    count(acquired target AND had relevant_skills)
    / count(had relevant_skills)

Conditioned on the top-5 candidate skills most relevant to the gap (by
graph distance). This captures real-world learning paths — if many React
developers successfully learned Vue, that bridge is empirically validated.

**Graceful degradation:** when acquisition history is unavailable or sparse,
the system falls back to GraphOnly mode (graph + embedding scoring only).
This follows the cold-start pattern from `learning/cold-start.md` — start
with structural signals, layer in collaborative data as it accumulates.

The acquisition probability is combined with the transferability score as a
weighted sum (configurable blend, initial 0.7 transferability + 0.3
acquisition probability when history is available).

---

## Learning Path Ordering

When multiple gaps exist, the order in which a candidate addresses them
matters. Some skills are prerequisites for others.

**Dependency detection:** if closing gap A produces a skill that appears in
the bridge path for gap B, then A should be addressed before B. This creates
a partial ordering among gaps.

**Topological sort:** gaps are arranged in dependency order. Where no
dependency exists, higher-priority gaps (from `prioritisation.md`) come first.

**Output:** the ordered list is included in the response as
recommended_learning_order — a sequence of gap IDs from "start here" to
"address last."

---

## Certification Handling

Certifications are a distinct mechanism for closing gaps — they provide
structured, verifiable evidence of skill acquisition.

**Recommendation:** extend SkillNode with a node_type field (enum: skill,
certification, course) rather than encoding certifications as edge metadata.
This allows certifications to participate fully in graph traversal — a
certification node connects to the skills it covers via parent edges, and
the BFS naturally discovers it as a bridge path.

See `taxonomy/graph-model/schema.md` for the node_type addition.

**Bridge ranking bonus:** certification and course nodes receive a
configurable bonus (initial value 1.2x multiplier) in bridge scoring,
reflecting that structured learning is often more efficient than self-directed
skill transfer. The bonus is applied after the transferability score.

---

## Output Format

Each gap receives up to 3 bridge suggestions, sorted by descending
transferability score. Each suggestion includes:

- Source skill (what the candidate already has)
- Path through the taxonomy (sequence of skills and edge types)
- Transferability score (0–1)
- Acquisition probability (0–1, when history is available)
- Certification recommendation (if a certification node appears in the path)
- Estimated effort level (derived from path length and delta magnitude)

---

## Cross-References

- Trait contract and variants: `contracts.md` (BridgeSuggester)
- Prioritised gaps input: `prioritisation.md`
- Graph traversal API: `taxonomy/graph-model/contracts.md` (HierarchyReader)
- Proficiency for source skills: `proficiency-scoring.md`
- Cold-start degradation pattern: `learning/cold-start.md`
- Certification node type: `taxonomy/graph-model/schema.md`
- Response schema: `integration.md`
