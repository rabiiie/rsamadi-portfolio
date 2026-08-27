# CI Pipeline — Which Checks Are Allowed to Block

> Sanitized: no proprietary code, credentials or client data.

## Scope

One polyglot repository: Java 17 / Spring Boot backend, React + Capacitor frontend, two Python services, and the full PostgreSQL/PostGIS schema. One developer. Production users.

Starting state: three checks — secret scanning, npm dependency audit, and a partial compile-and-test — with coverage uneven by language, the database schema under no automatic control, and delivery performed manually.

**Governing rule.** A pipeline is assessed by whether any commit on the main branch is deployable without manual intervention, not by the number of scanners it runs. Checks whose result would not be interpretable at the time of adoption are deferred:

- Coverage is measured after the Spring context boots in CI. Coverage over a test set that excludes application startup describes a different application.
- Integration tests follow versioned migrations, because the migrations build the schema the tests need.
- Style and bug analysis enters with a baseline of the existing code.

---

## 1. Blocking policy

| Check | Mode | Reasoning |
|---|---|---|
| Secret scanning over full history | fails the run | a leaked credential is not a matter of degree |
| Production dependency audit (npm) | fails the run | small known surface; every finding actionable today |
| Compile and unit tests | fails the run | — |
| Spring context boot against a real PostGIS container | fails the run | possible only after the schema was versioned |
| SAST | findings informative, tool failure fatal | see 1.2 |
| Java dependency vulnerabilities | informative, scheduled | new advisories appear without code changing |
| Frontend lint | informative, temporary | 736 pre-existing warnings; hardens once triaged |

### 1.1 New checks enter informative, with a hardening condition

A check that blocks from day one against a repository with history halts work while its initial findings are triaged, and is subsequently ignored. New controls enter informative and harden when their finding list is empty or justified.

Informative status carries a date or a condition. Without one it is permanent by default.

Exception: a check that examines only a pull request's diff and not pre-existing code blocks from the start, since it produces no historical noise.

### 1.2 Findings informative is not tool informative

The SAST job does not carry `continue-on-error`. Findings do not fail it — the scanner is not invoked in error mode. The scan failing to execute does fail it.

Under `continue-on-error` both cases are indistinguishable, and a misconfiguration or network failure would persist unnoticed while the pipeline reported that SAST runs. What is attenuated is the result of the analysis, not the health of the tool.

---

## 2. Expiring exceptions

Dependency findings can be accepted; every acceptance carries an expiry date, past which the build fails again. An exception that no longer matches a live advisory also fails the build, so the list cannot accumulate dead entries.

Admission policy: an exception is only admissible once the affected surface has been verified as unused. If the advisory has a fix, it is applied.

Secret scanning follows the same shape. Historical findings are allowlisted **by commit**, each annotated with the credential type and the date it was revoked, so full-history scanning does not block every run while new findings are still caught. The path allowlist contains only non-project directories — dependencies, build artifacts, IDE files. Source files are never allowlisted: a secret found in source is rotated and removed from the file.

---

## 3. Schema versioning

**Problem.** The schema was not versioned. Approximately 70 loose `.sql` files in the repository root, applied by hand, with no record of application order, idempotency, or the state of each environment.

**Downstream impact.** The context-boot job was informative because the boot test required a database with the schema already built and there was no automated way to build one — so the boot test was excluded from the test run entirely.

**Fix and verification.**

- Baseline generated with `pg_dump --schema-only` over a containerized copy, then verified by loading it into an empty database: 351 relations, 868 user indexes, 1,014 functions, 424 constraints, identical to the source, no errors.
- Of 1,264 tables, 1,002 are created by the application at runtime (per-project staging tables, daily history partitions) and are excluded — including them would make the baseline grow on its own. Parent partitioned tables are included.
- Both paths verified end to end. Empty database: the migration runs, builds the schema, and JPA validation accepts it against the entities. Existing database: a baseline is recorded and the migration does not run.
- The test container image is PostGIS rather than plain `postgres`, since the baseline creates spatial extensions. Those extensions' schemas are created `IF NOT EXISTS` because they ship with the image; application schemas are created without that clause so a real collision fails.

**Migrations are disabled by default.** The application's runtime database user has no DDL permission. Applying migrations is a deployment step with its own credentials, not a side effect of application startup against a shared server.

**Two defects recorded during verification**, unrelated to CI: a partitioned table with no `DEFAULT` partition, and the fact that the first real-environment startup would write a migration-history table.

---

## 4. Query plan assertions

The container built for the boot test also allows CI to assert how queries execute, not only that the application starts.

The plan tests seed enough rows for the planner to use indexes, then assert via `EXPLAIN` that each critical query still uses the index it was designed around. A dropped index fails the pull request instead of surfacing as a production incident.

Two implementation notes:

- `SET enable_seqscan = off` prices sequential scans, it does not forbid them. On a small fixture a missing index still yields a sequential scan rather than an error, so the assertion must read the plan and the fixture must be large enough to be representative.
- These tests are slow and select by JUnit tag, so the fast build does not run them. They execute as their own job, deterministic enough for every pull request — unlike timing measurements, which are not, and run on a schedule.

**Split:** plans per pull request, timings on a schedule. A plan is a property of the query and is stable across machines. A timing is a property of the machine, and asserting it on shared CI runners produces flaky failures.

---

## 5. Substitutions forced by repository tier

The low-cost phase assumed code scanning, dependency review and branch protection were available. On a private repository all three sit behind the same paid tier. What is lost is not the warning but the block at merge time.

| Planned | Substitute | What was lost |
|---|---|---|
| GitHub-native code scanning | Semgrep OSS | none material — covers Java and JavaScript, runs on the runner, no GitHub API dependency, metrics disabled |
| Dependency review | OWASP Dependency-Check | **not equivalent** — scans the whole tree periodically instead of blocking a PR diff. Pre-merge block gone; Maven transitive dependencies gain first coverage |
| Renovate | Dependabot | grouping quality in a polyglot repo; Dependabot is native, free on private repos, and its per-ecosystem grouping covers six fronts: two Maven modules, npm, two Python services, and the workflow's own actions |

Branch protection is recorded as deferred pending a paid plan. Until it exists, "fails the run" means the run goes red, not that a merge is mechanically prevented.

---

## 6. Cost control

- **Path filtering.** A frontend-only pull request does not compile Java. Implemented through job outputs and `if` conditions rather than workflow-level path filters: a job skipped by `if` counts as satisfied for branch protection, while one skipped by a path filter leaves it pending indefinitely.
- **Concurrency with cancel-in-progress on pull requests only.** On the main branch and the scheduled run the job completes, because its result is that branch's history.
- **The slow scan runs on a schedule.** A full vulnerability-database download takes close to an hour and reports nothing new about a code change. Triggers: push, weekly cron, manual dispatch.
- **SBOM produced on every build** and retained as an artifact, so a future license policy is a configuration change.

---

## Current state

Foundations and low-cost controls are green and running. Deferred phases, sequenced by the same adoption rule: frontend tests starting with the authorization resolution logic, lint and dependency auditing for the Python services, coverage, architecture rules enforcing decisions currently held only in prose, automated staging delivery, artifact signing and provenance.

## Tools

GitHub Actions · gitleaks · Semgrep OSS · OWASP Dependency-Check · CycloneDX SBOM · Dependabot · Testcontainers (PostGIS) · Flyway · JUnit · ESLint · npm audit
