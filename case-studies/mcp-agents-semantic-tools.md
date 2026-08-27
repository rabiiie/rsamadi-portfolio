# Domain AI Agents on MCP — Giving a Model Tools Instead of a Database

> Sanitized write-up — no proprietary code, credentials, or client data. Identifiers are generalized. This one is about the interface between a language model and a business: what you expose, what you refuse to expose, and what a number has to carry with it before an agent is allowed to say it out loud.

## The short version

Two domain agents over an FTTH platform, answering operational questions in natural language — how many homes are passed in this municipality, which contracts were cancelled after the civil works were already done, what's ready for activation.

One is built on a **restricted SQL surface**. The other on **19 purpose-built tools over the Model Context Protocol**, wrapping the same business logic the official reports use. They're different on purpose, and the reason is the interesting part.

## Why not text-to-SQL

Text-to-SQL is the default demo, and it's the wrong default here for a reason that has nothing to do with whether the model can write SQL.

This platform's numbers already have definitions. "Homes passed" is a function over status codes, validated against the client's technical standard. The weekly report excludes certain projects. "Invoiced" means a specific hand-filled field is non-empty. Those definitions live in a semantic SQL layer that the reports read from.

A model writing its own SQL doesn't get any of that. It gets tables. It would produce a number that is *arithmetically correct and institutionally wrong* — and then state it fluently, in a sentence, to somebody who is going to paste it into a report that goes to a client.

So the rule the tool layer is built on: **tools wrap the existing business logic, they never reimplement it.** The agent and the official weekly report resolve to the same view. They cannot disagree, because there is only one definition and neither of them owns it.

That constraint is what makes the tools purpose-built rather than generic. Nineteen of them, parameterized — resolve a project, summarize milestones for one project or the whole portfolio, list contracts by status, fetch a single home's case, find what's ready for the next construction stage. Not one `query(sql)`.

**It also solved a problem I didn't anticipate.** Users don't say project codes, they say place names. A generic SQL tool would have had the model guessing at a join. A dedicated resolver tool turns the name into a code, or comes back saying it couldn't — which is a much better failure than a confident query against the wrong project.

## Every number arrives with its own coverage

This is the part I'd argue for hardest if someone wanted to cut it.

A dashboard can quietly drop rows it can't classify. The bar renders slightly shorter, or an "unknown" column sits there at 3 %, and a human reading a chart applies normal human skepticism to it.

An agent has no such affordance. It says *"4,812 homes passed"* in a well-formed sentence, and the fluency is itself an implicit claim of completeness.

So every aggregation tool returns the figure **plus a coverage block**: what was included, and what caveats apply. A period query over milestones comes back with, in effect, *"and 340 of these have no date recorded, so they're outside this period count."* If nothing was excluded, the caveat list is empty — which is a statement too.

Two smaller things ride along in the same payload, and both exist because of how a model reads its own tool output:

- **Provenance.** Each response names the view it came from, identified as *the source of the official weekly report*. If the user asks where the number comes from, that's data the model already has instead of something it invents.
- **A relationship the model must not get wrong.** One response returns two related metrics where one is a strict subset of the other, and it ships with the literal instruction `"HP+ is a subset of HP; do not add them."` Two numbers side by side is an invitation to sum them. That sentence costs nothing and removes a whole category of confidently-wrong answer.

The general shape: **if a number needs a caveat when a human says it, the tool has to carry that caveat in the payload.** Prompt instructions are the wrong place — they apply to the conversation, not to the number.

List tools are paginated with a hard row cap, and when results are truncated the response says so in a field the orchestrator can act on, with a message telling it to narrow the query. An LLM handed 50,000 rows doesn't answer better; it answers worse, more expensively, or not at all.

## Authorization, in three layers — and the one that ran on the wrong thread

The business requirement was stated plainly: assigned access to AI at all, and *if you're given one domain's agent, it only answers about that domain.*

**Barrier one and two** turned out to be the same fix. The agent endpoints were gated by a wildcard on a general application role — so anyone authenticated got in, and the agent identifier was a path segment the proxy forwarded without validating. A user could type a different agent's name into the URL. Now a single catalog maps agent to domain, and a guard checks it in both proxies *before* forwarding, which also means an unauthorized request is rejected without spending a model call.

**Barrier three** is resource scope, and it uses Token Relay rather than an internal API key: the browser's JWT goes to the Python orchestrator, which validates signature and expiry only, then forwards the token intact to the Java MCP server, which validates it again as a resource server and applies the same project-scope guard the REST API uses. **Python never decides authorization** — it authenticates, and that's all. The authorization decision stays in the service that owns the data, and there's exactly one implementation of it.

Then the part worth the write-up.

The MCP server executes tools on a **Reactor elastic scheduler — a different thread from the HTTP request**. Spring Security's context is thread-bound. So by the time a tool asked "which projects may this user see?", the security context was empty. The user's identity had been validated twice and then dropped on the floor between the filter and the tool.

It failed closed, which is the good direction. But the tempting fix is the dangerous one: the check is failing, the call is internal, the token was already validated upstream — so drop the check. That reasoning ends with the only enforcement point for project scope removed, and everything still working perfectly in every test, because in a test you're the user who has access to everything.

