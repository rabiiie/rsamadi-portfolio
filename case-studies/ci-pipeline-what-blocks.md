# CI With Security Built In — Designing What Blocks and What Doesn't

> Sanitized write-up — no proprietary code, credentials, or client data. This one is about a design question that gets skipped: not *which* checks to run, but which ones are allowed to stop the work.

## The short version

A check that blocks from day one on a repository with history teaches the team to ignore CI. So new controls enter informative — with a stated expiry condition, never indefinitely — and every accepted vulnerability carries a date past which the build fails again.

The distinction that carries the most weight: a scanner *finding* issues doesn't fail its job, but the scanner *failing to run* does. Collapse those two and the pipeline reports that it runs SAST while running nothing. Underneath it all, the foundation that had to come first — seventy loose `.sql` files, applied by hand, that were blocking every check downstream.

## Context

One repository, polyglot: a Java 17 / Spring Boot backend, a React + Capacitor frontend, two Python services, and the full PostgreSQL/PostGIS schema. One developer. Real users in production.

The starting point was three checks — secret scanning, npm dependency audit, and a partial compile-and-test — with coverage uneven by language, the database schema outside any automatic control at all, and delivery done by hand.

## The rule the whole design starts from

> **A mature pipeline is not measured by how many scanners it runs, but by whether any commit on the main branch is deployable without manual intervention.**

Everything that doesn't contribute to that property is secondary, and no check gets added before the foundation that makes it meaningful exists.

That single sentence decided the ordering. Checks whose result wouldn't be *interpretable* today — coverage, integration tests, static analysis over legacy code — were deliberately postponed:

- **Coverage** is measured after the Spring context boots in CI, not before. Coverage computed over a test set that excludes application startup describes a different application.
- **Integration tests** come after versioned migrations, because it's the migrations that build the schema the test needs.
- **Style and bug analysis** enters with a baseline of the existing code, never on a clean slate.

## What blocks and what doesn't

This is the part that took the most thought.

| Check | Mode | Reasoning |
|---|---|---|
| Secret scanning over full history | fails the run | a leaked credential is not a matter of degree |
| Production dependency audit (npm) | fails the run | small, known surface; every finding is actionable today |
| Compile and unit tests | fails the run | — |
| Spring context boot against a real PostGIS container | fails the run | only became possible after the schema was versioned |
| SAST | **findings informative, tool failure fatal** | see below |
| Java dependency vulnerabilities | informative, scheduled | new advisories appear without the code changing |
| Frontend lint | informative, temporarily | 736 pre-existing warnings; hardens once triaged |

Two distinctions carry most of the weight here.

### New checks enter informative — with a stated hardening condition

A check that blocks from day one on a repository with history stops the work while its initial noise is triaged, and the team learns to ignore it. So new controls enter informative and harden when their finding list is empty or justified.

The important qualifier: informative status must carry a **date or a condition**, never indefinite permanence. Otherwise "informative" is just a permanent excuse.

The exception is any check that examines only a pull request's diff and not pre-existing code. Those block from the start, because they produce no historical noise.

### Findings informative is not the same as tool informative

The SAST job deliberately does **not** carry `continue-on-error`. Finding issues doesn't fail it — the scanner isn't invoked in error mode. But the scan *failing to run* does fail it.

With `continue-on-error`, both cases look identical: red, and ignored. A misconfiguration or a network failure would have gone unnoticed indefinitely, and the pipeline would have reported "we run SAST" while running nothing.

What gets attenuated is the **result of the analysis**, not the **health of the tool**.

This is the single most useful idea in the pipeline. A job that comes out red no matter what teaches people to ignore CI — which is worse than not having the job.

## Exceptions that expire

Dependency findings can be accepted, but every acceptance carries an expiry date. Past it, CI fails again: **an exception cannot become permanent by being forgotten.** And an exception that no longer matches a live advisory *also* fails the build, so the list can't quietly accumulate dead entries.

The policy attached to it: an exception is only admissible once the affected surface has been verified as unused. If the advisory has a fix, it gets fixed. Put the fire out — don't disable the alarm.

Secret scanning has the same shape. Historical findings are allowlisted **by commit**, each annotated with what the credential was and the date it was revoked, so scanning the full history doesn't block every run while anything *new* is still caught. The path allowlist contains only directories that aren't project code — dependencies, build artifacts, IDE files. Never our own source: a secret found in source gets rotated and removed from the file, not silenced.

## The foundation that had to come first

The schema wasn't versioned. Around seventy loose `.sql` files in the repository root, applied by hand, with no record anywhere of the application order, their idempotency, or the state of each environment.

That one gap was blocking everything downstream: the context-boot job was marked informative because the boot test needed a database with the schema already built, and there was no automatic way to build one — so the boot test was excluded from the test run entirely.

Fixing it:

