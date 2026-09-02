# Rendimiento y capacidad: diagnóstico en navegador y base de datos

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente.

## Contexto

Módulo de seguimiento de una plataforma FTTH. Tablas editables de cientos de miles de registros y
más de 40 columnas, con edición en línea, agrupación, filtros, ordenación, columnas fijas y
bloqueo optimista. Carga actual de 50 a 100 usuarios concurrentes; objetivo, 500.

No había incidencias abiertas por parte de los usuarios. El trabajo fue una revisión preventiva
antes de ampliar el despliegue, con dos objetivos: localizar los límites de carga del sistema y
dimensionar la infraestructura de producción, que tenía asignadas cuatro vCPU sin ninguna
medición que lo justificara.

Alcance: las dos mitades del mismo problema, cliente y base de datos. Perfilado, instrumentación,
correcciones y los mecanismos automáticos que quedaron para impedir regresiones.

## Resultados

- **Repintado.** Durante el desplazamiento del cursor sobre la tabla, el recálculo de estilo baja del 45,7% al 16,1% del perfil, y la ocupación del hilo principal del 92,7% al 85,3%.
- **Escritura.** Guardado de una fila: de 730 ms a 113 ms, tras eliminar el trabajo por fila que se ejecutaba en cada operación.
- **Base de datos.** El histograma de historial pasa de leer 477.000 filas por petición a consultar un resumen diario de unas decenas.
- **Capacidad.** Pruebas de carga sin degradación a 150 usuarios concurrentes y punto de saturación localizado en 300 (187 req/s). El dimensionado de producción se decidió con esa cifra: ampliación de núcleos de CPU, no ajuste de parámetros.
- **Web Worker eliminado** y ADR propio corregido, tras verificar que el coste de serialización con `structuredClone` es unas diez veces superior al del cálculo que se pretendía descargar, en los cuatro tamaños de entrada probados.
- **Resumen precalculado descartado** antes de implementarlo, al no mostrar la medición previa ninguna ganancia.

## Decisiones de ingeniería

- **Navegador.** Las 952 transiciones CSS simultáneas se originaban en una transición declarada a nivel de celda en lugar de fila, multiplicada por 40 columnas. El comparador de `React.memo` de la fila omitía el flag de selección, un defecto de corrección con síntomas de rendimiento.
- **Base de datos.** El resumen diario lo mantiene una tarea programada y no el trigger de auditoría, para no trasladar el coste al camino de escritura recién optimizado. El recálculo es por días completos, no por marca de agua sobre `id`: una secuencia no garantiza el orden de commit.
- **CI.** Los planes de ejecución están cubiertos por tests con Testcontainers sobre PostGIS real, que verifican con `EXPLAIN (ANALYZE, BUFFERS)` el uso de índice de cada consulta crítica. La eliminación de un índice bloquea el pull request.

