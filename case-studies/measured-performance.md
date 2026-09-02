# Rendimiento medido — el navegador y el servidor del mismo problema

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente.

## De qué va

El módulo de seguimiento de la plataforma FTTH es un conjunto de tablas editables: cientos de
miles de registros, más de 40 columnas, edición en línea, agrupación, filtros, ordenación,
columnas fijas y bloqueo optimista. Hoy 50-100 usuarios concurrentes; el objetivo son 500.

Los usuarios reportaban que pasar el ratón y hacer scroll iba pesado, que seleccionar una fila
tardaba un momento visible en pintarse, y una captura de heap hacía sospechar de una fuga de
memoria. En paralelo había una pregunta del negocio sin respuesta medida: cuánta gente puede
trabajar a la vez y qué máquina hace falta. La de producción tenía cuatro vCPU, que es lo que le
habían dado, sin ninguna medición detrás.

Son las dos mitades del mismo problema y se trabajaron con el mismo método, así que van juntas.

---

## Parte 1 · El navegador

### 1. La fuga no era una fuga

La captura de heap enseñaba nodos desconectados. Los retenía la contabilidad del
hot-module-replacement, no el código de la aplicación: en una build de producción no aparecían.

De ahí sale la regla que apliqué al resto del trabajo: perfilar la build que se despliega. React
en desarrollo va sin minificar, corre comprobaciones extra, y `<StrictMode>` invoca cada render
dos veces.

---

### 2. Cuatro hipótesis leyendo el código, y qué dijo medirlas

Saqué cuatro hipótesis leyendo el fuente. Las cuatro resultaron falsas.

| Hipótesis | Cómo la comprobé | Resultado |
|---|---|---|
| Los indicadores de estado parpadean — siete clases CSS declaran `animation: … infinite` | `document.getAnimations()` en la página viva | 0 en ejecución |
| Selectores `:hover` caros, con cuatro `:not()` encadenados | Selector Stats de DevTools | 13,5 ms de 13.006 ms de recálculo de estilo — el 0,1% |
| Un componente padre re-renderiza dos veces por interacción | Build sin `<StrictMode>` | Doble invocación solo en desarrollo; en producción no existe |
| El virtualizador está roto — monta 100 filas para 18 visibles | Contar filas en el DOM | Estaba perfilando una tabla distinta de la que había tocado |

Leer el CSS te dice que una regla existe. No te dice a cuántos elementos alcanza.

---

### 3. Lo que sí encontraron las mediciones

1. **952 transiciones CSS simultáneas.** Una transición de `background-color` de 0,1 s estaba declarada en la **celda** y no en la **fila**. Con 40 columnas se multiplica por 40. El "parpadeo" que reportaban era esto, no las animaciones.
2. **7.332 invalidaciones de estilo desde un solo selector.** El rayado de filas y el estado hover se aplicaban a las celdas (`tr:hover td:not(.pinnedCell)`), así que pasar el ratón por una fila invalidaba todas sus celdas. Movido a la fila.
3. **`will-change` sobre elementos cuya propiedad animada no se compone**: una capa de compositor por celda, sin nada que componer.
4. **Un indicador de progreso animando `width`**, que fuerza layout en cada frame. Sustituido por `transform: scaleX()`.
5. **El getter del contenedor de scroll del virtualizador devolvía el contenedor equivocado**, así que montaba 100 filas y no 28.
6. **Tres props reconstruidas en cada render**, que anulaban el `React.memo` de todas las filas. Lo encontré registrando qué prop cambiaba de identidad en cada fila, no leyendo el código.
7. **El comparador de `React.memo` se dejaba fuera el flag de selección.** La fila solo se repintaba cuando algo ajeno forzaba un render, y de ahí venía el "se pone al día cuando hago scroll" que describían.

El número 7 es un fallo de corrección escondido dentro de una optimización de rendimiento. La
funcionalidad pasaba las pruebas a mano. Lo que lo separó de una lentitud normal fue la
descripción exacta de un usuario: *"no se repinta hasta que hago scroll"*.

---

### 4. Resultados

