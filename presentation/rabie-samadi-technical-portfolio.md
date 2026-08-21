# Talking Points — AppFibra

If I only get 60 seconds to explain what I built:

I'm the only developer on AppFibra, a SaaS platform for FTTH fiber deployment in Germany — 200+ municipalities, 200,000+ homes, three clients running on it in production.

**Stack**: Spring Boot backend, React frontend, PostgreSQL/PostGIS for the spatial data, FastAPI for the AI agent layer, Keycloak/OAuth2 for identity.

**What I actually own**: everything, end to end — requirements, architecture, GIS pipelines, security, AI agents, deployment, and keeping it running when something breaks in production.

**The parts I'd want to talk about in an interview:**

- The GIS side — vector tiles, multi-client layer resolvers, precomputed construction status tables.
- The AI agents — built on MCP with a semantic tool layer instead of text-to-SQL, with a feedback-to-eval loop instead of guessing whether a prompt change helped.
- The security model — Keycloak/OAuth2, RBAC, resource scopes down to project/city level, M2M auth between the Java and Python services.
- The audit backbone — catching when an external system silently overwrites data, with field-level diffing and restore.
- The performance work — four confident hypotheses from reading code, all four wrong, and one architecture decision of my own reversed by its first measurement.
- Capacity — how many concurrent users the platform actually takes, answered with a load test and a controlled experiment rather than an estimate, and the second time the measurement said don't build the thing at all.
- The CI pipeline — the design question isn't which scanners to run, it's which ones are allowed to stop the work.

**Why it's not "just another CRUD app"**: spatial data at scale, AI agents wired into real operational data instead of a demo, and security hardening that had to survive an actual identity migration — all shipped and running, not theoretical.

---

## If the conversation turns to performance

*[Full write-up: [Measured Performance Diagnosis](../case-studies/measured-performance-diagnosis.md)]*

The follow-up grids handle hundreds of thousands of records across 40+ columns, and users said hovering and scrolling felt heavy.

I had four hypotheses from reading the code. All four were wrong. The blinking indicators I blamed — `document.getAnimations()` said zero were running. The expensive `:hover` selectors — Selector Stats put them at 0.1% of the cost. The double render — that was StrictMode, development only. The broken virtualizer — I was profiling a different grid than the one I'd changed.

What it actually was: a 0.1-second transition declared on the *cell* instead of the *row*, multiplied by forty columns into 952 concurrent animations. Style recalculation went from 45.7% of the profile to 16.1%.

**The one I'd lead with**: I deleted a Web Worker that an architecture decision *I had written* required. I benchmarked it and serializing the data to send it cost 7-10× more than the computation itself, at every size from 25 to 10,000 rows. Sending data to a worker doesn't take it off the main thread — the main thread pays the full structured clone before letting go. I rewrote the decision record from "use a worker" to "only if computing exceeds transferring", and required both numbers to be measured before anything else goes into one.

The habit underneath all of it: measure the variance before you measure the difference, and isolate one variable at a time. Everything else was tooling.

## If they ask about scale, or "how many users can it take"

*[Full write-up: [Capacity and Database Performance](../case-studies/capacity-and-database-performance.md)]*

Nobody knew, so I measured it. k6 scripts that reproduce the real mix of gestures, with thresholds that fail the run. The knee sits between 150 and 300 concurrent users; at 300 all six thresholds crossed at 187 req/s. Every gesture degraded together, which pointed at a shared resource rather than one bad query — and a controlled experiment separated the connection pool (small, real) from the CPU (dominant). That told the business to buy cores instead of paying me to tune settings that weren't the constraint.

The database half: a save path from 730 ms to 113 ms by taking per-row work off it, and a history chart that was reading 477,000 rows on every request. That one is the better story — 283 ms sounds survivable until the plan shows each request recruiting three of the four vCPU, which is why ten concurrent users measured 934 ms. **Parallelism that helps one user is what sinks ten.**

**The one I'd lead with here**: when I took the same treatment to the second table, the measurement said don't. 232 rows in the window, and planning the query cost forty-five times more than executing it — so the precomputed summary was never built, and the number plus the threshold that would reverse it went into the code instead. Deciding not to build is the same discipline; it just leaves less to point at.

And the plans are now tests: they run against a real PostGIS container and assert via `EXPLAIN` that each critical query still uses its index. Plans per pull request, timings on a schedule — a plan is a fact about the query, a timing is a fact about the machine, and asserting timings on shared runners just teaches people to ignore the pipeline.

## If the conversation turns to delivery and CI

*[Full write-up: [CI With Security Built In](../case-studies/ci-pipeline-what-blocks.md)]*

One polyglot repo — Java, React, two Python services, and the whole PostGIS schema — with me as the only developer and real users in production.

The rule I designed it around: **a mature pipeline isn't measured by how many scanners it runs, but by whether any commit on main is deployable without manual intervention.** That decided the ordering — coverage is meaningless before the app boots in CI, and integration tests are impossible before the schema is versioned, so those came first.

The distinction I'd actually want to discuss: **findings informative is not the same as tool informative.** The SAST job doesn't fail when it finds things, but it does fail when the scan doesn't run. With a blanket `continue-on-error` those two look identical — red and ignored — and a broken scanner would report "we run SAST" while running nothing. A job that's always red teaches people to ignore CI, which is worse than not having it.

Same thinking on dependency exceptions: each one carries an expiry date, so it can't become permanent by being forgotten, and one that no longer matches a live advisory also fails, so the list can't accumulate dead entries. Put the fire out; don't disable the alarm.

And the honest part: half the plan hit a paywall — code scanning and branch protection need a paid tier on private repos. I substituted what I could, wrote down **what each substitution lost**, and recorded branch protection as deferred rather than quietly dropping it. Six months from now, nobody should be able to mistake this pipeline for doing something it doesn't.