**Stack.** Chrome DevTools · React DevTools Profiler · PostgreSQL `EXPLAIN (ANALYZE, BUFFERS)` ·
k6 · Testcontainers · JUnit 5 · GitHub Actions. Detalle en [Herramientas](#herramientas).

---

## Metodología de diagnóstico

**Requisito previo: build de producción.** El perfilado se hace sobre la aplicación compilada tal
como se despliega, no sobre `npm run dev`. En desarrollo React no está minificado, ejecuta
comprobaciones adicionales y `<StrictMode>` invoca cada render dos veces, de forma que toda
medición incluye trabajo que el usuario nunca ejecuta. Dos consecuencias directas en este
trabajo: la sospecha de fuga de memoria del §1 se descartó al repetirla sobre la build correcta,
y un INP de 3 segundos registrado inicialmente correspondía al servidor de desarrollo.

Orden de las pestañas de DevTools, de menor a mayor coste de análisis:

1. **Consola.** Un minuto de revisión. Errores y avisos de React aparecen ahí y con frecuencia contienen el diagnóstico. En este caso, sin errores ni avisos: eso descarta bucles de render y advertencias repetidas por pasada.
2. **Red.** Peticiones que dispara cada gesto, cantidad, tamaño y encadenamiento. Localiza el trabajo que no corresponde al navegador. Resultado: los 232 ms de apertura del editor de celda no proceden de React, sino de una ida y vuelta de comprobación de permisos previa a la edición. Corresponde a otra capa y queda fuera de este trabajo.
3. **Rendimiento.** Grabación de una única interacción aislada (solo desplazamiento del cursor, sin scroll ni clics), guardada como línea base antes de cualquier cambio. Lectura de *self* frente a *total*: el coste alto puede señalar al efecto y no a la causa. El recálculo de estilo era el efecto; la causa estaba en las transiciones (§4).
4. **Selector Stats**, dentro de esa misma grabación. Aporta el coste de cada selector CSS y el número de elementos alcanzados, dato que no se obtiene leyendo la hoja de estilos. Origen de las 7.332 invalidaciones desde un único selector (§3).
5. **Memoria.** Dos capturas de heap y vista de comparación para la sospecha de fuga, siempre sobre build de producción; en caso contrario se mide el bundler (§1).
6. **React Profiler.** Componentes que se repintan sin necesidad. Origen de las tres props reconstruidas en cada render y del comparador de `React.memo` incompleto, junto con el registro de qué prop cambiaba de identidad en cada fila. Ninguno de los dos es visible en lectura de código (§3).

Cuando las herramientas estándar no bastan, instrumentación propia: `document.getAnimations()`
para el número real de animaciones activas y un `MutationObserver` para contar mutaciones del DOM
por interacción. Un conteo es estable entre ejecuciones y señala la causa; un tiempo absoluto
depende de la máquina y solo describe el síntoma. De ahí proceden las 952 transiciones
simultáneas y las 232 mutaciones al editar la primera celda.

Dos criterios transversales. Primero, hipótesis explícitas y verificación de todas: la lectura de
código produce candidatos, no evidencia, y las cuatro hipótesis iniciales se descartaron (§2).
Segundo, instrumento invariable entre ejecuciones: una modificación del script de carga entre la
medición previa y la posterior invalidó esa comparación.

En servidor, el orden es equivalente con otras herramientas: verificación de que se ejecuta el
escenario declarado (§6), medición de throughput y no de usuarios virtuales, que son la presión
aplicada y no el resultado (§7), aislamiento del recurso compartido mediante una prueba de una
sola variable para distinguir pool de CPU (§8), y solo entonces análisis del plan de ejecución
con `EXPLAIN (ANALYZE, BUFFERS)` (§9). El cierre es automatizado: una mejora sin test que la
defienda revierte sin aviso (§11).

---

## Parte 1. Navegador

### 1. Descarte de la fuga de memoria

Al repetir la captura de heap sobre la build de producción, los nodos desconectados no aparecen.
Los retenía la contabilidad del hot-module-replacement, no el código de la aplicación.

A partir de ese punto, todo el perfilado se hizo sobre la build desplegable.

### 2. Hipótesis iniciales y su verificación

Cuatro hipótesis formuladas por lectura del fuente. Ninguna se confirmó.

| Hipótesis | Verificación | Resultado |
|---|---|---|
| Los indicadores de estado parpadean: siete clases CSS declaran `animation: … infinite` | `document.getAnimations()` sobre la página en ejecución | 0 animaciones activas |
| Selectores `:hover` costosos, con cuatro `:not()` encadenados | Selector Stats de DevTools | 13,5 ms de 13.006 ms de recálculo de estilo (0,1%) |
| Un componente padre re-renderiza dos veces por interacción | Build sin `<StrictMode>` | Doble invocación exclusiva de desarrollo |
| Virtualizador defectuoso: monta 100 filas para 18 visibles | Recuento de filas en el DOM | El perfilado correspondía a una tabla distinta de la modificada |

La lectura del CSS confirma que una regla existe, pero no cuántos elementos alcanza.

### 3. Hallazgos y correcciones

1. **952 transiciones CSS simultáneas.** Una transición de `background-color` de 0,1 s declarada en la celda en lugar de la fila, multiplicada por 40 columnas. Trasladada a la fila. Origen del parpadeo reportado.
2. **7.332 invalidaciones de estilo desde un único selector.** El rayado y el hover se aplicaban a las celdas (`tr:hover td:not(.pinnedCell)`), de forma que el desplazamiento del cursor invalidaba las 40. Trasladado a la fila.
3. **`will-change` sobre elementos cuya propiedad animada no se compone.** Una capa de compositor por celda sin contenido que componer. Retirado.
4. **Indicador de progreso animando `width`**, con layout forzado en cada frame. Sustituido por `transform: scaleX()`.
5. **El getter del contenedor de scroll del virtualizador devolvía el contenedor incorrecto** y montaba 100 filas en lugar de 28. Corregido.
6. **Tres props reconstruidas en cada render**, que anulaban el `React.memo` de todas las filas. Localizadas registrando el cambio de identidad de cada prop por fila; no son visibles en lectura de código.
7. **El comparador de `React.memo` omitía el flag de selección.** La fila solo se repintaba cuando un render ajeno la forzaba.

El séptimo es un defecto de corrección dentro de una optimización de rendimiento. La
funcionalidad supera las pruebas manuales y solo se distingue de una lentitud ordinaria al
describir el síntoma con precisión: la fila no se repinta hasta que se hace scroll, y entonces
aparece actualizada. No es latencia, es pérdida de la notificación de cambio.

### 4. Medición antes y después, con el cursor como única variable

Experimento aislado: únicamente desplazamiento del cursor sobre la tabla, sin scroll, clics ni
escritura, antes y después. Las cifras son proporciones del perfil y no milisegundos absolutos,
porque las proporciones se mantienen entre ejecuciones en una máquina compartida.

| Métrica | Antes | Después |
|---|---|---|
| `Recalculate style` (self) | 45,7 % | 16,1 % |
| Rendering | 52,3 % | 36,2 % |
| Ocupación del hilo principal | 92,7 % | 85,3 % |
| `Event: animationiteration` | 35,0 % | no aparece |
| Animaciones simultáneas (pico) | 953 | 1 · 15 · 4 |

`Recalculate style` cambia además de forma: de 7,5% self / 42,1% total a 16,1% self ≈ 16,1%
total. En la medición previa era el efecto de las transiciones y la causa estaba aguas arriba; el
trabajo restante es propio.

Recuento de mutaciones del DOM por interacción, con un `MutationObserver` sobre las filas
montadas, para separar el trabajo legítimo del sobrante:

| Interacción | Filas afectadas | Mutaciones | Lectura |
|---|---|---|---|
| Marcar el check de una fila | 1 | 3 | Mínimo posible |
| Editar una celda con el grupo ya activo | 2 | 25 | Correcto: la celda que pierde el foco y la que lo gana |
| Editar la primera celda | 28 | 232 | Legítimo: el doble clic activa el grupo y todas las filas visibles pasan a editables |

El tercer caso presenta el perfil de una regresión. Para distinguirlo se aisló la variable:
activar el grupo primero y medir después la segunda celda. Sin ese paso, la optimización habría
eliminado un repintado que la funcionalidad requiere.

### 5. Revisión de un ADR propio: eliminación del Web Worker

Un ADR anterior, redactado por mí, exigía calcular la completitud por grupo en un Web Worker para
descargar el hilo principal. La decisión no tenía medición asociada. Medida, no aporta ganancia en
ningún tamaño de entrada.

El banco ejecuta el algoritmo de producción sobre 40 columnas y filas sintéticas con la forma de
las reales. Medianas de 40 muestras, por lotes: `performance.now()` tiene una resolución de unos
0,1 ms y una pasada única devuelve `0.000 ms`, interpretable como coste nulo.

| Filas | Hilo principal | Solo `structuredClone` | Worker persistente | Worker por cambio |
|---|---|---|---|---|
| 25 | 0,014 ms | 0,105 ms | 0,300 ms | 5,600 ms (412×) |
| 100 | 0,034 ms | 0,356 ms | 0,500 ms | 5,700 ms (170×) |
| 1.000 | 0,356 ms | 3,195 ms | 3,800 ms | 9,700 ms (27×) |
| 10.000 | 4,810 ms | 46,840 ms | 55,300 ms | 86,900 ms (18×) |

Con 10.000 filas el cálculo cuesta 4,810 ms y la serialización del payload, por sí sola, 46,840
ms. El envío a un worker no descarga ese coste del hilo principal, que paga el `structuredClone`
completo antes de ceder el control. Un worker resulta rentable cuando el cálculo cuesta más que
la transferencia. Una pasada lineal única no cumple esa condición: la clonación recorre la misma
estructura y además reserva memoria para copiarla, mientras que el algoritmo solo lee. Los casos
rentables son el trabajo superlineal, los datos que ya residen en el worker y los payloads
transferibles sin clonación: ordenaciones grandes, joins, parseo de ficheros, criptografía.

Como referencia de escala, un frame son 16,7 ms y 0,014 ms equivale al 0,08% de uno.

El ADR pasó de "usar un worker" a "solo si el cálculo cuesta más que la transferencia", con las
dos cifras obligatorias antes de mover trabajo. Los ficheros del worker se eliminaron.

---

## Parte 2. Servidor y base de datos

### 6. Pruebas de carga: el escenario que no se ejecutaba

Los escenarios declarados combinan gestos reales (listar, filtrar, editar, exportar, abrir
historial) en las proporciones de producción. Las primeras ejecuciones daban resultados
favorables y la salida nombraba un escenario `default` que el script no define.

`K6_VUS`, `K6_DURATION` y `K6_ITERATIONS` son opciones propias de k6, también cuando se pasan
como `-e K6_VUS=300`, que es la sintaxis para entregar una variable al script. k6 las consume,
descarta los escenarios declarados y ejecuta uno plano. La combinación de gestos no llegó a
ejecutarse. Una medición del elemento equivocado es peor que la ausencia de medición, porque
aporta una cifra.

El primer diagnóstico fue incorrecto: atribuí el problema a una variable de entorno perdida en la
máquina, cuando el origen era el flag de línea de comandos. Corregido al comprobar el
sospechoso más directo.

Los scripts pasaron a leer `LOAD_VUS` y `LOAD_DURATION`, que k6 no reclama, y a verificar que el
escenario en ejecución es el esperado, con un fallo explícito que nombra la causa probable.

### 7. Punto de saturación

Medición de throughput en peticiones por segundo. Los usuarios virtuales son la presión aplicada,
no un resultado.

| Carga | Resultado |
|---|---|
| 150 concurrentes | Todos los umbrales en verde salvo el p95 del export (539 ms) |
| 300 concurrentes | Los seis umbrales superados, 187 req/s |

El codo se sitúa entre ambos puntos: a partir de ahí el throughput se aplana y la latencia sube.
La degradación afecta a todos los gestos a la vez, lo que descarta una consulta concreta y apunta
a un recurso compartido: el pool de conexiones o la CPU. El dimensionado de producción se decidió
con esta cifra.

### 8. Pool de conexiones frente a CPU

La opción inmediata es ampliar el pool, por ser un cambio de una línea. Prueba controlada:
modificar el tamaño del pool, sin ningún otro cambio, y repetir la medición. El pool aportaba una
mejora real, pero el factor dominante era la CPU: cuatro vCPU constituyen el techo y ningún
ajuste del pool lo desplaza.

La conclusión entregada al negocio fue ampliar núcleos, en lugar de facturar un ajuste de
parámetros que no era la restricción.

### 9. Optimización en base de datos

**Guardado, de 730 ms a 113 ms.** Existía trabajo por fila ejecutándose en cada guardado.
Eliminado, no paralelizado.

**Histograma de historial, 477.000 filas leídas por petición.** Recalculaba en cada petición unas
cifras ya inmutables. `EXPLAIN (ANALYZE, BUFFERS)` sobre la ventana de 30 días:

```
Parallel Index Scan using idx_..._timestamp
  rows=159009 loops=3        <- 477.027 filas leídas
Workers Launched: 2
Execution Time: 283 ms
```

283 ms aislados resultan asumibles. El plan explica por qué no lo son: cada petición reclutaba
tres de las cuatro vCPU, y con diez usuarios concurrentes la mediana subía a 934 ms. El
paralelismo que beneficia a un usuario es el que degrada a diez.

La tabla solo recibe altas y se sella siempre con la hora actual, de modo que 29 de los 30 días
del gráfico son inmutables. La consulta pasa a leer un resumen diario de unas decenas de filas.
Dos decisiones de implementación, ambas con una alternativa aparente y errónea:

- **Tarea programada, no trigger.** Mantener el resumen desde el trigger de auditoría cargaría el coste a cada guardado, y el guardado acababa de bajar de 730 ms precisamente por retirarle trabajo. El camino de escritura desconoce la existencia del resumen.
- **Recálculo de dos días completos, sin marca de agua por `id`.** Una secuencia no garantiza el orden de commit: la transacción con el id 100 puede confirmar antes que la del 99, y una marca de agua sobre `id` omitiría esa fila de forma permanente. El recálculo por día es idempotente.

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

232 filas frente a 477.000, y la planificación de la consulta cuesta 45 veces más que su
ejecución: medio milisegundo es el presupuesto completo.

No se implementó. Un resumen precalculado no aportaba ganancia y añadía una tabla, una tarea
programada y un minuto de desfase. La medición y el umbral de volumen que invertiría la decisión
quedaron documentados en el código, para que el razonamiento sea recuperable y no solo la
conclusión.

---

## 11. Mecanismos que quedaron en el repositorio

Un directorio `qa/` con scripts de carga ejecutables, un banco de microbenchmarks y write-ups de
medición fechados, uno por sesión, con las cifras que justificaron cada cambio. El método reside
en el ADR que lo decidió; la carpeta guarda lo ejecutable y sus resultados.

Los planes de ejecución convertidos en tests. La eliminación accidental de un índice es invisible
hasta que producción se degrada, de modo que los tests corren contra un contenedor PostGIS real y
verifican con `EXPLAIN` que cada consulta crítica sigue usando el índice para el que se diseñó.
La eliminación de un índice deja el pull request en rojo. El mecanismo es genérico por tabla, no
específico de la primera, porque la segunda lo necesitaba antes de estar escrita. Su construcción
y la trampa que presenta la implementación evidente están en el
[write-up del pipeline](ci-pipeline-what-blocks.md#4-tests-sobre-los-planes-de-ejecución).

El trabajo de navegador se aplicó después a una segunda tabla (historial de auditoría, reversión
de cambios y los componentes compartidos resultantes de repetir el proceso), de forma deliberada
una vez la primera estaba correcta. Esa segunda pasada costó días en lugar de semanas porque los
mecanismos ya existían y solo las mediciones eran nuevas.

## 12. Defecto introducido durante el trabajo

La incorporación de la IP de cliente al contexto de auditoría era aparentemente inocua. Sin
embargo, `current_setting('x', true)` devuelve cadena vacía, no NULL, una vez fijada en la
sesión, de forma que sobre conexiones de pool la cadena de `COALESCE` almacenaba `''` de forma
permanente. Lo detectó un test escrito para ese mismo cambio, antes de publicarlo, y por eso es
una nota al pie y no un incidente de calidad de datos.

Regla generalizable: en una conexión de pool, el estado de sesión no es estado limpio.

## Errores de instrumentación detectados

- `set scrollTop` ocupando el 36% del perfil correspondía al script de carga propio, no a la aplicación.
- Los dobles renders en desarrollo procedían de `<StrictMode>` y se interpretaron inicialmente como dobles renders reales.
- Un INP de 3 segundos medido contra el servidor de desarrollo, sin valor para la build desplegable.
- Modificación del script de carga entre una ejecución posterior y una previa, que invalidó esa comparación. Repetida con el instrumento invariable.
- k6 consumiendo las variables y ejecutando un escenario no declarado ([§6](#6-pruebas-de-carga-el-escenario-que-no-se-ejecutaba)).

## Puntos abiertos

- `Commit`, `Hit test` y `Layerize` suponen ahora el 55% del perfil de hover: composición y pintado, no cálculo de estilo. Rendimientos decrecientes, sin planificación mientras no haya impacto reportado.
- Un INP de 232 ms en la apertura del editor de celda, no atribuible a React. Corresponde a una ida y vuelta de red para comprobación de permisos previa a la edición. Se trata en otra capa.

## Herramientas

Chrome DevTools (capturas de heap y vista de comparación, panel de Performance con self y total,
Selector Stats, Live Metrics/INP) · `document.getAnimations()` · `MutationObserver` · React
DevTools Profiler · scripts de carga y banco de microbenchmarks propios · k6 (escenarios propios,
umbrales como contrato, `Trend`/`Rate`/`Counter`) · `EXPLAIN (ANALYZE, BUFFERS)` de PostgreSQL ·
Testcontainers y Flyway para los tests de plan · selección por tag de JUnit para mantenerlos
fuera de la build rápida · GitHub Actions · ADRs
