# CV-to-Profile — Cost Analysis

Cross-cutting concern. Justifies why the rule-based, embedding-assisted
architecture achieves near-zero marginal cost at enterprise scale, and
documents the cost profile at each growth stage. No LLMs, no pay-per-call
bottlenecks, no external API dependencies in the critical path.

Depends on: understanding of the full pipeline (`parsing.md`,
`extraction.md`, `confidence.md`).

---

## Per-CV Processing Cost Breakdown

Each stage's computational cost at steady state:

**Text extraction (PDF/DOCX parsing):** CPU-only, sub-100ms per document.
Lightweight library, no model inference. Cost: negligible.

**Section segmentation (rule-based):** string matching and Jaro-Winkler
computation. CPU-only, sub-10ms. Cost: negligible.

**Entity extraction (patterns + regex):** CPU-only, sub-50ms. Pattern
matching, list parsing, date regex — all O(n) in document length.
Cost: negligible.

**Embedding generation:** the only non-trivial cost. Each CV requires
embedding of approximately 20–50 text fragments (skills, n-grams). With the
sentence encoder (~22M parameters, likely all-MiniLM-L6-v2 — unconfirmed):
approximately 2ms per embedding on CPU, 40–100ms total per CV. On GPU:
approximately 0.1ms per embedding, 2–5ms total per CV.

**Taxonomy matching:** cosine similarity against pre-computed SkillNode
embeddings. With an ANN index (HNSW): sub-1ms per query, 20–50ms total per
CV for all fragments. Alternative: matrix multiplication against the full
pre-loaded embedding table — O(fragments * nodes * 384), feasible on CPU
for 5K–10K active nodes.

**Confidence scoring:** arithmetic operations over extraction results.
Sub-1ms. Cost: negligible.

---

## Total Per-CV Cost (steady state)

- **CPU-only:** approximately 200–300ms per CV, dominated by embedding
  generation
- **With GPU for embeddings:** approximately 50–100ms per CV
- No external API calls. No per-CV licensing fees. No LLM token costs.

---

## Cost at Each Growth Stage

### Startup (< 1K CVs/day)

Single server, CPU-only processing. Infrastructure cost dominated by
storage and API hosting, not CV parsing. Estimated parsing cost
contribution: < $5/month in compute.

### Growth (1K–50K CVs/day)

Add GPU for embedding acceleration if latency matters. A single GPU handles
50K CVs in under 2 hours batch processing. Estimated parsing cost
contribution: < $50/month (GPU cost amortised across all embedding tasks
including query compilation and taxonomy operations).

### Enterprise (50K+ CVs/day)

Horizontal scaling via stateless worker pool. Each worker processes CVs
independently with a shared, read-only SkillNode embedding table. Linear
scaling with no architectural changes. The pipeline is embarrassingly
parallel — each CV is processed in isolation.

At 50K CVs/day on a single machine: 50K * 0.3s = 4.2 hours of CPU time.
Easily handled with parallelisation across available cores.

---

## Caching Strategy

Three layers of caching reduce redundant computation:

**SkillResolver cache:** keyed by (raw text hash) to (taxonomy ID,
confidence). Expected hit rate > 80% for common skills ("Python",
"JavaScript"), dramatically reducing embedding computations for the most
frequent mentions. Cache invalidated when taxonomy changes (ChangeObserver
event from `graph-model/schema.md`).

**SkillNode embedding table:** pre-computed and held in memory. At 10K
nodes * 384 floats * 4 bytes = approximately 15 MB — trivially fits in
RAM on any server. Refreshed when taxonomy changes.

**Document deduplication:** SHA-256 hash of document bytes. If hash matches
a previously processed CV, return the cached tensor without re-processing.
Handles re-uploads and duplicate submissions. Cache keyed by (document hash,
pipeline version) to ensure re-processing when extraction logic changes.

---

## Comparison to LLM-Based Approaches

At enterprise scale (50K CVs/day):

- **LLM parsing:** 50K CVs * approximately 2K tokens/CV * $0.01/1K tokens
  = $1,000/day = approximately $30,000/month (conservative estimate with a
  mid-tier model; frontier models cost significantly more)
- **This architecture:** < $100/month in compute (dominated by server
  hosting, not per-CV marginal cost)
- **Cost advantage:** approximately 300x cheaper at enterprise scale

Additional LLM risks avoided: rate limiting, API availability dependency,
non-deterministic outputs, prompt injection via CV content, data privacy
concerns with sending CVs to external services.

---

## Cost Invariants

The following properties hold regardless of scale:

1. **Zero external API calls** in the critical parsing path. All inference
   runs locally.
2. **Marginal cost approaches zero.** Fixed costs (server, GPU) are
   amortised; per-CV variable cost is purely CPU cycles.
3. **Linear scaling.** Doubling throughput requires doubling workers — no
   superlinear cost growth, no coordination overhead.
4. **Model cost is fixed.** The sentence encoder is a one-time download
   (~90 MB). No ongoing model licensing or token costs.

---

## Cross-References

- Embedding model usage: `extraction.md` (Skills Section, Narrative)
- SkillNode embedding table: `taxonomy/graph-model/schema.md`
- Pipeline stages and latency: `overview.md`
- ChangeObserver for cache invalidation: `graph-model/schema.md`
