# Capacity and Database Performance — Load Testing and Query Work

> Sanitized: no proprietary code, credentials or client data.

## Scope

Multi-client FTTH platform, one reference table of a few hundred thousand rows and forty columns. Open question from the business with no measured answer: how many users can work concurrently, and what machine is required. The production machine had four vCPU, not sized against any measurement.

Browser-side work on the same grid: [Data Grid Performance](measured-performance-diagnosis.md).

---

## 1. Instrument defect: scenarios silently discarded

**Symptom.** Initial load runs reported healthy figures. The output named a scenario `default`, which the script does not define. The script's own scenarios mix real gestures — list, filter, edit, export, open history — in production proportions.

**Root cause.** `K6_VUS`, `K6_DURATION` and `K6_ITERATIONS` are k6's own options, including when passed as `-e K6_VUS=300`, which is the syntax for passing a variable to the script. k6 claims them, discards every scenario the script declares and runs a flat default instead. The gesture mix was never executed.

The first diagnosis attributed this to a stray environment variable on the machine rather than the flag on the command line, and was corrected.

**Fix.**

- Scripts read `LOAD_VUS` / `LOAD_DURATION`, which k6 does not claim.
- The script asserts that the executing scenario is the expected one and fails with an error naming the likely cause.

---

## 2. Saturation point

Throughput in requests per second is the reported unit; virtual users are the applied pressure, not a result.

| Load | Result |
|---|---|
| 150 concurrent | All thresholds green except export p95 (539 ms) |
| 300 concurrent | All six thresholds crossed, 187 req/s |

The knee sits between the two: past it, throughput flattens while latency climbs.

All gestures degraded together. A single slow endpoint indicates one query; uniform degradation indicates a shared resource. Two candidates: the connection pool or the CPU.

---

## 3. Isolating pool from CPU

**Method.** Change the pool size, hold everything else constant, re-measure.

**Result.** The pool contribution was real and small. The CPU dominated. Four vCPU was the ceiling and pool tuning does not move it.

**Action.** Machine sized on cores rather than connection settings.

---

## 4. Query costs

**Save path: 730 ms → 113 ms.** Per-row work executing on every save. Removed rather than parallelized.

**History histogram: 477,000 rows read per request.** The query recomputed, on every request, figures that could no longer change. `EXPLAIN (ANALYZE, BUFFERS)` over the 30-day window:

```
Parallel Index Scan using idx_..._timestamp
  rows=159009 loops=3        <- 477,027 rows read
Workers Launched: 2
Execution Time: 283 ms
```

283 ms in isolation is survivable; the plan shows why it is not. Each request recruited three of the four vCPU. At ten concurrent users the median measured 934 ms. Parallelism that benefits a single user degrades concurrent load.

The table is append-only and stamped with the current time, so 29 of the 30 days in the chart are immutable. The query now reads a daily rollup of a few dozen rows.

Two implementation decisions:

- **Scheduled job, not a trigger.** Maintaining the rollup from the audit trigger would charge every save for it, and the save had just been reduced from 730 ms by removing per-save work. The write path has no knowledge of the rollup.
- **Recompute two full days; no `id` watermark.** A sequence does not guarantee commit order — the transaction holding id 100 can commit before the one holding 99, and a high-water mark over `id` would skip that row permanently. Recomputation by day is idempotent; late arrivals land in the next pass.

**Also applied:** keyset pagination instead of `OFFSET`, and trigram GIN indexes for the text searches that were performing sequential scans.

---

## 5. Regression guard on query plans

A dropped index is invisible until production is slow. The query plans are asserted as tests: they run against a real PostGIS container and assert via `EXPLAIN` that each critical query still uses the index it was designed around. Removing an index fails the pull request.

The mechanism is per-table rather than specific to the first table, because the second table required it before it was written.

Construction of those tests, and the trap in the obvious implementation: [CI Pipeline](ci-pipeline-what-blocks.md#4-query-plan-assertions).

---

## 6. Rollup not built on the second table

**Proposal.** Copy the rollup mechanism to the second table.

**Measurement first.**

```
Index Only Scan
  rows=232          <- 11,232 rows in the whole table
Planning Time:  25.137 ms
Execution Time:  0.560 ms
```

232 rows, against 477,000 on the first table. Planning costs 45× more than execution, at which point the query is not the constraint — 0.560 ms is the entire budget.

**Decision.** Not built. A precomputed summary would add a table, a scheduled job and a minute of staleness for no measurable gain. The measurement and the row-volume threshold that would reverse the decision are recorded in the code.

---

## 7. Defect introduced during this work

Adding the client IP to the audit context: `current_setting('x', true)` returns an empty string, not NULL, once it has been set in a session. On pooled connections the `COALESCE` chain then stored `''` permanently. Caught by a test written for the change, before release.

Generalization: on a pooled connection, session state is not fresh state.

## Tools

k6 (custom scenarios, thresholds as contracts, `Trend`/`Rate`/`Counter`) · PostgreSQL `EXPLAIN (ANALYZE, BUFFERS)` · Testcontainers + Flyway for plan-guard tests · JUnit tag selection to keep them off the fast build · GitHub Actions · ADRs