- Baseline generated with `pg_dump --schema-only` over a containerized copy, then **verified by loading it into an empty database**: 351 relations, 868 user indexes, 1,014 functions, 424 constraints — identical to the source, no errors.
- Of 1,264 tables, **1,002 are created by the application at runtime** (per-project staging tables, daily history partitions). Those aren't schema and stay out; including them would make the baseline unreadable and force it to grow on its own. The parent partitioned tables are in.
- Both paths verified end to end: **empty database** → the migration runs, builds the schema, and JPA validation accepts it against the entities; **existing database** → a baseline is recorded and the migration does *not* run.
- The test container image is PostGIS, not plain `postgres` — the baseline creates spatial extensions. Those extensions' schemas are created `IF NOT EXISTS` because they arrive with the image. Application schemas are created *without* that clause, on purpose, so a real collision fails loudly.

And a decision that looks like a bug until you read it twice: **migrating is not a side effect of starting up.** Migrations stay disabled by default, because the application's runtime database user has no DDL permission and shouldn't have one. Applying migrations is a deployment step with its own credentials, not something that happens whenever a developer starts the app against a shared server. The startup failure that would otherwise occur is the correct answer to a wrong premise.

Two findings fell out of that verification that had nothing to do with CI — a partitioned table with no `DEFAULT` partition, and the fact that the first real-environment startup would write a migration-history table. Both were recorded before touching anything.

## Once there was a real database in CI, it could guard more than boot

The container built for the boot test turned out to be worth more than the test it was built for. A database that CI can create from scratch means CI can assert things about *how queries run*, not just that the application starts.

So the query plans became tests. They seed enough rows for the planner to take indexes seriously, then assert via `EXPLAIN` that each critical query still uses the index it was designed around. An index dropped by accident stops being a production incident discovered by a slow afternoon and becomes a red pull request.

Two things I'd want to know before copying this:

- `SET enable_seqscan = off` **prices** sequential scans, it doesn't forbid them. On a small fixture a missing index still yields a sequential scan instead of an error, so the assertion has to read the plan and the fixture has to be big enough to be honest.
- These tests are slow and they select by JUnit tag, so the fast build doesn't pay for them. They run as their own job — deterministic enough for every pull request, unlike timing measurements, which are not and run on a schedule instead.

The split matters: **plans per pull request, timings on a schedule.** A plan is a fact about the query and is stable on any machine. A timing is a fact about the machine, and asserting it on shared CI runners produces exactly the flaky red that teaches people to ignore the pipeline.

## Reality check: the plan met a paywall

The low-cost phase was written assuming code scanning, dependency review, and branch protection were cheap. On a **private** repository, all three sit behind the same paid tier. What's lost isn't the warning — it's the *block at merge time*.

The substitutions, each with what it actually costs:

- **Semgrep OSS instead of the GitHub-native scanner.** Covers Java and JavaScript, runs entirely on the runner, depends on no GitHub API, and is invoked with metrics disabled so no usage data leaves.
- **OWASP Dependency-Check instead of dependency review.** **Not equivalent**, and it matters: one scans the whole tree periodically, the other blocked a pull request's diff. The pre-merge block is gone. In exchange, Maven transitive dependencies get their first coverage of any kind.
- **Dependabot instead of Renovate.** Renovate groups better in a polyglot repository, but requires installing a third-party app with repository access. Dependabot is native, free on private repos, and its per-ecosystem grouping covers the reason Renovate was chosen — across six fronts: two Maven modules, npm, two Python services, and the workflow's own actions.

Writing down **what was lost** in each substitution matters as much as making it. Otherwise, six months later, the pipeline looks like it does something it doesn't.

Branch protection is recorded as **deferred pending a paid plan**, not quietly dropped. Until it exists, "fails the run" means the run goes red — not that a merge is mechanically prevented. Saying that plainly is the difference between a pipeline and a claim about one.

## Cost control

- **Path filtering**: a frontend-only pull request doesn't compile Java. Implemented through job outputs and `if` conditions rather than workflow-level path filters — a job skipped by `if` counts as satisfied for branch protection, while one skipped by a path filter leaves it waiting forever. That difference is easy to get wrong and expensive to debug.
- **Concurrency with cancel-in-progress on pull requests only.** On the main branch and on the scheduled run the job finishes: its result *is* that branch's history.
- **The slow scan runs on a schedule, not per PR.** A full vulnerability-database download takes the better part of an hour and tells you nothing new about a code change. It runs on push, weekly on cron, and on manual dispatch so it can still be exercised on demand.
- An SBOM is produced on every build and retained as an artifact, which is what makes a future license policy a configuration change rather than a project.

## Where it stands

Foundations and low-cost controls are green and running. The later phases — frontend tests starting with the authorization resolution logic, lint and dependency auditing for the Python services, coverage, architecture rules enforcing decisions that today live only in prose, automated staging delivery, artifact signing and provenance — are sequenced by the same rule that ordered the first two: a control is adopted when its result would be interpretable, not when it sounds valuable.

## Takeaway

The value of a pipeline isn't its scanner count. It's that **every red job means something, every exception has an expiry date, and every substitution has its cost written down.**

## Tools

GitHub Actions · gitleaks · Semgrep OSS · OWASP Dependency-Check · CycloneDX SBOM · Dependabot · Testcontainers (PostGIS) · Flyway · JUnit · ESLint · npm audit
