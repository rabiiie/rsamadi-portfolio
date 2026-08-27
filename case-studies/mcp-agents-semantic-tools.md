# Domain AI Agents on MCP — Semantic Tools Over Text-to-SQL

> Sanitized: no proprietary code, credentials or client data. Identifiers generalized.

## Scope

Two domain agents over an FTTH platform, answering operational questions in natural language: homes passed in a given municipality, contracts cancelled after civil works were completed, records ready for activation.

- Agent A: restricted SQL surface (pre-existing, retrofitted with an admission guard).
- Agent B: 19 purpose-built tools over the Model Context Protocol, wrapping the same views the official reports read.
- Orchestration: Python/FastAPI, streaming SSE. Tool server: Spring Boot, Spring AI MCP server over SSE.
- Authentication: Token Relay with Keycloak JWT.

---

## 1. Interface decision: tools, not text-to-SQL

**Constraint.** Platform metrics have existing definitions implemented in a semantic SQL layer: homes-passed is a function over delivery status codes validated against the client's technical standard; the weekly report excludes a list of projects; "invoiced" is a specific field being non-empty. A model generating its own SQL reads base tables and reproduces none of that. The output is arithmetically valid and inconsistent with the official report for the same question.

**Rule adopted.** Tools wrap the existing business functions; they never reimplement them. The agent and the weekly report resolve to the same view.

**Implementation.** 19 parameterized tools — project resolution, milestone summary per project and per portfolio, contracts by status, single-record case lookup, readiness queries, anomaly detection. No generic `query(sql)` tool.

**Secondary effect.** Users refer to municipalities by name; the database keys on project codes. A dedicated resolver tool converts name to code and returns a miss when it cannot, instead of the model constructing a join against the wrong project.

**Rejected during review of the first design.** `REPLACE(home_id, '-', '_')` in joins, which disables index usage. The fix is a normalized column in the database, not a string transform in the tool layer.

---

## 2. Coverage contract on aggregate responses

**Problem.** An aggregate that silently excludes rows is acceptable in a dashboard, where the reader sees a chart and applies normal skepticism. An agent emits the figure as a sentence, and the fluency carries an implicit claim of completeness.

**Design.** Every aggregation tool returns the figure plus a `coverage` object: `{included, excluded[{reason,count}], caveats}`. A period query over milestones returns the count for the period and, where applicable, a caveat naming the rows with no date recorded and therefore outside it. An empty caveat list is itself a statement.

Two additional fields in the same payload:

- `source` — names the view the figure came from, identified as the source of the official weekly report. Provenance is data the model holds rather than text it generates.
- Relationship constraints. One response returns two metrics where one is a strict subset of the other and ships the literal string `"HP+ is a subset of HP; do not add them"`. Two related numbers in one payload are otherwise summed.

**Rule.** A caveat required when a human states the number must travel in the payload, not in the system prompt. Prompt instructions apply to the conversation; they do not attach to the figure.

**Pagination.** List tools enforce a hard row cap. When results are truncated the response sets `has_more` and returns a message instructing the orchestrator to narrow the query. Aggregates and exports are separate channels and not subject to the list cap.

---

## 3. Authorization

Three layers, from a stated requirement: assigned access to AI at all, and domain restriction — a user granted one domain's agent must not receive answers about another.

**Layer 1 and 2 — agent admission.** The stream endpoint was gated by a wildcard path pattern plus a general application role, so any authenticated user passed. The agent identifier was a path segment forwarded by the proxy without validation, so a user could substitute another agent's identifier in the URL.

Fix: an `AgentCatalog` mapping agent identifier to domain, and `AgentGuard.assertAllowed(agentId)` called in both proxies before forwarding. Authority shape `ROLE_<domain>_AGENT`; one assignment satisfies both requirements. Rejection returns 403 without a model call. `SecurityConfig` changed from a wildcard to `.authenticated()`, with the authorization decision moved to the guard, which knows the agent identifier.

**Layer 3 — resource scope, via Token Relay.** The browser's JWT is sent to the Python orchestrator, which validates signature and expiry only (`PyJWKClient`) and forwards the token unmodified to the Spring MCP server. Spring validates it as an OAuth2 resource server and applies the same project-scope guard used by the REST API. Python performs authentication, never authorization. M2M and scheduled jobs continue to use an internal token header.

**Defect: security context lost across threads.** The Spring AI MCP server executes tool callbacks on a Reactor elastic scheduler, not the HTTP request thread. `SecurityContextHolder` is thread-bound, so the context was empty when a tool queried the caller's project scopes.