Experimento aislado: **solo mover el ratón** sobre la tabla, sin scroll, sin clics, sin escribir,
antes y después. Doy proporciones del perfil y no milisegundos absolutos, porque las proporciones
se mantienen entre ejecuciones en una máquina compartida y los milisegundos no.

| Métrica | Antes | Después |
|---|---|---|
| `Recalculate style` (self) | 45,7 % | 16,1 % |
| Rendering | 52,3 % | 36,2 % |
| Ocupación del hilo principal | 92,7 % | 85,3 % |
| `Event: animationiteration` | 35,0 % | no aparece |
| Animaciones simultáneas (pico) | 953 | 1 · 15 · 4 |

`Recalculate style` además cambió de forma: de 7,5 % self / 42,1 % total a 16,1 % self ≈ 16,1 %
total. Antes era la víctima de las transiciones y el arreglo estaba aguas arriba; el trabajo que
queda ahora es suyo.

Mutaciones del DOM por interacción, contadas con un `MutationObserver` sobre las filas montadas:

| Interacción | Filas tocadas | Mutaciones | Lectura |
|---|---|---|---|
| Marcar el check de una fila | 1 | 3 | El mínimo posible |
| Editar una celda, con el grupo ya activo | 2 | 25 | Correcto: la celda que pierde el foco y la que lo gana |
| Editar la **primera** celda | 28 | 232 | Legítimo: el doble clic activa el grupo, y todas las filas visibles pasan a editables |

La tercera fila parecía una regresión y no lo era. Distinguirlo exigió aislar la variable:
activar el grupo primero y **después** medir la segunda celda. Sin ese paso habría "optimizado"
un repintado que la funcionalidad necesita.

---

### 5. Borrar un Web Worker que exigía una decisión mía

Un ADR que había escrito yo exigía calcular la completitud por grupo en un Web Worker, para
quitar ese trabajo del hilo principal. Nunca se había medido.

Lo medí con un banco que corre el algoritmo de producción sobre 40 columnas y filas sintéticas
con la forma de las reales, medianas de 40 muestras. Por lotes, porque `performance.now()` está
limitado a ~0,1 ms y medir una sola pasada devuelve `0.000 ms`, que se lee como "gratis".

| Filas | Hilo principal | Solo `structuredClone` | Worker persistente | Worker por cambio |
|---|---|---|---|---|
| 25 | 0,014 ms | 0,105 ms | 0,300 ms | 5,600 ms (412×) |
| 100 | 0,034 ms | 0,356 ms | 0,500 ms | 5,700 ms (170×) |
| 1.000 | 0,356 ms | 3,195 ms | 3,800 ms | 9,700 ms (27×) |
| 10.000 | 4,810 ms | 46,840 ms | 55,300 ms | 86,900 ms (18×) |

El worker no gana en ningún tamaño. Ni con 10.000 filas: calcular cuesta 4,810 ms y **solo
serializar el payload para mandarlo** cuesta 46,840 ms.

Mandar datos a un worker no quita ese coste del hilo principal: el hilo principal paga el
`structuredClone` entero antes de soltar. Un worker sale a cuenta cuando calcular cuesta más que
transferir, y una pasada lineal única no puede cumplir eso, porque clonar recorre la misma
estructura y encima reserva memoria para copiarla, mientras que el algoritmo solo lee. Los
workers valen para trabajo superlineal, para datos que ya viven dentro del worker, o para
payloads que se pueden transferir sin clonar: ordenar a lo bestia, joins, parsear ficheros,
criptografía.

Para dar escala: un frame son 16,7 ms. 0,014 ms es el 0,08% de uno.

Cambié el ADR de *"usar un worker"* a *"solo si calcular cuesta más que transferir"*, con las dos
cifras obligatorias antes de mover nada, y borré los ficheros del worker.

---

## Parte 2 · El servidor

### 6. El instrumento descartaba mis escenarios en silencio

Las primeras ejecuciones de carga daban cifras estupendas. La salida nombraba un escenario
llamado `default`, que el script no define: los suyos mezclan gestos reales —listar, filtrar,
editar, exportar, abrir historial— en las proporciones de producción.

