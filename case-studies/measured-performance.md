# Rendimiento y capacidad: diagnóstico en navegador y base de datos

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente.

## Contexto

Módulo de seguimiento de una plataforma FTTH. Tablas editables de cientos de miles de registros y
más de 40 columnas, con edición en línea, agrupación, filtros, ordenación, columnas fijas y
bloqueo optimista. Carga actual de 50 a 100 usuarios concurrentes; objetivo, 500.

Sin incidencias abiertas por parte de los usuarios. Revisión preventiva previa a la ampliación
del despliegue, con dos objetivos: localizar los límites de carga del sistema y dimensionar la
infraestructura de producción, que tenía asignadas cuatro vCPU sin medición que lo justificara.

Alcance: cliente y base de datos. Perfilado, instrumentación, correcciones y los tests que
impiden la regresión.

## Resultados

- **Repintado.** Desplazamiento del cursor sobre la tabla: recálculo de estilo del 45,7% al 16,1% del perfil, ocupación del hilo principal del 92,7% al 85,3%.
- **Escritura.** Guardado de una fila: de 730 ms a 113 ms, tras eliminar el trabajo por fila que se ejecutaba en cada operación.
- **Base de datos.** Histograma de historial: de 477.000 filas leídas por petición a un resumen diario de unas decenas.
- **Capacidad.** Sin degradación a 150 usuarios concurrentes, saturación en 300 (187 req/s). El dimensionado de producción se decidió con esa cifra: ampliación de núcleos, no ajuste de parámetros.
- **Web Worker eliminado** y ADR propio corregido: la serialización con `structuredClone` cuesta unas diez veces más que el cálculo que se pretendía descargar, en los cuatro tamaños probados.
- **Resumen precalculado descartado** antes de implementarlo: 232 filas frente a las 477.000 del caso que sí lo justificaba.

## Decisiones de ingeniería

- **Navegador.** Las 952 transiciones CSS simultáneas procedían de una transición declarada en la celda en lugar de la fila, multiplicada por 40 columnas. El comparador de `React.memo` de la fila omitía el flag de selección: defecto de corrección con síntomas de rendimiento.
- **Base de datos.** El resumen diario lo mantiene una tarea programada y no el trigger de auditoría, para no devolver el coste al camino de escritura recién optimizado. Recálculo por días completos, no por marca de agua sobre `id`, porque una secuencia no garantiza el orden de commit.
- **CI.** Tests de plan de ejecución con Testcontainers sobre PostGIS real: verifican con `EXPLAIN (ANALYZE, BUFFERS)` el uso de índice de cada consulta crítica. La eliminación de un índice bloquea el pull request.