The actual fix captures the authentication at invocation time, exposes it through an explicit context for the duration of the tool call, and clears it in a `finally`. The clearing isn't decoration: a pooled scheduler thread that keeps one user's identity hands it to the next request.

**The generalizable version:** when you bolt a framework onto an async runtime, ask which of your security assumptions were quietly thread-bound. They don't announce themselves — they just stop being true.

## The other agent: a SQL surface built to be attacked

The first agent predates the MCP work and already answered through SQL. Rather than rewrite it, it got a real admission guard, and the design decisions there are worth stating because the naive version of each is wrong.

- **Allowlist of tables actually referenced, not a blocklist of dangerous words.** Blocklists lose. The question is "does every table in this query appear on the list", which is answerable.
- **Literals and comments are stripped before the analysis.** Otherwise a permitted table name inside a string or a comment can vouch for a query that also touches a forbidden one.
- **Unqualified table names must be on the list too** — and this is the one that isn't obvious. Postgres resolves them through `search_path`, which reaches the shared schema where other clients' data lives. Accepting unqualified names in bulk is a cross-client read with no exploit required. Anything belonging to that client must be written schema-qualified.
- **One schema is allowed wholesale**, because it belongs entirely to a single client. That's a deliberate ergonomic exception: adding a view for that client shouldn't require editing a security class, or people will stop adding views — or worse, put them in the shared schema.

Two agents, two surfaces, one principle: the surface is as wide as the domain requires and not one table wider.

## What actually broke: an SSE session reused across tasks

The MCP agent's stream would hang for about thirty seconds and die, then throw on teardown: *attempted to exit a cancel scope in a different task than it was entered in.*

**The diagnosis came from the comparison, not the stack trace.** The other agent — same platform, same model, same streaming path, but talking over plain HTTP instead of MCP — worked perfectly. That eliminated rate limiting, the network, the model, and the streaming layer in one step, and left the MCP client holding the bag.

The client opened its SSE connection once and reused the session across calls. The MCP SDK builds on anyio task groups, whose cancel scopes must be entered and exited **in the same task** — and the streaming response is an async generator, so every `yield` back to the web framework crosses a task boundary. The reader task was orphaned; the tool's response was never delivered; the connection sat there until it timed out.

Every call now opens, initializes, uses, and closes its own connection with no yields in between. That costs a reconnect per call. Connection pooling is a real optimization and it's written down as one — but it's an optimization on top of something correct, not a substitute for fixing it.

## Turning a thumbs-down into a test case

Every other write-up in this portfolio refuses to change something without a measurement. An agent is where that's hardest: the output is prose, the failure is "that answer was wrong-ish", and the temptation is to edit the prompt and see if it feels better.

So the feedback is wired to the evaluations.

A negative rating in production doesn't become a ticket. It's joined back to the conversation turn that produced it **and to the tool calls underneath it** — which tool ran, with what arguments — then heuristically classified into a probable failure category and inserted as a *pending candidate*, deduplicated by feedback id. A human confirms or corrects the category before it becomes a permanent evaluation case. The classifier is allowed to be wrong precisely because it isn't the last step.

That last detail is the one that makes it work. Without the tool calls attached, a bad answer is unattributable: you can't tell whether the model picked the wrong tool, extracted the wrong parameters, or got the right data and summarized it badly. Those are three different bugs with three different fixes, and the raw text of a complaint distinguishes none of them.

The suites split along exactly those lines — tool routing, planning, name resolution, answer synthesis, and per-domain question banks. Scoring uses a model as judge, because the answers are prose and string equality would only measure phrasing; runs are persisted with pass/fail per case, so a prompt change produces a number rather than an impression.

Two honest limits: the judge belongs to the same model family as the agent under test, which is a known weakness in this style of evaluation, and the suite runs sequentially to stay under rate limits, so it's slow enough that it's a deliberate act rather than something on every commit.

## Where it stands, honestly

The MCP path works end to end. The open item is stated rather than smoothed over: **the model runs on a free tier**, and the agent makes about two calls per question, so rapid successive questions hit a rate limit. The backoff then outlasts the servlet's async timeout, which turns a rate limit into a dead stream. Accepted and deferred by decision, not by oversight — moving the stream off the servlet path removes the timeout half, and the other half is a billing change.

The design was rewritten once already: the first version was drafted against raw tables, and reading real user questions against it showed the business logic it needed was already implemented in the semantic layer. The second version wraps that layer instead of duplicating it, and it exists because seven actual questions from actual users were used as the specification instead of an imagined API.

## Takeaway

The hard part of putting a model in front of a business is not the model. It's that **a language model's output has no visible uncertainty**, so everything that a human would have hedged has to be made structural: the definition it can't reinvent, the caveat travelling with the number, the scope check that survives the thread change, and the table it cannot name.

## Tools

Model Context Protocol · Spring AI MCP server (SSE) · Spring Boot · Spring Security OAuth2 Resource Server · Keycloak (Token Relay) · FastAPI · Python · anyio · Google Gemini · PostgreSQL semantic views · ADRs as the record of what was decided and why