`K6_VUS`, `K6_DURATION` y `K6_ITERATIONS` **son opciones propias de k6**, también cuando las
pasas como `-e K6_VUS=300`, que es exactamente la sintaxis de pasarle una variable a tu script.
k6 se las queda, descarta todos los escenarios que el script declara y ejecuta uno plano por
defecto. La mezcla de gestos no se ejecutó nunca.

Una medición de la cosa equivocada es peor que no medir, porque viene con un número puesto.

El primer diagnóstico que di fue incorrecto: culpé a una variable de entorno perdida en la
máquina, cuando era el flag escrito en la línea de comandos. Lo corregí al ver que el flag era el
sospechoso obvio.

Los scripts pasaron a leer `LOAD_VUS` y `LOAD_DURATION`, que k6 no reclama, y a comprobar que el
escenario en ejecución es el esperado, fallando con un error que nombra la causa probable. Al
siguiente le costará un minuto, no una tarde.

---

### 7. El punto de saturación

Lo que significa algo es el throughput en peticiones por segundo. Los usuarios virtuales son la
presión que aplicas, no un resultado.

| Carga | Resultado |
|---|---|
| 150 concurrentes | Todos los umbrales en verde salvo el p95 del export (539 ms) |
| 300 concurrentes | Los seis umbrales cruzados, 187 req/s |

El codo está entre los dos: a partir de ahí el throughput se aplana mientras la latencia sube.

Y degradaron **todos los gestos a la vez**. Un solo endpoint lento apunta a una query; que se
degrade todo al mismo tiempo apunta a un recurso compartido. Dos candidatos: el pool de
conexiones o la CPU.

---

### 8. Separar el pool de la CPU

Lo tentador es agrandar el pool, porque es cambiar una línea. Hice la versión controlada: cambiar
el tamaño del pool, no tocar nada más, volver a medir. El pool aportaba algo, y era real. La CPU
dominaba. Cuatro vCPU era el techo y ninguna cantidad de tuneo del pool lo mueve.

Es una conclusión aburrida y es de las útiles: le dijo al negocio que comprara núcleos y no que
me pagara por ajustar unos parámetros que no eran la restricción.

---

### 9. Lo que estaba haciendo la base de datos

**El guardado: de 730 ms a 113 ms.** Había trabajo por fila ejecutándose en cada guardado. El
arreglo fue quitarlo, no paralelizarlo.

**El histograma de historial: 477.000 filas leídas por petición.** Recalculaba en cada petición
unas cifras que ya no podían cambiar. `EXPLAIN (ANALYZE, BUFFERS)` sobre la ventana de 30 días:

```
Parallel Index Scan using idx_..._timestamp
  rows=159009 loops=3        <- 477.027 filas leídas
Workers Launched: 2
Execution Time: 283 ms
```

283 ms sueltos parecen asumibles. No lo son, y el plan explica por qué: cada petición reclutaba
**tres de las cuatro vCPU**. Con diez usuarios concurrentes la mediana se fue a 934 ms. El
paralelismo que ayuda a un usuario es justo lo que hunde a diez.

La tabla solo recibe altas y siempre se sella con la hora actual, así que de los 30 días del
gráfico 29 son inmutables. Ahora lee un resumen diario de unas pocas decenas de filas.

Dos decisiones de implementación, las dos con una alternativa tentadora y equivocada:

- **Un job programado, no un trigger.** Mantener el resumen desde el trigger de auditoría le cobraría el coste a cada guardado, y el guardado acababa de bajar de 730 ms precisamente por quitarle trabajo. El camino de escritura no sabe que el resumen existe.
- **Recalcular dos días enteros, sin marca de agua por `id`.** Una secuencia no garantiza el orden de commit: la transacción que tiene el id 100 puede confirmar antes que la del 99, y una marca de agua sobre `id` se saltaría esa fila para siempre. Recalcular por día es idempotente: lo que llegue tarde entra en la siguiente pasada.

También: paginación por keyset y no por `OFFSET`, e índices GIN de trigramas para las búsquedas
de texto que estaban haciendo scan.

---

### 10. El resumen que decidí no construir

La propuesta era copiar el mecanismo del resumen a la segunda tabla. Antes de escribirlo, medir:

```
Index Only Scan
  rows=232          <- 11.232 filas en toda la tabla
Planning Time:  25.137 ms
Execution Time:  0.560 ms
```

