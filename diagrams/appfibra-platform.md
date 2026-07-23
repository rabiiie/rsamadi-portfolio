# AppFibra Platform Diagrams

High-level and sanitized — the point is to show how the pieces fit together, not the implementation.

## Platform Architecture

```mermaid
flowchart LR
    Web["Web Office UI"] --> API["Spring Boot SaaS Backend"]
    Mobile["Mobile Field UI<br/>Capacitor + SQLite"] --> API
    API --> DB[("PostgreSQL / PostGIS")]
    API --> GIS["GIS services<br/>imports, vector tiles, analysis"]
    API --> IAM["Keycloak / OAuth2<br/>resource scopes"]
    API --> Agents["FastAPI AI Agents"]
    API --> Photos["Photo Processing Service"]
    API --> BI["Reports, dashboards<br/>scheduled jobs"]
    API --> Ext["External Workorders API<br/>mirror + audit"]

    GIS --> DB
    BI --> DB
    Agents --> API
    Photos --> API
```

## Identity & Resource Authorization

```mermaid
sequenceDiagram
    participant User
    participant IdP as Keycloak/OAuth2
    participant API as Spring Boot API
    participant Guard as Resource Guard
    participant DB as PostgreSQL/PostGIS

    User->>IdP: Authenticate
    IdP-->>User: JWT with roles/modules
    User->>API: Request data/export/dashboard
    API->>API: Validate issuer, signature, audience
    API->>Guard: Check client/module/resource scope
    Guard->>DB: Query only allowed projects/cities
    DB-->>API: Filtered result
    API-->>User: Authorized response
```

## Multi-Client GIS Pipeline

```mermaid
flowchart TB
    Files["Source files<br/>SHP / GPKG / DXF"] --> Staging["Staging import"]
    Staging --> Unified["Unified feature storage<br/>client-aware"]
    Unified --> MVs["Materialized views<br/>tiles + analysis"]
    MVs --> Resolver["Client-specific resolvers<br/>layer roles + mappings"]
    Resolver --> Study["Precomputed construction status<br/>study/KPI tables"]
    Study --> Frontend["Map panels, dashboards<br/>construction comparison"]
```

## AI Agent Integration

```mermaid
flowchart LR
    UI["User chat / assistant UI"] --> Proxy["Spring Boot proxy"]
    Proxy --> Agent["FastAPI Agent Platform"]
    Agent --> Tools["Validated tools<br/>queries + domain actions"]
    Tools --> Data[("Operational data")]
    Agent --> Eval["Feedback-to-eval<br/>regression checks"]
    Agent --> Stream["SSE streaming response"]
    Stream --> UI
```

## Production Risk Controls

```mermaid
flowchart TB
    External["External system"] --> Mirror["Local mirror"]
    Mirror --> Audit["Append-only audit"]
    Audit --> Diff["Field-by-field diff"]
    Diff --> Alerts["Mass-change detection"]
    Alerts --> Restore["Historical reconstruction<br/>restore export"]
```
