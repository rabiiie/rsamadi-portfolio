# Capacity and Database Performance — Sizing a Machine With a Measurement Instead of a Guess

Companion to [Measured Performance Diagnosis](measured-performance-diagnosis.md). That one is the browser half: why a grid felt slow with the server out of the picture. This is the other half — what happens when fifty people use it at once, and what the database was actually doing under them.

## Context

A multi-client FTTH platform, one reference table of a few hundred thousand rows and forty columns, and a question from the business that had no measured answer: *how many people can use this at the same time, and what machine does it need?*

The honest answer at the time was that nobody knew. The machine had four vCPU because that is what it was given.

## The instrument lied first

The first load runs looked wonderful and meant nothing. The script defines named scenarios that mix real gestures — list, filter, edit, export, open history — in realistic proportions. The output kept reporting a scenario called `default`, which the script does not define.

The cause: `K6_VUS`, `K6_DURATION` and `K6_ITERATIONS` are **k6's own options**, including when passed as `-e K6_VUS=300`. That looks exactly like passing a variable to your script. It isn't. k6 claims them, discards every scenario the script declares, and silently runs a flat default one instead.

So the mix was never running. A measurement of the wrong thing is worse than no measurement, because it comes with a number attached.

Two changes came out of it. The scripts read `LOAD_VUS` / `LOAD_DURATION`, which k6 has no opinion about. And the script now **fails loudly** if the expected scenario is not the one executing, naming the likely cause in the error text — so the next person loses a minute instead of an afternoon.

I also got the explanation wrong the first time, blaming a stray environment variable on the machine rather than the flag being typed on the command line. The guard exists because of that, not despite it.

## Finding the knee

Throughput is the unit that means something — requests per second — not the number of virtual users, which is just the pressure you apply.

| Load | Result |
|---|---|
| 150 concurrent | All thresholds green except export p95 (539 ms) |
| 300 concurrent | **All six thresholds crossed**, 187 req/s |

The knee sits between the two. Past it, throughput flattens while latency climbs — the platform stops going faster and starts going slower, which is the definition of saturation.

Every gesture degraded together. That pattern matters: one slow endpoint means one bad query, but *everything* degrading in step means a shared resource. Two candidates — the connection pool or the CPU.

## Separating the pool from the CPU

The temptation is to enlarge the pool, because it is a one-line change. I ran the controlled version instead: change the pool size, change nothing else, measure again.

The pool contributed a little and was real. The CPU dominated. Four vCPU was the ceiling, and no amount of pool tuning was going to move it.

That is a boring conclusion and it is the useful kind: it told the business to buy cores rather than pay me to tune connection settings that were not the constraint.

## What the database was doing

With the ceiling understood, the per-query costs were worth attacking.

**The save path: 730 ms → 113 ms.** The cost was per-row work happening on every save. Removing it, not parallelizing it, was the fix.

**The history histogram: 477,000 rows per request.** It counted from scratch, on every request, figures that could no longer change. `EXPLAIN (ANALYZE, BUFFERS)` over the 30-day window:

```
Parallel Index Scan using idx_..._timestamp
  rows=159009 loops=3        <- 477,027 rows read
Workers Launched: 2
Execution Time: 283 ms
```

283 ms alone looks survivable. It isn't, and the plan says why: each request recruited **three of the four vCPU**. At ten concurrent users the median measured 934 ms. Parallelism that helps one user is exactly what sinks ten.

The table is append-only and always stamped with the current time, so of thirty days in the chart, twenty-nine are immutable. It now reads a daily rollup of a few dozen rows.

Two decisions inside that are worth stating, because both had a tempting wrong answer:

- **A scheduled job, not a trigger.** Maintaining the rollup from the audit trigger would charge every save for it — and the save had just been brought down from 730 ms precisely by taking work off it. The write path does not know the rollup exists.
- **Recompute two whole days, no `id` watermark.** A sequence does not guarantee commit order: the transaction holding id 100 can commit before the one holding 99, and a high-water mark over `id` would skip that row permanently. Recomputing by day is idempotent — anything arriving late lands in the next pass.

**Also:** keyset pagination instead of `OFFSET`, and trigram GIN indexes for the text searches that were scanning.

## Making it stay fixed

An index deleted by accident is invisible until production is slow. So the query plans became tests: they run against a real PostGIS container and assert via `EXPLAIN` that each critical query still uses the index it was designed around. Delete one and the pull request goes red.

The mechanism is per-table, not specific to the first one. That was deliberate: the second table needed it before it was written.

How those tests are built — and the trap in the obvious way to write them — is in the pipeline write-up: [CI With Security Built In](ci-pipeline-what-blocks.md#once-there-was-a-real-database-in-ci-it-could-guard-more-than-boot).

## The measurement that said don't build it

The obvious move on the second table was to copy the rollup across. I measured first:

```
Index Only Scan
  rows=232          <- 11,232 rows in the whole table
Planning Time:  25.137 ms
Execution Time:  0.560 ms
```

232 rows, not 477,000. And **planning the query costs forty-five times more than executing it** — at which point the query is not the problem, because half a millisecond is the entire budget. A precomputed summary could not win anything, and would have added a table, a scheduled job and a minute of staleness in exchange.

So it wasn't built. The measurement and the volume threshold that would reverse the decision are written into the code, so the next person inherits the reasoning rather than the conclusion.

Deciding not to build something is the same discipline as building it. It just leaves less to point at.

## What I got wrong

- **The k6 explanation**, described above. Wrong cause, confidently stated, corrected once the flag was the obvious suspect.
- **A fix of mine that introduced a bug.** Adding the client IP to the audit context looked clean, but `current_setting('x', true)` returns an empty string — not NULL — once it has been set in a session. On pooled connections the `COALESCE` chain then stored `''` forever. My own third test caught it, which is the only reason it is a footnote rather than a data-quality incident. The lesson generalizes: on a pooled connection, session state is not fresh state.

## Takeaway

The load work and the query work answered different questions and needed different instruments, but the same habit: **isolate one variable, and trust the plan over the intuition.**

The most valuable single output was not a speedup. It was being able to answer "what machine do we need" with a number and the experiment that produced it.

## Tools

k6 (custom scenarios, thresholds as contracts, `Trend`/`Rate`/`Counter`) · PostgreSQL `EXPLAIN (ANALYZE, BUFFERS)` · Testcontainers + Flyway for plan-guard tests · JUnit tag selection to keep them off the fast build · GitHub Actions · ADRs as the record of what was decided and why
