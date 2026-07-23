# AppFibra Architecture Notes

## Main Components

```mermaid
flowchart TB
    Users["Users"] --> Web["Web Office UI"]
    Users --> Mobile["Mobile Field UI<br/>offline-first"]

    Web --> API["Spring Boot Backend"]
    Mobile --> API

    API --> DB[("PostgreSQL / PostGIS")]
    API --> GIS["GIS Import & Tile Pipelines"]
    API --> BI["Reporting / BI / Scheduled Jobs"]
    API --> Agent["AI Agent Proxy"]
    API --> Photo["Photo Processing Proxy"]
    API --> Ext["External Workorders Integration"]

    GIS --> DB
    BI --> DB
```

## Key Design Decisions

- GIS-first data model using PostgreSQL/PostGIS.
- Native SQL/JDBC for heavy GIS and reporting workloads.
- Batch/precomputed tables for expensive construction study data.
- Offline-first mobile workflows for field operations.
- Enterprise security controls around user sessions and internal services.
- Federated identity and resource-level authorization for project/city-scoped access.
- Multi-client GIS evolution through configurable layer roles and client-specific resolvers.
- Internal M2M token authentication between Java and Python services.
- Append-only audit design for external system synchronization.

## Security Themes

- RBAC by client and module.
- Keycloak/OAuth2 identity foundation.
- Resource scopes validated in the backend before list, export, audit, dashboard, and mutation operations.
- Tenant isolation with database-level controls.
- CSRF on user mutations.
- Restricted CORS in production.
- Session and user audit trails.
- Internal service authentication for backend-to-agent and backend-to-photo calls.

```mermaid
flowchart LR
    User["User"] --> IdP["Keycloak / OAuth2"]
    IdP --> JWT["JWT with roles<br/>organizations + modules"]
    JWT --> API["Spring Boot Resource Server"]
    API --> Entitlement["Client / module entitlement"]
    Entitlement --> Scope["Resource scope guard"]
    Scope --> Query["Backend-filtered query"]
    Query --> Data[("Allowed project/city data")]
```

## Multi-Client GIS Pipeline

```mermaid
flowchart TB
    Source["Source files<br/>SHP / GPKG / DXF"] --> Import["Import runner<br/>staging tables"]
    Import --> Unified["Unified GIS tables<br/>client-aware data"]
    Unified --> Views["Materialized views<br/>latest features + analysis"]
    Views --> Resolver["Client resolver config<br/>layer roles + KPIs"]
    Resolver --> Status["Construction status<br/>precomputed study data"]
    Status --> UI["Tiles, dashboards<br/>and construction bar"]
```
