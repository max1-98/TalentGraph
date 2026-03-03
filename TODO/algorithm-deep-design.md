# Algorithm Deep Design

Each algorithm skeleton needs thinking through — fleshing out the structure,
clarifying interfaces, identifying open questions. One task per algorithm.

---

## Foundation

- [x] **Taxonomy: define the graph data model.** What are the node and edge
  types? What attributes does each node carry (ID, canonical name, embedding,
  usage count, status)? What edge types exist beyond parent/alias/related/
  supersedes? How large can the graph grow before query performance degrades?
  *Done: `algorithm/taxonomy/graph-model/schema.md` and `contracts.md`.*

- [x] **Taxonomy: detail the three automated pipelines.** Ingestion,
  deduplication, and hierarchy inference are sketched but not specified.
  Define: trigger conditions, batch vs streaming, thresholds for promotion
  and merge, rollback mechanics.
  *Done: `algorithm/taxonomy/ingestion/`, `deduplication/`, `hierarchy/`.*

## Profile Construction

- [x] **CV-to-Profile: specify the parsing stage.** Which document formats are
  supported? What is the section-labelling strategy (heuristic, ML-based,
  hybrid)? How is OCR integrated for scanned documents? Define the
  DocumentParser trait contract.
  *Done: `algorithm/cv-to-profile/parsing.md`, `extraction.md`,
  `cost-model.md`, `evaluation.md`, `taxonomy/bootstrap.md`.*

- [x] **CV-to-Profile: define the confidence model.** The skeleton mentions a
  0-to-1 confidence score but not how it is computed. Define: which fields
  contribute, how missing vs inferred vs explicit data are weighted, what
  thresholds trigger enrichment.
  *Done: `algorithm/cv-to-profile/confidence.md`.*

- [x] **Profile Enrichment: define the merge strategy.** When external signals
  conflict with CV data, what wins? Define: conflict resolution rules,
  whether candidates are notified of enrichment, how the enrichment log is
  structured, privacy opt-out mechanics.
  *Done: `algorithm/profile-enrichment/merge-strategy.md` (conflict hierarchy,
  field-level rules, confidence model) and `privacy.md` (opt-out, enrichment
  log, notification, retention, GDPR).*

## Analysis Algorithms

- [x] **Gap Finder: define gap prioritisation formula.** The skeleton says
  "importance weight * market demand" but this needs fleshing out. Define:
  how importance weights are normalised, how market demand is quantified,
  whether recency or growth rate matters, how bridge suggestions are ranked.
  *Done: `algorithm/gap-finder/proficiency-scoring.md`, `semantic-detection.md`,
  `prioritisation.md`, `contracts.md`, `integration.md`.*

- [x] **Gap Finder: specify bridge suggestion mechanics.** TaxonomyGraph
  adjacency is mentioned but not detailed. Define: how many hops are
  considered, how bridge paths are scored, whether certifications are a
  special node type in the graph.
  *Done: `algorithm/gap-finder/bridge-suggestions.md`, `contracts.md`. Node type
  enum added to `taxonomy/graph-model/schema.md`.*

## Recommendation Algorithms

- [x] **Career Pathing: define trajectory peer similarity.** What distance
  metric is used over career vectors? How many peers are retrieved? How is
  "comparable career stage" determined — years of experience, seniority
  level, or something else?
  *Done: `algorithm/career-pathing/peer-similarity.md`.*

- [x] **Career Pathing: specify the time estimator.** The skeleton lists a
  TimeEstimator trait but gives no indication of how time-to-close-gap is
  computed. Define: data sources, methodology (peer median, regression,
  heuristic), confidence bounds.
  *Done: `algorithm/career-pathing/time-estimation.md`.*

- [x] **Role Recommendation: address compilation cost.** The algorithm scores
  a candidate against many roles, each needing a compiled QueryTensor. Are
  role tensors pre-compiled and cached? What is the expected corpus size?
  How is staleness handled for compiled role tensors?
  *Done: `algorithm/role-recommendation/compilation.md`.*

- [x] **Role Recommendation: define diversification strategy.** "Cluster by
  employer, industry, seniority" is mentioned but not specified. Define:
  clustering method, how many clusters, minimum representation per cluster
  in top-N results.
  *Done: `algorithm/role-recommendation/diversification.md`.*

## Backlog

Low-priority items deferred until higher-priority algorithms are stable.

- [x] **Co-Founder Matching: define team tensor aggregation.** Element-wise max
  for skills, centroid for embeddings — but what about career trajectory and
  practical fit dimensions? Define the full aggregation strategy and how it
  handles teams of varying size.
  *Done: `algorithm/cofounder-matching/team-tensor.md`.*

- [x] **Co-Founder Matching: specify "gap-emphasis mode" scoring.** The skeleton
  says scoring weights are shifted to emphasise f1 and f3 for gaps. Define:
  how weights are adjusted, whether this is a separate weight set or a
  runtime modifier, how it interacts with per-vertical weights.
  *Done: `algorithm/cofounder-matching/gap-emphasis.md`.*
