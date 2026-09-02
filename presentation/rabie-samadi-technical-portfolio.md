# Puntos de conversación: AppFibra

Soy el único desarrollador de AppFibra, una plataforma SaaS para el despliegue de fibra FTTH en Alemania. Más de 200 municipios, más de 200.000 homes y tres clientes trabajando con ella en producción.

**Stack**: backend en Spring Boot, frontend en React, PostgreSQL/PostGIS para el dato espacial, FastAPI para la capa de agentes de IA, y Keycloak/OAuth2 para identidad.

**Responsabilidad**: ciclo completo. Requisitos, arquitectura, pipelines de GIS, seguridad, agentes, despliegue, y mantenerlo en pie cuando algo se rompe en producción.

**Puntos técnicos:**

- **GIS.** Vector tiles, resolvers de capa por cliente, tablas de estado de obra precalculadas.
- **Agentes de IA.** Sobre MCP, con una capa de tools semánticas y no text-to-SQL, y un circuito que convierte el feedback negativo en casos de evaluación, para medir el efecto de cada cambio de prompt.
- **Modelo de seguridad.** Keycloak/OAuth2, RBAC, scopes de recurso hasta proyecto o ciudad, autenticación M2M entre los servicios Java y Python.
- **La columna de auditoría.** Detectar cuándo un sistema externo sobrescribe datos en silencio, con diff campo a campo y capacidad de revertir.
- **Rendimiento.** Cuatro hipótesis sacadas de leer el código, las cuatro falsas, y una decisión de arquitectura mía revertida por su primera medición.
- **Capacidad.** Límite real de usuarios concurrentes, medido con prueba de carga y experimento controlado en lugar de estimado. En el segundo caso, la medición descartó la implementación.
- **Pipeline de CI.** Qué comprobaciones bloquean y cuáles informan, y con qué condición de endurecimiento.

**Alcance técnico**: dato espacial a escala, agentes de IA sobre datos de operación en producción y un refuerzo de seguridad aplicado durante una migración de identidad.

---

## Si la conversación va a rendimiento

*[Write-up completo: [rendimiento medido](../case-studies/measured-performance.md)]*

Las tablas de seguimiento manejan cientos de miles de registros con más de 40 columnas. Sin incidencias abiertas: revisión preventiva antes de ampliar el despliegue.

Cuatro hipótesis obtenidas leyendo el código, las cuatro descartadas. Indicadores parpadeando: `document.getAnimations()` devolvía 0 animaciones activas. Selectores `:hover` costosos: 0,1% del coste según Selector Stats. Doble render: `<StrictMode>`, solo en desarrollo. Virtualizador roto: el perfilado correspondía a otra tabla.

La causa real: una transición de 0,1 segundos declarada en la **celda** y no en la **fila**, multiplicada por cuarenta columnas hasta 952 animaciones simultáneas. El recálculo de estilo bajó del 45,7% del perfil al 16,1%.

**Punto de partida**: eliminación de un Web Worker que exigía una decisión de arquitectura **escrita por mí**. Lo medí, y serializar los datos para mandarlos costaba entre 7 y 10 veces más que el propio cálculo, en todos los tamaños de 25 a 10.000 filas. El envío a un worker no descarga el hilo principal, que paga el clonado completo antes de ceder el control. ADR corregido de "usar un worker" a "solo si el cálculo cuesta más que la transferencia", con las dos cifras obligatorias.

Criterio de método: medir la variabilidad antes que la diferencia, y aislar una variable por experimento.

## Si preguntan por escala, o "cuántos usuarios aguanta"

*[Write-up completo: [rendimiento medido](../case-studies/measured-performance.md#parte-2-servidor-y-base-de-datos)]*

Sin medición previa. Scripts de k6 que reproducen la mezcla real de gestos, con umbrales que tumban la ejecución. El codo está entre 150 y 300 usuarios concurrentes; a 300 se cruzaron los seis umbrales, a 187 req/s. Degradaron todos los gestos a la vez, lo que apuntaba a un recurso compartido y no a una query mala, y un experimento controlado separó el pool de conexiones (real, pequeño) de la CPU (dominante). La conclusión entregada al negocio fue ampliar núcleos, no ajustar parámetros que no eran la restricción.

La mitad de base de datos: un guardado de 730 ms a 113 ms quitándole trabajo por fila, y un gráfico de historial que leía 477.000 filas en cada petición. 283 ms aislados resultan asumibles; el plan muestra que cada petición reclutaba tres de las cuatro vCPU, y con diez usuarios concurrentes la mediana subía a 934 ms.

**Segundo caso**: al aplicar el mismo tratamiento a la segunda tabla, la medición lo desaconsejó. 232 filas en la ventana, y planificar la query costaba cuarenta y cinco veces más que ejecutarla. Así que el resumen precalculado no se construyó, y en su lugar entró en el código el número y el umbral que invertiría esa decisión.

Y los planes son ahora tests: corren contra un contenedor PostGIS real y comprueban con `EXPLAIN` que cada query crítica sigue usando su índice. Planes en cada pull request, tiempos en ejecución programada: el plan es propiedad de la query y el tiempo, del runner; comprobar tiempos en runners compartidos produce fallos intermitentes.

## Si la conversación va a entrega y CI

*[Write-up completo: [pipeline de CI](../case-studies/ci-pipeline-what-blocks.md)]*

Un repositorio políglota (Java, React, dos servicios Python y el esquema PostGIS entero), conmigo como único desarrollador y usuarios reales en producción.

Criterio de diseño: el objetivo es que cualquier commit de main sea desplegable sin intervención manual, no acumular escáneres. De ahí el orden: la cobertura no significa nada antes de que la aplicación arranque en CI, y los tests de integración son imposibles antes de versionar el esquema, así que esas dos cosas fueron primero.

Distinción relevante: hallazgos informativos no equivale a herramienta informativa. El job de SAST no falla cuando encuentra cosas, pero sí falla cuando el escaneo no llega a ejecutarse. Con un `continue-on-error` general los dos casos se ven igual, rojo e ignorado, y un escáner roto informaría de que SAST pasa mientras no se ejecuta.

Lo mismo con las excepciones de dependencias: cada una lleva fecha de caducidad, así que no puede volverse permanente por olvido, y una que ya no corresponde a ningún aviso vivo también tumba la build, así que la lista no acumula entradas muertas. Apagar el fuego, no desactivar la alarma.

La mitad del plan quedó condicionada al plan de pago: el escaneo de código y la protección de rama lo requieren en repositorios privados. Sustituí lo sustituible, documenté qué se pierde en cada sustitución y dejé la protección de rama anotada como aplazada.
