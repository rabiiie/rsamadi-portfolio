# Diagramas de la plataforma AppFibra

Vista de alto nivel, saneada: cómo encajan las piezas, sin detalle de implementación.

## Arquitectura de la plataforma

```mermaid
flowchart LR
    Web["Web de oficina"] --> API["Backend Spring Boot"]
    API --> DB[("PostgreSQL / PostGIS")]
    API --> GIS["Servicios GIS<br/>importación, vector tiles, análisis"]
    API --> IAM["Keycloak / OAuth2<br/>scopes por recurso"]
    API --> Agents["Agentes de IA<br/>FastAPI"]
    API --> Photos["Servicio de fotos"]
    API --> BI["Informes, cuadros de mando<br/>jobs programados"]
    API --> Ext["API externa de partes<br/>espejo + auditoría"]

    GIS --> DB
    BI --> DB
    Agents --> API
    Photos --> API
```

## Identidad y autorización por recurso

```mermaid
sequenceDiagram
    participant Usuario
    participant IdP as Keycloak/OAuth2
    participant API as API Spring Boot
    participant Guard as Guard de recurso
    participant DB as PostgreSQL/PostGIS

    Usuario->>IdP: Autenticarse
    IdP-->>Usuario: JWT con roles y módulos
    Usuario->>API: Pide datos, export o cuadro de mando
    API->>API: Valida emisor, firma y audiencia
    API->>Guard: Comprueba cliente, módulo y scope
    Guard->>DB: Consulta solo proyectos/ciudades permitidos
    DB-->>API: Resultado ya filtrado
    API-->>Usuario: Respuesta autorizada
```

## Pipeline GIS multicliente

```mermaid
flowchart TB
    Files["Ficheros de origen<br/>SHP / GPKG / DXF"] --> Staging["Importación a staging"]
    Staging --> Unified["Almacén unificado<br/>consciente del cliente"]
    Unified --> MVs["Vistas materializadas<br/>tiles y análisis"]
    MVs --> Resolver["Resolvers por cliente<br/>roles de capa y mapeos"]
    Resolver --> Study["Estado de obra precalculado<br/>tablas de estudio y KPI"]
    Study --> Frontend["Paneles de mapa, cuadros de mando<br/>comparación de obra"]
```

## Integración de los agentes de IA

```mermaid
flowchart LR
    UI["Chat / asistente"] --> Proxy["Proxy Spring Boot"]
    Proxy --> Agent["Plataforma de agentes<br/>FastAPI"]
    Agent --> Tools["Tools validadas<br/>consultas y acciones de dominio"]
    Tools --> Data[("Datos de operación")]
    Agent --> Eval["Feedback a evaluación<br/>detección de regresiones"]
    Agent --> Stream["Respuesta en streaming SSE"]
    Stream --> UI
```

## Controles de riesgo en producción

```mermaid
flowchart TB
    External["Sistema externo"] --> Mirror["Espejo local"]
    Mirror --> Audit["Auditoría solo de altas"]
    Audit --> Diff["Diff campo a campo"]
    Diff --> Alerts["Detección de cambios masivos"]
    Alerts --> Restore["Reconstrucción histórica<br/>export para revertir"]
```
