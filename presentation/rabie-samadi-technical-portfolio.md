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

**Why it's not "just another CRUD app"**: spatial data at scale, AI agents wired into real operational data instead of a demo, and security hardening that had to survive an actual identity migration — all shipped and running, not theoretical.
