# Arquitectura de AppFibra

El mapa, no la historia. El razonamiento detrás de cada decisión concreta está en los [casos](../case-studies).

## Componentes principales

```mermaid
flowchart TB
    Users["Usuarios"] --> Web["Web de oficina"]
    Users --> Mobile["App de campo<br/>offline-first"]

    Web --> API["Backend Spring Boot"]
    Mobile --> API

    API --> DB[("PostgreSQL / PostGIS")]
    API --> GIS["Importación GIS y tiles"]
    API --> BI["Informes, BI y jobs programados"]
    API --> Agent["Proxy de agentes de IA"]
    API --> Photo["Proxy del servicio de fotos"]
    API --> Ext["Integración de partes externos"]

    GIS --> DB
    BI --> DB
```

## Decisiones de diseño

- **Modelo de datos GIS de origen**, sobre PostgreSQL/PostGIS. No un esquema relacional al que se le añade geometría después.
- **JPA/Hibernate para el CRUD sencillo** (autenticación), y **SQL nativo con JDBC para GIS e informes**, donde un ORM esconde justo los planes de ejecución que hay que tunear.
- **Los números caros de estudio de obra se precalculan** en tablas por lotes, y no se calculan en cada petición: calcularlos al vuelo no aguanta los objetivos de usuarios concurrentes de la plataforma.
- **El trabajo de campo es offline-first.** Las cuadrillas trabajan donde la cobertura no es fiable, así que las escrituras se encolan en el dispositivo y se sincronizan cuando vuelve la conexión.
- **El GIS pasó de un cliente a tres sin reescritura**, porque los roles de capa y los resolvers son configurables por cliente y no están cableados al primero.
- **Los servicios Java y Python se autentican entre sí con tokens M2M.** Ni secretos compartidos, ni endpoints internos abiertos por confianza.
- **Lo que llega del sistema externo de partes es solo-altas y auditado**, así que un cambio hecho por un tercero aparece como una fila y no como una sobrescritura silenciosa.

## Seguridad

- **RBAC por cliente y por módulo**: un rol concedido sobre los datos de un cliente no se filtra a los de otro.
- **Identidad federada con Keycloak/OAuth2**, no una tabla de usuarios propia. El JWT lleva roles, organizaciones y módulos.
- **El scope de recurso se comprueba en el backend** antes de cada listado, export, auditoría, cuadro de mando y mutación. No se esconde solo en la interfaz.
- **Row-level security en Postgres** como segunda capa detrás de las comprobaciones del backend, no como sustituta de ellas.
- **CSRF en cada mutación**, y CORS cerrado en producción, no abierto por comodidad.
- **La sesión y las acciones de usuario se auditan**, así que "quién hizo esto" tiene respuesta.
- **Las llamadas del backend al agente y al servicio de fotos se autentican igual** que tendría que hacerlo un cliente externo: que algo sea interno no lo hace de confianza.

```mermaid
flowchart LR
    User["Usuario"] --> IdP["Keycloak / OAuth2"]
    IdP --> JWT["JWT con roles<br/>organizaciones y módulos"]
    JWT --> API["Resource Server Spring Boot"]
    API --> Entitlement["Derecho por cliente y módulo"]
    Entitlement --> Scope["Guard de scope de recurso"]
    Scope --> Query["Consulta filtrada en el backend"]
    Query --> Data[("Datos de los proyectos permitidos")]
```

## Pipeline GIS multicliente

```mermaid
flowchart TB
    Source["Ficheros de origen<br/>SHP / GPKG / DXF"] --> Import["Runner de importación<br/>tablas de staging"]
    Import --> Unified["Tablas GIS unificadas<br/>datos por cliente"]
    Unified --> Views["Vistas materializadas<br/>features actuales y análisis"]
    Views --> Resolver["Configuración por cliente<br/>roles de capa y KPIs"]
    Resolver --> Status["Estado de obra<br/>datos de estudio precalculados"]
    Status --> UI["Tiles, cuadros de mando<br/>y barra de obra"]
```
