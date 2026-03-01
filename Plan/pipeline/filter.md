# Stage 1 — Bitwise Pre-Filter

Eliminates impossible matches using packed integer comparisons. No floating
point, no branching. Runs on the entire candidate universe.

---

## Purpose

Reduce the candidate set from ~1M to ~50K by checking hard constraints that
can be evaluated with integer and bitwise operations. This is the cheapest
stage — it runs in approximately 1 ms for one million candidates.

## Input / Output

- **In:** full candidate universe (all FilterPacks, contiguous in memory)
- **Out:** list of candidate IDs that passed all checks (~50K typical)

## The FilterPack Structure

Each candidate's filterable fields are packed into a 32-byte struct aligned
to one cache line for SIMD processing:

- Quantised latitude and longitude (integer, ~1 km precision)
- Salary minimum and maximum (in thousands)
- Notice period in days
- Bitflags: open-to-work, employment types, remote-ok
- Years of experience (fixed-point, 0.1-year precision)
- Skill bloom filter (64-bit)
- Industry bloom filter (32-bit)

Total: exactly 32 bytes. 1M candidates = 32 MB of contiguous, cache-friendly
memory.

## Checks Performed

1. **Required flags** — bitwise AND against query flags (employment type,
   open-to-work). Fails if any required flag is missing.
2. **Notice period** — integer comparison against query maximum
3. **Salary floor** — candidate minimum must not exceed query budget maximum
4. **Experience floor** — candidate years must meet query minimum
5. **Skill bloom** — 64-bit bloom filter AND against query skill bloom. If
   result is zero (candidate has none of the required skills), eliminated.
6. **Location bounding box** — quantised lat/lng difference against query
   radius. Cheaper than Haversine; a rectangular approximation.

All checks are simple integer comparisons — SIMD-friendly, branch-free where
possible.

## The Skill Bloom Filter

A 64-bit bloom filter built from the candidate's skill IDs. Checks "does this
candidate have ANY of the required skills?" in a single bitwise AND.

- False positives are acceptable (caught in Stage 3)
- False negatives are impossible (the bloom is built with OR semantics)
- At 64 bits with ~50 skills per person, false positive rate is approximately
  2 % — a cheap tradeoff for a massive speedup

## Relationship to f5 (Practical Fit)

Stage 1 and f5 evaluate the same constraints (location, salary, notice,
employment type) but Stage 1 uses wider bounds and integer approximations.
A candidate 10 km outside the radius passes Stage 1 but receives a low f5
score. This two-tier approach preserves both speed and precision.

## Scaling

At 10M candidates, Stage 1 takes ~8 ms. The bottleneck is memory bandwidth —
scanning 320 MB of FilterPacks. SIMD intrinsics (likely AVX2/AVX-512) process
eight candidates per CPU cycle.
