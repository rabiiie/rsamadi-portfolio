# Rendimiento medido — el navegador y el servidor del mismo problema

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente.

## Resumen

El módulo de seguimiento de la plataforma FTTH es un conjunto de tablas editables: cientos de
miles de registros, más de 40 columnas, edición en línea, agrupación, filtros, ordenación,
columnas fijas y bloqueo optimista. Hoy 50-100 usuarios concurrentes; el objetivo son 500.

Nadie se había quejado. Quise saber qué aguantaba antes de que lo descubriera un cliente, y el
negocio tenía además una pregunta sin respuesta medida: cuánta gente puede trabajar a la vez y
qué máquina hace falta. La de producción tenía cuatro vCPU sin ninguna medición detrás.

Perfilé e instrumenté las dos mitades del mismo problema, el navegador y la base de datos. Quité
los cuellos de botella de repintado y las consultas caras, y dimensioné el hardware de producción
con una prueba de carga en vez de con una estimación.

## Resultados

- **Repintado.** Pasar el ratón por la tabla: el recálculo de estilo baja del 45,7% al 16,1% del perfil, y la ocupación del hilo principal del 92,7% al 85,3%.
- **Escritura.** Guardar una fila: de 730 ms a 113 ms, quitando el trabajo por fila que se ejecutaba en cada guardado.
- **Base de datos.** El histograma del historial pasa de leer 477.000 filas en cada petición a un resumen diario de unas decenas.
- **Capacidad.** k6 en verde a 150 usuarios concurrentes, y el punto de saturación exacto a 300, con 187 req/s. Con esa cifra se dimensionó la máquina de producción, y la conclusión fue comprar núcleos y no ajustar parámetros.
- **Un Web Worker borrado**, y el ADR que lo exigía —escrito por mí— corregido: medir dijo que serializar el payload con `structuredClone` cuesta unas diez veces más que el cálculo que iba a mover, en los cuatro tamaños probados.
- **Un resumen precalculado que no llegué a construir**, porque medirlo primero dijo que no ganaba nada.

## Decisiones de ingeniería

- **Navegador.** Las 952 transiciones CSS simultáneas venían de declararlas en la celda y no en la fila, con 40 columnas multiplicando. Y el comparador de `React.memo` de la fila se dejaba fuera el flag de selección, que es un fallo de corrección disfrazado de lentitud.
- **Base de datos.** El resumen diario lo mantiene un job programado y no el trigger de auditoría, para no devolverle al guardado el coste que le acababa de quitar; y recalcula días enteros en vez de avanzar una marca de agua por `id`, porque una secuencia no garantiza el orden de commit.
- **CI.** Los planes de ejecución son tests: corren contra un contenedor PostGIS real y comprueban con `EXPLAIN (ANALYZE, BUFFERS)` que cada consulta crítica sigue usando su índice. Si alguien borra uno, el pull request se pone en rojo.