232 filas, frente a 477.000 en la primera tabla. Y **planificar la query cuesta 45 veces más que
ejecutarla**: en ese punto la query no es el problema, porque medio milisegundo es el presupuesto
entero.

No lo construí. Un resumen precalculado no podía ganar nada y habría añadido una tabla, un job
programado y un minuto de desfase a cambio. La medición y el umbral de volumen que invertiría la
decisión están escritos en el código, para que el siguiente herede el razonamiento y no solo la
conclusión.

---

## Lo que quedó de las dos mitades

### Qué se montó para que no se repita

Del lado del navegador, un directorio `qa/` con scripts ejecutables, un banco de microbenchmarks
y write-ups de medición fechados, uno por sesión, con las cifras que justificaron cada cambio. El
método vive en el ADR que lo decidió; la carpeta guarda solo lo que se ejecuta y lo que produjo.

Del lado del servidor, los planes de ejecución convertidos en tests. Un índice borrado sin querer
es invisible hasta que producción va lenta, así que los tests corren contra un contenedor PostGIS
real y comprueban con `EXPLAIN` que cada query crítica sigue usando el índice para el que se
diseñó. Si alguien borra uno, el pull request se pone en rojo. El mecanismo es por tabla y no
específico de la primera, porque la segunda lo necesitaba antes de estar escrita. Cómo están
construidos, y la trampa que tiene la forma obvia de escribirlos, está en el
[write-up del pipeline](ci-pipeline-what-blocks.md#4-tests-sobre-los-planes-de-ejecución).

El mismo tratamiento del navegador se aplicó después a una segunda tabla —historial de auditoría,
revertir cambios, y los componentes compartidos que salieron de hacerlo dos veces—, y a propósito
después de que la primera estuviera correcta. La segunda pasada costó días y no semanas porque
los mecanismos ya existían y solo eran nuevas las mediciones.

### Los instrumentos mintieron en las dos mitades

En el navegador: `set scrollTop` ocupando el 36% del perfil era mi propio script de carga; los
dobles renders eran `<StrictMode>` y al principio los tomé por dobles renders de verdad; un INP
de 3 segundos medido contra el servidor de desarrollo no dice nada de la build que se despliega;
y edité el script de carga entre una ejecución "después" y una "antes", lo que invalidó esa
comparación y obligó a repetirla con el instrumento quieto.

En el servidor: k6 quedándose las variables y ejecutando un escenario que yo no había escrito.

En las dos mitades el primer número que salió era falso, y en las dos el trabajo real empezó
cuando dejé de creerme la herramienta.

### Un fallo que introduje yo en este trabajo

Añadir la IP del cliente al contexto de auditoría parecía limpio. Pero `current_setting('x', true)`
devuelve una cadena vacía, no NULL, en cuanto se ha fijado una vez en la sesión. Sobre conexiones
de un pool, la cadena de `COALESCE` guardaba `''` para siempre. Lo cazó un test que escribí para
ese mismo cambio, antes de publicarlo, y por eso es una nota al pie y no un incidente de calidad
de datos.

Lo generalizable: **en una conexión de pool, el estado de sesión no es estado fresco.**

### Abierto, a propósito

- `Commit`, `Hit test` y `Layerize` son ahora el 55% del perfil de hover: composición y pintado, no cálculo de estilo. Rendimientos decrecientes; no está planificado mientras nadie se queje.
- Un INP de 232 ms al abrir el editor de celda **no es React**. Es una ida y vuelta de red pidiendo permisos antes de abrir la celda. Otra capa, se sigue aparte.

## Herramientas

Chrome DevTools (capturas de heap y vista de comparación, panel de Performance con self y total,
Selector Stats, Live Metrics/INP) · `document.getAnimations()` · `MutationObserver` · React
DevTools Profiler · scripts de carga y banco de microbenchmarks propios · k6 (escenarios propios,
umbrales como contrato, `Trend`/`Rate`/`Counter`) · `EXPLAIN (ANALYZE, BUFFERS)` de PostgreSQL ·
Testcontainers y Flyway para los tests de plan · selección por tag de JUnit para mantenerlos
fuera de la build rápida · GitHub Actions · ADRs