**Stack.** Chrome DevTools · React DevTools Profiler · PostgreSQL `EXPLAIN (ANALYZE, BUFFERS)` ·
k6 · Testcontainers · JUnit 5 · GitHub Actions. Detalle en [Herramientas](#herramientas).

---

## Metodología de diagnóstico

**Validación del entorno de pruebas.** Las mediciones se realizaron exclusivamente sobre la build
compilada de producción. En el servidor de desarrollo, el overhead de React y `<StrictMode>`
arrojaban falsos positivos: un INP inicial de 3 segundos y una supuesta fuga de memoria se
descartaron al medir sobre el entorno real.

Pestañas de DevTools, en orden de coste de análisis:

1. **Consola.** Sin errores ni avisos. Descarta bucles de render.
2. **Red.** Peticiones por gesto, tamaño y encadenamiento. Los 232 ms de apertura del editor de celda no son de React: son una comprobación de permisos contra el servidor previa a la edición. Otra capa, fuera de alcance.
3. **Rendimiento.** Grabación de una interacción aislada (solo cursor, sin scroll ni clics), guardada como línea base. Lectura de *self* frente a *total*: el recálculo de estilo era el efecto, la causa estaba en las transiciones (§4).
4. **Selector Stats**, sobre esa misma grabación. Coste por selector y número de elementos alcanzados: 7.332 invalidaciones desde un único selector (§3).
5. **Memoria.** Dos capturas de heap y vista de comparación, sobre build de producción.
6. **React Profiler.** Tres props reconstruidas en cada render y un comparador de `React.memo` incompleto, localizados registrando el cambio de identidad de cada prop por fila (§3).

Instrumentación propia donde las herramientas estándar no llegan: `document.getAnimations()` para
el número de animaciones activas y un `MutationObserver` para las mutaciones del DOM por
interacción. De ahí salen las 952 transiciones simultáneas y las 232 mutaciones al editar la
primera celda. Un conteo es estable entre ejecuciones; un tiempo absoluto depende de la máquina.

Hipótesis explícitas y verificación de todas: las cuatro iniciales, obtenidas por lectura de
código, se descartaron (§2). Instrumento invariable entre ejecuciones: una modificación del
script de carga entre la medición previa y la posterior invalidó esa comparación.

En servidor, el mismo orden: verificación de que se ejecuta el escenario declarado (§6),
throughput y no usuarios virtuales (§7), prueba de una sola variable para distinguir pool de CPU
(§8), y plan de ejecución con `EXPLAIN (ANALYZE, BUFFERS)` (§9). Cierre automatizado (§11).

---

## Parte 1. Navegador

### 1. Descarte de la fuga de memoria

Repetida la captura de heap sobre la build de producción, los nodos desconectados no aparecen.
Los retenía la contabilidad del hot-module-replacement.

### 2. Hipótesis iniciales y su verificación

Cuatro hipótesis formuladas por lectura del fuente. Ninguna se confirmó.

| Hipótesis | Verificación | Resultado |
|---|---|---|
| Los indicadores de estado parpadean: siete clases CSS declaran `animation: … infinite` | `document.getAnimations()` sobre la página en ejecución | 0 animaciones activas |
| Selectores `:hover` costosos, con cuatro `:not()` encadenados | Selector Stats de DevTools | 13,5 ms de 13.006 ms de recálculo de estilo (0,1%) |
| Un componente padre re-renderiza dos veces por interacción | Build sin `<StrictMode>` | Doble invocación exclusiva de desarrollo |
| Virtualizador defectuoso: monta 100 filas para 18 visibles | Recuento de filas en el DOM | El perfilado correspondía a una tabla distinta de la modificada |

### 3. Hallazgos y correcciones

1. **952 transiciones CSS simultáneas.** Transición de `background-color` de 0,1 s declarada en la celda en lugar de la fila, multiplicada por 40 columnas. Trasladada a la fila. Origen del parpadeo reportado.
2. **7.332 invalidaciones de estilo desde un único selector.** Rayado y hover aplicados a las celdas (`tr:hover td:not(.pinnedCell)`): el desplazamiento del cursor invalidaba las 40. Trasladado a la fila.
3. **`will-change` sobre elementos cuya propiedad animada no se compone.** Una capa de compositor por celda sin contenido que componer. Retirado.
4. **Indicador de progreso animando `width`**, con layout forzado en cada frame. Sustituido por `transform: scaleX()`.
5. **Getter del contenedor de scroll del virtualizador devolviendo el contenedor incorrecto**: montaba 100 filas en lugar de 28. Corregido.
6. **Tres props reconstruidas en cada render**, que anulaban el `React.memo` de todas las filas.
7. **Comparador de `React.memo` sin el flag de selección.** La fila solo se repintaba cuando un render ajeno la forzaba: no se actualiza hasta hacer scroll, y entonces aparece al día. Defecto de corrección, no de latencia, y superaba las pruebas manuales.

### 4. Medición antes y después, con el cursor como única variable

Experimento aislado: únicamente desplazamiento del cursor sobre la tabla, sin scroll, clics ni
escritura. Proporciones del perfil y no milisegundos absolutos, por tratarse de una máquina
compartida.

| Métrica | Antes | Después |
|---|---|---|
| `Recalculate style` (self) | 45,7 % | 16,1 % |
| Rendering | 52,3 % | 36,2 % |
| Ocupación del hilo principal | 92,7 % | 85,3 % |
| `Event: animationiteration` | 35,0 % | no aparece |
| Animaciones simultáneas (pico) | 953 | 1 · 15 · 4 |

`Recalculate style` pasa de 7,5% self / 42,1% total a 16,1% self ≈ 16,1% total: el trabajo
restante es propio y no arrastrado por las transiciones.

Mutaciones del DOM por interacción, con un `MutationObserver` sobre las filas montadas:

| Interacción | Filas afectadas | Mutaciones | Lectura |
|---|---|---|---|
| Marcar el check de una fila | 1 | 3 | Mínimo posible |
| Editar una celda con el grupo ya activo | 2 | 25 | Correcto: la celda que pierde el foco y la que lo gana |
| Editar la primera celda | 28 | 232 | Legítimo: el doble clic activa el grupo y todas las filas visibles pasan a editables |

El tercer caso tiene el perfil de una regresión. Se aisló la variable activando el grupo primero
y midiendo después la segunda celda; sin ese paso, la optimización habría eliminado un repintado
que la funcionalidad requiere.

### 5. Revisión de un ADR propio: eliminación del Web Worker

Un ADR anterior, redactado por mí, exigía calcular la completitud por grupo en un Web Worker. La
decisión no tenía medición asociada.

Banco de microbenchmarks con el algoritmo de producción sobre 40 columnas y filas sintéticas con
la forma de las reales. Medianas de 40 muestras, por lotes: una pasada única devuelve
`0.000 ms` por la resolución de `performance.now()`.

| Filas | Hilo principal | Solo `structuredClone` | Worker persistente | Worker por cambio |
|---|---|---|---|---|
| 25 | 0,014 ms | 0,105 ms | 0,300 ms | 5,600 ms (412×) |
| 100 | 0,034 ms | 0,356 ms | 0,500 ms | 5,700 ms (170×) |
| 1.000 | 0,356 ms | 3,195 ms | 3,800 ms | 9,700 ms (27×) |
| 10.000 | 4,810 ms | 46,840 ms | 55,300 ms | 86,900 ms (18×) |

Con 10.000 filas, cálculo 4,810 ms y serialización 46,840 ms. El hilo principal paga el
`structuredClone` completo antes de ceder el control, de modo que el worker no descarga nada: el
algoritmo es una pasada lineal que solo lee, y la clonación recorre la misma estructura
reservando memoria.

ADR corregido de "usar un worker" a "solo si el cálculo cuesta más que la transferencia", con las
dos cifras obligatorias antes de mover trabajo. Ficheros del worker eliminados.

---

## Parte 2. Servidor y base de datos

### 6. Pruebas de carga: el escenario que no se ejecutaba

Los escenarios declarados combinan gestos reales (listar, filtrar, editar, exportar, abrir
historial) en las proporciones de producción. Las primeras ejecuciones daban resultados
favorables y la salida nombraba un escenario `default` que el script no define.

`K6_VUS`, `K6_DURATION` y `K6_ITERATIONS` son opciones propias de k6, también al pasarlas como
`-e K6_VUS=300`. k6 las consume, descarta los escenarios declarados y ejecuta uno plano. La
combinación de gestos no llegó a ejecutarse.

El primer diagnóstico fue incorrecto: atribuí el problema a una variable de entorno perdida en la
máquina, cuando el origen era el flag de línea de comandos.

Los scripts pasaron a leer `LOAD_VUS` y `LOAD_DURATION`, que k6 no reclama, y a verificar que el
escenario en ejecución es el esperado, con un fallo explícito que nombra la causa probable.

### 7. Punto de saturación

Throughput en peticiones por segundo. Los usuarios virtuales son la presión aplicada, no el
resultado.

| Carga | Resultado |
|---|---|
| 150 concurrentes | Todos los umbrales en verde salvo el p95 del export (539 ms) |
| 300 concurrentes | Los seis umbrales superados, 187 req/s |

El codo se sitúa entre ambos puntos. La degradación afecta a todos los gestos a la vez, lo que
descarta una consulta concreta y apunta a un recurso compartido: pool de conexiones o CPU.

### 8. Pool de conexiones frente a CPU

Prueba controlada: modificar el tamaño del pool, sin ningún otro cambio, y repetir la medición.
El pool aportaba una mejora real, pero el factor dominante era la CPU: cuatro vCPU son el techo y
ningún ajuste del pool lo desplaza. La conclusión entregada al negocio fue ampliar núcleos.

### 9. Optimización en base de datos

**Guardado, de 730 ms a 113 ms.** Trabajo por fila ejecutándose en cada guardado. Eliminado, no
paralelizado.

**Histograma de historial, 477.000 filas leídas por petición.** Recalculaba en cada petición
cifras ya inmutables. `EXPLAIN (ANALYZE, BUFFERS)` sobre la ventana de 30 días:

```
Parallel Index Scan using idx_..._timestamp
  rows=159009 loops=3        <- 477.027 filas leídas
Workers Launched: 2
Execution Time: 283 ms
```

Cada petición reclutaba tres de las cuatro vCPU; con diez usuarios concurrentes la mediana subía
a 934 ms.

La tabla solo recibe altas y se sella con la hora actual, de modo que 29 de los 30 días del
gráfico son inmutables. La consulta pasa a leer un resumen diario de unas decenas de filas. Dos
decisiones de implementación:

- **Tarea programada, no trigger.** Mantener el resumen desde el trigger de auditoría cargaría el coste a cada guardado, que acababa de bajar de 730 ms por retirarle trabajo. El camino de escritura desconoce la existencia del resumen.
- **Recálculo de dos días completos, sin marca de agua por `id`.** Una secuencia no garantiza el orden de commit: la transacción con el id 100 puede confirmar antes que la del 99, y la marca de agua omitiría esa fila de forma permanente. El recálculo por día es idempotente.

Además: paginación por keyset en sustitución de `OFFSET`, e índices GIN de trigramas para las
búsquedas de texto que estaban haciendo scan.

### 10. Resumen precalculado descartado

La propuesta consistía en replicar ese mecanismo en la segunda tabla. Medición previa a la
implementación:

```
Index Only Scan
  rows=232          <- 11.232 filas en toda la tabla
Planning Time:  25.137 ms
Execution Time:  0.560 ms
```

232 filas frente a 477.000, y la planificación cuesta 45 veces más que la ejecución. No se
implementó: no había ganancia, y añadía una tabla, una tarea programada y un minuto de desfase.
La medición y el umbral de volumen que invertiría la decisión quedaron documentados en el código.

---

## 11. Mecanismos que quedaron en el repositorio

Directorio `qa/` con scripts de carga ejecutables, banco de microbenchmarks y write-ups de
medición fechados, uno por sesión, con las cifras que justificaron cada cambio.

Planes de ejecución convertidos en tests. Corren contra un contenedor PostGIS real y verifican
con `EXPLAIN` que cada consulta crítica sigue usando el índice para el que se diseñó; la
eliminación de un índice deja el pull request en rojo. El mecanismo es genérico por tabla, no
específico de la primera. Construcción y trampas, en el
[write-up del pipeline](ci-pipeline-what-blocks.md#4-tests-sobre-los-planes-de-ejecución).

El trabajo de navegador se aplicó después a una segunda tabla (historial de auditoría, reversión
de cambios y los componentes compartidos resultantes), una vez la primera estaba correcta. Esa
segunda pasada costó días en lugar de semanas: los mecanismos ya existían y solo las mediciones
eran nuevas.

## 12. Defecto introducido durante el trabajo

Al incorporar la IP de cliente al contexto de auditoría, `current_setting('x', true)` devuelve
cadena vacía y no NULL una vez fijada en la sesión, de forma que sobre conexiones de pool la
cadena de `COALESCE` almacenaba `''` de forma permanente. Lo detectó un test escrito para ese
mismo cambio, antes de publicarlo. En una conexión de pool, el estado de sesión no es estado
limpio.

## Errores de instrumentación detectados

- `set scrollTop` ocupando el 36% del perfil correspondía al script de carga propio, no a la aplicación.
- Dobles renders en desarrollo procedentes de `<StrictMode>`, interpretados inicialmente como dobles renders reales.
- INP de 3 segundos medido contra el servidor de desarrollo.
- Modificación del script de carga entre una ejecución posterior y una previa, que invalidó esa comparación. Repetida con el instrumento invariable.
- k6 consumiendo las variables y ejecutando un escenario no declarado ([§6](#6-pruebas-de-carga-el-escenario-que-no-se-ejecutaba)).

## Puntos abiertos

- `Commit`, `Hit test` y `Layerize` suponen el 55% del perfil de hover: composición y pintado, no cálculo de estilo. Rendimientos decrecientes, sin planificación mientras no haya impacto reportado.
- INP de 232 ms en la apertura del editor de celda, no atribuible a React: ida y vuelta de red para comprobación de permisos previa a la edición. Se trata en otra capa.

## Herramientas

Chrome DevTools (capturas de heap y vista de comparación, panel de Performance con self y total,
Selector Stats, Live Metrics/INP) · `document.getAnimations()` · `MutationObserver` · React
DevTools Profiler · scripts de carga y banco de microbenchmarks propios · k6 (escenarios propios,
umbrales como contrato, `Trend`/`Rate`/`Counter`) · `EXPLAIN (ANALYZE, BUFFERS)` de PostgreSQL ·
Testcontainers y Flyway para los tests de plan · selección por tag de JUnit para mantenerlos
fuera de la build rápida · GitHub Actions · ADRs