**Herramientas.** Chrome DevTools · React DevTools Profiler · `EXPLAIN (ANALYZE, BUFFERS)` de
PostgreSQL · k6 · Testcontainers · JUnit 5 · GitHub Actions. El detalle está
[al final](#herramientas).

---

Lo que sigue es el desglose: por dónde se empieza un análisis así, y qué salió en cada paso.

## Por dónde se empieza

**Antes que ninguna pestaña, la build.** Hay que abrir la aplicación compilada como se despliega,
no `npm run dev`. En desarrollo React va sin minificar, corre comprobaciones extra y
`<StrictMode>` invoca cada render dos veces, así que todo lo que midas ahí incluye trabajo que el
usuario nunca paga. No es un detalle de rigor: la sospecha de fuga de memoria del §1 se cayó aquí,
y un INP de 3 segundos que llegué a apuntar estaba medido contra el servidor de desarrollo.

Después, las pestañas de DevTools por orden de coste, de la más barata a la más cara:

1. **Consola.** Un minuto. Los errores y los avisos de React salen ahí y a veces la respuesta ya está puesta. Aquí salió limpia, que también es información: descarta el bucle de renders y el aviso que se repite en cada pasada.
2. **Red.** Qué peticiones dispara cada gesto, cuántas, de qué tamaño y si van en cascada. Es donde aparece el trabajo que no es del navegador. De aquí salió que abrir el editor de una celda cuesta 232 ms y que **no es React**: es una ida y vuelta pidiendo permisos antes de dejar editar. Ese sigue abierto a propósito, porque se arregla en otra capa.
3. **Rendimiento.** Grabar **una sola interacción aislada** —solo mover el ratón, sin scroll y sin clics— y guardar esa grabación como línea base antes de tocar nada. Es la pestaña donde hay que leer *self* frente a *total*: lo que sale caro puede ser la víctima y no la causa. El recálculo de estilo lo era, y el arreglo estaba aguas arriba, en las transiciones (§4).
4. **Selector Stats**, dentro de esa misma grabación. Dice cuánto cuesta cada selector CSS y a cuántos elementos alcanza, que es justo lo que leer la hoja de estilos no dice. De aquí salieron las 7.332 invalidaciones desde un solo selector (§3).
5. **Memoria.** Dos capturas de heap y la vista de comparación, para la sospecha de fuga. Sobre la build de producción, o mides el bundler (§1).
6. **React Profiler.** Qué componentes se repintan y cuáles no deberían. Las tres props reconstruidas en cada render y el comparador de `React.memo` incompleto salieron de aquí, y de registrar qué prop cambiaba de identidad en cada fila; leyendo el código no se ven (§3).

Cuando las pestañas se quedan cortas, **conteos propios**: `document.getAnimations()` para saber
cuántas animaciones hay vivas de verdad, y un `MutationObserver` para contar mutaciones del DOM
por interacción. Un conteo es estable entre ejecuciones y apunta a la causa; un milisegundo varía
con la máquina y solo describe el síntoma. De ahí salieron las 952 transiciones simultáneas y las
232 mutaciones al editar la primera celda.

Dos reglas que atraviesan todo lo anterior. **Hipótesis primero, y comprobarlas todas**: leer el
código produce candidatos, no pruebas, y escribí cuatro que fallaron las cuatro (§2). Y **repetir
con el instrumento quieto**: la vez que edité el script de carga entre la ejecución "antes" y la
"después" tuve que tirar la comparación.

**En el servidor el orden es el mismo con otras herramientas.** Comprobar que se ejecuta el
escenario que escribí (§6); medir throughput y no usuarios virtuales, que son la presión y no el
resultado (§7); separar los recursos compartidos con una prueba de una sola variable, para saber
si manda el pool o la CPU (§8); y solo entonces bajar al plan de la query con
`EXPLAIN (ANALYZE, BUFFERS)` (§9). Y cerrar con algo que se ejecute solo: una mejora que no está
defendida por un test vuelve al estado anterior sin que nadie se entere (§11).

---

## Parte 1 · El navegador

### 1. La fuga no era una fuga

Repetí la captura de heap sobre la build de producción y los nodos desconectados no estaban. Los
retenía la contabilidad del hot-module-replacement, no el código de la aplicación.

Desde ahí perfilé siempre la build que se despliega. React en desarrollo va sin minificar, corre
comprobaciones extra, y `<StrictMode>` invoca cada render dos veces.

### 2. Cuatro hipótesis, y lo que dijo medirlas

Escribí cuatro hipótesis leyendo el fuente y comprobé las cuatro. Ninguna se sostuvo.

| Hipótesis | Cómo la comprobé | Resultado |
|---|---|---|
| Los indicadores de estado parpadean — siete clases CSS declaran `animation: … infinite` | `document.getAnimations()` en la página viva | 0 en ejecución |
| Selectores `:hover` caros, con cuatro `:not()` encadenados | Selector Stats de DevTools | 13,5 ms de 13.006 ms de recálculo de estilo — el 0,1% |
| Un componente padre re-renderiza dos veces por interacción | Build sin `<StrictMode>` | Doble invocación solo en desarrollo |
| El virtualizador está roto — monta 100 filas para 18 visibles | Contar filas en el DOM | Estaba perfilando una tabla distinta de la que había tocado |

Leer el CSS dice que una regla existe. No dice a cuántos elementos alcanza.

### 3. Lo que encontraron las mediciones

Siete cosas, con lo que hice en cada una:

1. **952 transiciones CSS simultáneas.** Una transición de `background-color` de 0,1 s estaba declarada en la **celda** y no en la **fila**, y con 40 columnas se multiplica por 40. Movida a la fila. El "parpadeo" que reportaban era esto.
2. **7.332 invalidaciones de estilo desde un solo selector.** El rayado y el hover se aplicaban a las celdas (`tr:hover td:not(.pinnedCell)`), así que pasar el ratón invalidaba las 40. Movido a la fila.
3. **`will-change` sobre elementos cuya propiedad animada no se compone.** Una capa de compositor por celda sin nada que componer. Retirado.
4. **Un indicador de progreso animando `width`**, que fuerza layout en cada frame. Sustituido por `transform: scaleX()`.
5. **El getter del contenedor de scroll del virtualizador devolvía el contenedor equivocado**, y montaba 100 filas en vez de 28. Corregido.
6. **Tres props reconstruidas en cada render**, que anulaban el `React.memo` de todas las filas. Las encontré registrando qué prop cambiaba de identidad en cada fila; leyendo el código no se ven.
7. **El comparador de `React.memo` se dejaba fuera el flag de selección.** La fila solo se repintaba cuando algo ajeno forzaba un render.

El séptimo es un fallo de corrección escondido dentro de una optimización de rendimiento: la
funcionalidad pasaba las pruebas a mano, y de una lentitud cualquiera solo se distingue al
describir el síntoma con precisión. La fila no se repinta *hasta* que haces scroll, y entonces
aparece al día. Eso no es que vaya lenta: es que no se entera.

### 4. Antes y después, con el ratón como única variable

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

Conté también las mutaciones del DOM por interacción, con un `MutationObserver` sobre las filas
montadas, para saber si lo que quedaba era trabajo legítimo:

| Interacción | Filas tocadas | Mutaciones | Lectura |
|---|---|---|---|
| Marcar el check de una fila | 1 | 3 | El mínimo posible |
| Editar una celda, con el grupo ya activo | 2 | 25 | Correcto: la celda que pierde el foco y la que lo gana |
| Editar la **primera** celda | 28 | 232 | Legítimo: el doble clic activa el grupo y todas las filas visibles pasan a editables |

La tercera parecía una regresión. Para distinguirlo aislé la variable: activar el grupo primero y
**después** medir la segunda celda. Sin ese paso habría "optimizado" un repintado que la
funcionalidad necesita.

### 5. Medí un ADR mío y lo borré

Un ADR que había escrito yo exigía calcular la completitud por grupo en un Web Worker, para
quitar ese trabajo del hilo principal. Nunca se había medido. Lo medí y no gana en ningún tamaño.

El banco corre el algoritmo de producción sobre 40 columnas y filas sintéticas con la forma de
las reales, medianas de 40 muestras, por lotes: `performance.now()` está limitado a ~0,1 ms y
medir una sola pasada devuelve `0.000 ms`, que se lee como "gratis".

| Filas | Hilo principal | Solo `structuredClone` | Worker persistente | Worker por cambio |
|---|---|---|---|---|
| 25 | 0,014 ms | 0,105 ms | 0,300 ms | 5,600 ms (412×) |
| 100 | 0,034 ms | 0,356 ms | 0,500 ms | 5,700 ms (170×) |
| 1.000 | 0,356 ms | 3,195 ms | 3,800 ms | 9,700 ms (27×) |
| 10.000 | 4,810 ms | 46,840 ms | 55,300 ms | 86,900 ms (18×) |

Con 10.000 filas, calcular cuesta 4,810 ms y **solo serializar el payload para mandarlo** cuesta
46,840 ms. Mandar datos a un worker no quita ese coste del hilo principal: el hilo principal paga
el `structuredClone` entero antes de soltar. Un worker sale a cuenta cuando calcular cuesta más
que transferir, y una pasada lineal única no lo cumple, porque clonar recorre la misma estructura
y además reserva memoria para copiarla, mientras que el algoritmo solo lee. Los workers valen
para trabajo superlineal, para datos que ya viven dentro del worker, o para payloads
transferibles sin clonar: ordenaciones grandes, joins, parseo de ficheros, criptografía.

Para dar escala: un frame son 16,7 ms, y 0,014 ms es el 0,08% de uno.

Cambié el ADR de *"usar un worker"* a *"solo si calcular cuesta más que transferir"*, con las dos
cifras obligatorias antes de mover nada, y borré los ficheros del worker.

---

## Parte 2 · El servidor

### 6. Las pruebas de carga, y el escenario que nunca se ejecutó

Los escenarios que escribí mezclan gestos reales —listar, filtrar, editar, exportar, abrir
historial— en las proporciones de producción. Las primeras ejecuciones daban cifras estupendas y
la salida nombraba un escenario `default` que el script no define.

`K6_VUS`, `K6_DURATION` y `K6_ITERATIONS` **son opciones propias de k6**, también cuando se pasan
como `-e K6_VUS=300`, que es la sintaxis de pasarle una variable a tu script. k6 se las queda,
descarta los escenarios declarados y ejecuta uno plano. La mezcla de gestos no se ejecutó nunca,
y una medición de la cosa equivocada es peor que no medir, porque viene con un número puesto.

Mi primer diagnóstico fue incorrecto: culpé a una variable de entorno perdida en la máquina,
cuando era el flag de la línea de comandos. Lo corregí al ver que el flag era el sospechoso
obvio.

Los scripts pasaron a leer `LOAD_VUS` y `LOAD_DURATION`, que k6 no reclama, y a comprobar que el
escenario en ejecución es el esperado, fallando con un error que nombra la causa probable.

### 7. El punto de saturación

Medí throughput en peticiones por segundo. Los usuarios virtuales son la presión que se aplica,
no un resultado.

| Carga | Resultado |
|---|---|
| 150 concurrentes | Todos los umbrales en verde salvo el p95 del export (539 ms) |
| 300 concurrentes | Los seis umbrales cruzados, 187 req/s |

El codo está entre los dos: a partir de ahí el throughput se aplana mientras la latencia sube. Y
degradaron todos los gestos a la vez, lo que descarta una query concreta y apunta a un recurso
compartido: el pool de conexiones o la CPU. Con esa cifra se dimensionó la máquina de producción.

### 8. El pool o la CPU

Lo tentador es agrandar el pool, porque es cambiar una línea. Hice la prueba controlada: cambiar
el tamaño del pool, no tocar nada más, volver a medir. El pool aportaba algo real, pero la CPU
dominaba: cuatro vCPU era el techo y ningún ajuste del pool lo mueve.

La conclusión le dijo al negocio que comprara núcleos, en vez de pagarme por ajustar unos
parámetros que no eran la restricción.

### 9. Lo que estaba haciendo la base de datos

**El guardado, de 730 ms a 113 ms.** Había trabajo por fila ejecutándose en cada guardado. Lo
quité, en vez de paralelizarlo.

**El histograma de historial, 477.000 filas leídas por petición.** Recalculaba en cada petición
unas cifras que ya no podían cambiar. `EXPLAIN (ANALYZE, BUFFERS)` sobre la ventana de 30 días:

```
Parallel Index Scan using idx_..._timestamp
  rows=159009 loops=3        <- 477.027 filas leídas
Workers Launched: 2
Execution Time: 283 ms
```

283 ms sueltos parecen asumibles. El plan explica por qué no lo son: cada petición reclutaba
**tres de las cuatro vCPU**, y con diez usuarios concurrentes la mediana se fue a 934 ms. El
paralelismo que ayuda a un usuario es lo que hunde a diez.

La tabla solo recibe altas y siempre se sella con la hora actual, así que de los 30 días del
gráfico 29 son inmutables. Ahora lee un resumen diario de unas pocas decenas de filas. Dos
decisiones de implementación, las dos con una alternativa tentadora y equivocada:

- **Un job programado, no un trigger.** Mantener el resumen desde el trigger de auditoría le cobraría el coste a cada guardado, y el guardado acababa de bajar de 730 ms precisamente por quitarle trabajo. El camino de escritura no sabe que el resumen existe.
- **Recalcular dos días enteros, sin marca de agua por `id`.** Una secuencia no garantiza el orden de commit: la transacción con el id 100 puede confirmar antes que la del 99, y una marca de agua sobre `id` se saltaría esa fila para siempre. Recalcular por día es idempotente.

También: paginación por keyset en vez de `OFFSET`, e índices GIN de trigramas para las búsquedas
de texto que estaban haciendo scan.

### 10. El resumen que decidí no construir

La propuesta era copiar ese mecanismo a la segunda tabla. Lo medí antes de escribirlo:

```
Index Only Scan
  rows=232          <- 11.232 filas en toda la tabla
Planning Time:  25.137 ms
Execution Time:  0.560 ms
```

232 filas frente a 477.000, y **planificar la query cuesta 45 veces más que ejecutarla**: medio
milisegundo es el presupuesto entero.

No lo construí. Un resumen precalculado no podía ganar nada y habría añadido una tabla, un job
programado y un minuto de desfase. La medición y el umbral de volumen que invertiría la decisión
quedaron escritos en el código, para que el siguiente herede el razonamiento y no solo la
conclusión.

---

## 11. Lo que quedó montado

Un directorio `qa/` con scripts de carga ejecutables, un banco de microbenchmarks y write-ups de
medición fechados, uno por sesión, con las cifras que justificaron cada cambio. El método vive en
el ADR que lo decidió; la carpeta guarda lo que se ejecuta y lo que produjo.

Los planes de ejecución convertidos en tests. Un índice borrado sin querer es invisible hasta que
producción va lenta, así que los tests corren contra un contenedor PostGIS real y comprueban con
`EXPLAIN` que cada query crítica sigue usando el índice para el que se diseñó. Si alguien borra
uno, el pull request se pone en rojo. El mecanismo es por tabla, no específico de la primera,
porque la segunda lo necesitaba antes de estar escrita. Cómo están construidos, y la trampa que
tiene la forma obvia de escribirlos, está en el
[write-up del pipeline](ci-pipeline-what-blocks.md#4-tests-sobre-los-planes-de-ejecución).

El trabajo del navegador se aplicó después a una segunda tabla —historial de auditoría, revertir
cambios, y los componentes compartidos que salieron de hacerlo dos veces—, a propósito después de
que la primera estuviera correcta. Esa segunda pasada costó días y no semanas porque los
mecanismos ya existían y solo eran nuevas las mediciones.

## 12. Un fallo que introduje yo en este trabajo

Añadir la IP del cliente al contexto de auditoría parecía limpio. Pero `current_setting('x', true)`
devuelve una cadena vacía, no NULL, en cuanto se ha fijado una vez en la sesión, así que sobre
conexiones de un pool la cadena de `COALESCE` guardaba `''` para siempre. Lo cazó un test que
escribí para ese mismo cambio, antes de publicarlo, y por eso es una nota al pie y no un
incidente de calidad de datos.

Lo generalizable: **en una conexión de pool, el estado de sesión no es estado fresco.**

## Errores de instrumento que me encontré

- `set scrollTop` ocupando el 36% del perfil era mi propio script de carga, no la aplicación.
- Los dobles renders en desarrollo eran `<StrictMode>`, y al principio los tomé por dobles renders de verdad.
- Un INP de 3 segundos medido contra el servidor de desarrollo, que no dice nada de la build que se despliega.
- Edité el script de carga entre una ejecución "después" y una "antes", lo que invalidó esa comparación. Repetida con el instrumento quieto.
- k6 quedándose las variables y ejecutando un escenario que yo no había escrito ([§6](#6-las-pruebas-de-carga-y-el-escenario-que-nunca-se-ejecutó)).

## Abierto, a propósito

- `Commit`, `Hit test` y `Layerize` son ahora el 55% del perfil de hover: composición y pintado, no cálculo de estilo. Rendimientos decrecientes; no está planificado mientras nadie se queje.
- Un INP de 232 ms al abrir el editor de celda no es React. Es una ida y vuelta de red pidiendo permisos antes de abrir la celda. Otra capa, se sigue aparte.

## Herramientas

Chrome DevTools (capturas de heap y vista de comparación, panel de Performance con self y total,
Selector Stats, Live Metrics/INP) · `document.getAnimations()` · `MutationObserver` · React
DevTools Profiler · scripts de carga y banco de microbenchmarks propios · k6 (escenarios propios,
umbrales como contrato, `Trend`/`Rate`/`Counter`) · `EXPLAIN (ANALYZE, BUFFERS)` de PostgreSQL ·
Testcontainers y Flyway para los tests de plan · selección por tag de JUnit para mantenerlos
fuera de la build rápida · GitHub Actions · ADRs