The failure was closed rather than open. The available shortcut — removing the check on the grounds that the call is internal and the token was validated upstream — would have eliminated the only enforcement point for project scope, with no test failing, because test users hold full access.

Fix: `AuthAwareToolCallbackProvider` wraps every `ToolCallback`, captures the `Authentication` at invocation, exposes it through `McpAuthContext` for the duration of the call, and clears it in `finally`. The clear is required: a pooled scheduler thread retaining one caller's identity would apply it to the next request.

---

## 4. SQL admission guard for the pre-existing agent

Agent A answered through generated SQL before the MCP work and was retrofitted with an admission guard rather than rewritten.

- **Allowlist over the tables actually referenced**, not a blocklist of keywords. The check is whether every table following `FROM` or `JOIN` appears on the list or is a CTE declared in the query's own `WITH`.
- **Literals and comments stripped before analysis.** Otherwise a permitted table name inside a string or comment can vouch for a query that also references a forbidden one.
- **Unqualified table names must also be listed.** PostgreSQL resolves them through `search_path`, which reaches the shared `public` schema holding other clients' tables. Admitting unqualified names in bulk permits a cross-client read. Client tables must be referenced schema-qualified.
- **One schema is allowed wholesale** because it belongs entirely to a single client, so adding a view for that client does not require editing the guard class.
- `MAX_SQL_LEN` 8000. Read-only execution path.

---

## 5. Incident: SSE session reused across asyncio tasks

**Symptom.** The MCP agent stream hung for approximately 30 seconds, then terminated. Teardown raised `RuntimeError: Attempted to exit cancel scope in a different task than it was entered in`.

**Isolation.** Agent A — same platform, same model, same streaming path, but calling over plain `httpx` rather than MCP — was unaffected. No 429 responses. That eliminated rate limiting, the network, the model and the streaming layer, and localized the fault to the MCP client.

**Root cause.** The client opened the SSE connection once (`connect()` with an `AsyncExitStack`, stored on `self.session`) and reused it across `call_tool`. The MCP SDK's `sse_client` and `ClientSession` use anyio task groups, whose cancel scopes must be entered and exited in the same task. The streaming response is an async generator, so each `yield` back to the web framework crosses a task boundary. The SSE reader task was orphaned and the tool response was never delivered.

**Fix.** `list_tools`, `call_tool` and `read_resource` each open, initialize, use and close their own SSE connection inside a single `async with self._session()` block with no yields in between. The per-call reconnect is the accepted cost; connection pooling is recorded as a later optimization.

---

## 6. Evaluation and feedback pipeline

**Problem.** Agent output is prose and failures are qualitative, so prompt changes are otherwise evaluated by impression.

**Feedback pipeline.** A negative rating (`rating = -1`) in production is joined to the conversation turn that produced it and to the `tool_calls` underneath it — which tool ran, with which arguments — then classified heuristically into a probable failure category and inserted into `eval_candidates` with `status='pending'`, deduplicated by feedback id. A human confirms or corrects the category before it becomes a permanent case.

The tool-call join is what makes a bad answer attributable: wrong tool selected, wrong parameters extracted, and correct data summarized badly are three distinct defects with three distinct fixes, and the complaint text distinguishes none of them.

**Eval suites.** Six: tool routing, planning, name resolution, answer synthesis, and two domain question banks. Scoring uses an LLM judge, since string equality would measure phrasing. Runs are persisted with per-case pass/fail.

**Known limitations.** The judge belongs to the same model family as the agent under test. The runner executes sequentially (`Semaphore(1)`) to stay under rate limits, so the suite is slow and is run deliberately rather than per commit.

---

## Current state

The MCP path works end to end. One open item: the model runs on a free tier and the agent issues approximately two model calls per question with `tool_mode=ANY`, so successive requests hit the rate limit. The backoff (~54 s) exceeds the servlet async timeout (30 s), converting a rate limit into a terminated stream. Deferred by decision. Moving the stream off the servlet path removes the timeout; the remainder is a billing change.

**Design revision.** The first design targeted base tables. Profiling seven real user questions against it showed the required business logic already existed in the semantic SQL layer, including a project exclusion list applied by the official report that tools would otherwise contradict. The second version wraps that layer. Six blocking questions were resolved by a profiling script first — actual enum values, field distributions and cardinalities — because filtering on an assumed value returns zero rows without an error.

## Tech stack

Model Context Protocol · Spring AI MCP server (SSE) · Spring Boot · Spring Security OAuth2 Resource Server · Keycloak (Token Relay) · FastAPI · Python · anyio · Google Gemini · PostgreSQL semantic views
