# Deployment

Rollout strategy, rollback mechanisms, and infrastructure cost projections.

---

## Canary Rollout

Changes to scoring logic, weights, or pipeline configuration follow a
staged rollout:

1. **Shadow mode** (1 week) — new configuration runs alongside production,
   results logged but never served
2. **Canary** (5 % traffic) — a small fraction of live searches use the new
   configuration. Monitor for anomalies in ranking quality, latency, and
   error rates.
3. **Gradual ramp** — increase traffic to 25 %, 50 %, 100 % over days,
   gated on metrics
4. **Full rollout** — previous version remains registered for rollback

## Instant Rollback

Both the old and new versions of any component (scorer, weight set, stage
configuration) are registered simultaneously. Rollback is a configuration
change — switch the "enabled" list or weight path back to the previous
version. No recompilation, no restart required.

Rollback time: approximately 30 seconds (config file change + hot reload).

## Versioning

- Each scorer carries a version string that changes when its logic changes
- The scorer registry computes a composite version (hash of all active
  scorer versions) used for cache invalidation
- Weight sets are versioned and stored as separate files
- Pipeline configurations are version-controlled in the repo

## Infrastructure Cost Projections

| Component | 100K users | 1M users | 10M users |
|-----------|-----------|----------|-----------|
| Matching engine (RAM-bound) | 1x small instance (~$150/mo) | 1x medium instance (~$500/mo) | 3x large instances (~$4 500/mo) |
| Database (likely Postgres) | Small instance (~$200/mo) | Medium instance (~$600/mo) | Large + read replicas (~$2 000/mo) |
| Cache (likely Redis) | Small instance (~$100/mo) | Same (~$100/mo) | Medium instance (~$200/mo) |
| LLM costs (query compilation) | ~$50/mo | ~$200/mo | ~$500/mo |
| **Total** | **~$500/mo** | **~$1 300/mo** | **~$7 200/mo** |

For context: a single recruiter placement at typical agency rates covers
months of infrastructure cost.

## Scaling Strategy

- **Vertical first:** the matching engine is designed to run on a single
  machine with all tensors in memory. At 1M candidates (~4 GB tensors +
  ~1.5 GB indexes), this fits on a mid-range instance.
- **Horizontal at 10M+:** shard the tensor store and indexes across multiple
  engine replicas, each holding a subset of candidates. The API layer
  fans out queries and merges results.
- **Read replicas for the database:** search is read-heavy; profile writes
  are infrequent. Read replicas absorb search-adjacent queries.

## Monitoring

- Per-stage latency percentiles (p50, p95, p99) for every search
- Funnel analysis: candidate counts at each stage (detect filter drift)
- Scorer version tracking: which versions are active across instances
- Alert on: latency spikes, empty-result rate increases, NDCG drops below
  threshold
- Dashboard for shadow mode comparisons: primary vs. shadow ranking deltas

## Disaster Recovery

The matching engine's tensor store and indexes are derived data — they can
be fully rebuilt from the database. Recovery procedure:

1. Restore the database from backup
2. Run a full tensor recompilation (~35 minutes for 1M candidates on 16
   cores)
3. Rebuild vector indexes from the recompiled tensors
4. Resume serving

The matching engine is stateless beyond its in-memory data. Multiple
instances can be spun up behind a load balancer.
