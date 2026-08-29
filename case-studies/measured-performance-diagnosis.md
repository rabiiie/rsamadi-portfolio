# Rendimiento de una tabla de datos — diagnóstico en el navegador

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente.

## De qué va

El módulo de seguimiento de la plataforma FTTH es un conjunto de tablas editables: cientos de miles de registros, más de 40 columnas, edición en línea, agrupación, filtros, ordenación, columnas fijas y bloqueo optimista. Hoy 50-100 usuarios concurrentes; el objetivo son 500.

Lo que reportaban los usuarios: que pasar el ratón y hacer scroll iba pesado, que seleccionar una fila tardaba un momento visible en pintarse, y se sospechaba de una fuga de memoria en una captura de heap.

La otra mitad de este mismo problema, la del servidor, está aparte: [capacidad y base de datos](capacity-and-database-performance.md).

---

## 1. La fuga no era una fuga

**Lo que se veía.** Nodos desconectados en una captura de heap.

**Lo que era.** Esos nodos los retenía la contabilidad del hot-module-replacement, no el código de la aplicación. En una build de producción no aparecían.

**La regla que saqué.** Perfilar la build que se despliega. React en desarrollo va sin minificar, corre comprobaciones extra, y `<StrictMode>` invoca cada render dos veces — solo en desarrollo.

---

## 2. Cuatro hipótesis leyendo el código, y qué dijo medirlas

Saqué cuatro hipótesis leyendo el fuente. Las cuatro resultaron falsas.

| Hipótesis | Cómo la comprobé | Resultado |
|---|---|---|
| Los indicadores de estado parpadean — siete clases CSS declaran `animation: … infinite` | `document.getAnimations()` en la página viva | 0 en ejecución |
| Selectores `:hover` caros, con cuatro `:not()` encadenados | Selector Stats de DevTools | 13,5 ms de 13.006 ms de recálculo de estilo — el 0,1% |
| Un componente padre re-renderiza dos veces por interacción | Build sin `<StrictMode>` | Doble invocación solo en desarrollo; en producción no existe |
| El virtualizador está roto — monta 100 filas para 18 visibles | Contar filas en el DOM | Estaba perfilando una tabla distinta de la que había tocado |

Leer el CSS te dice que una regla existe. No te dice a cuántos elementos alcanza.

---

## 3. Lo que sí encontraron las mediciones

1. **952 transiciones CSS simultáneas.** Una transición de `background-color` de 0,1 s estaba declarada en la **celda** en vez de en la **fila**. Con 40 columnas se multiplica por 40. El "parpadeo" que reportaban era esto, no las animaciones.
2. **7.332 invalidaciones de estilo desde un solo selector.** El rayado de filas y el estado hover se aplicaban a las celdas (`tr:hover td:not(.pinnedCell)`), así que pasar el ratón por una fila invalidaba todas sus celdas. Movido a la fila.
3. **`will-change` sobre elementos cuya propiedad animada no se compone** — una capa de compositor por celda, sin nada que componer.
4. **Un indicador de progreso animando `width`**, que fuerza layout en cada frame. Sustituido por `transform: scaleX()`.
5. **El getter del contenedor de scroll del virtualizador devolvía el contenedor equivocado**, así que montaba 100 filas en vez de 28.
6. **Tres props reconstruidas en cada render**, que anulaban el `React.memo` de todas las filas. Lo encontré registrando qué prop cambiaba de identidad en cada fila, no leyendo el código.
7. **El comparador de `React.memo` se dejaba fuera el flag de selección.** La fila solo se repintaba cuando algo ajeno forzaba un render, y de ahí venía el "se pone al día cuando hago scroll" que describían.

El número 7 es un fallo de corrección escondido dentro de una optimización de rendimiento. La funcionalidad pasaba las pruebas a mano. Lo que lo separó de una lentitud normal fue la descripción exacta de un usuario: *"no se repinta hasta que hago scroll"*.

---

## 4. Resultados

Experimento aislado: **solo mover el ratón** sobre la tabla, sin scroll, sin clics, sin escribir, antes y después. Doy proporciones del perfil y no milisegundos absolutos, porque las proporciones se mantienen entre ejecuciones en una máquina compartida y los milisegundos no.

| Métrica | Antes | Después |
|---|---|---|
| `Recalculate style` (self) | 45,7 % | 16,1 % |
| Rendering | 52,3 % | 36,2 % |
| Ocupación del hilo principal | 92,7 % | 85,3 % |
| `Event: animationiteration` | 35,0 % | no aparece |
| Animaciones simultáneas (pico) | 953 | 1 · 15 · 4 |

`Recalculate style` además cambió de forma: de 7,5 % self / 42,1 % total a 16,1 % self ≈ 16,1 % total. Antes era la víctima de las transiciones y el arreglo estaba aguas arriba; el trabajo que queda ahora es suyo.

Mutaciones del DOM por interacción, contadas con un `MutationObserver` sobre las filas montadas:

| Interacción | Filas tocadas | Mutaciones | Lectura |
|---|---|---|---|
| Marcar el check de una fila | 1 | 3 | El mínimo posible |
| Editar una celda, con el grupo ya activo | 2 | 25 | Correcto: la celda que pierde el foco y la que lo gana |
| Editar la **primera** celda | 28 | 232 | Legítimo: el doble clic activa el grupo, y todas las filas visibles pasan a editables |

La tercera fila parecía una regresión y no lo era. Distinguirlo exigió aislar la variable: activar el grupo primero y **después** medir la segunda celda. Sin ese paso habría "optimizado" un repintado que la funcionalidad necesita.

---

## 5. Borrar un Web Worker que exigía una decisión mía

**El contexto.** Un ADR que había escrito yo exigía calcular la completitud por grupo en un Web Worker, para quitar ese trabajo del hilo principal. Nunca se había medido.

**Cómo lo medí.** Un banco con el algoritmo de producción, 40 columnas, filas sintéticas con la forma de las reales, medianas de 40 muestras. Por lotes, porque `performance.now()` está limitado a ~0,1 ms y medir una sola pasada devuelve `0.000 ms`, que se lee como "gratis".

| Filas | Hilo principal | Solo `structuredClone` | Worker persistente | Worker por cambio |
|---|---|---|---|---|
| 25 | 0,014 ms | 0,105 ms | 0,300 ms | 5,600 ms (412×) |
| 100 | 0,034 ms | 0,356 ms | 0,500 ms | 5,700 ms (170×) |
| 1.000 | 0,356 ms | 3,195 ms | 3,800 ms | 9,700 ms (27×) |
| 10.000 | 4,810 ms | 46,840 ms | 55,300 ms | 86,900 ms (18×) |

**El worker no gana en ningún tamaño.** Ni con 10.000 filas: calcular cuesta 4,810 ms y **solo serializar el payload para mandarlo** cuesta 46,840 ms.

Mandar datos a un worker no quita ese coste del hilo principal: el hilo principal paga el `structuredClone` entero antes de soltar. Un worker sale a cuenta cuando calcular cuesta más que transferir, y una pasada lineal única no puede cumplir eso — clonar recorre la misma estructura y encima reserva memoria para copiarla, mientras que el algoritmo solo lee. Los workers valen para trabajo superlineal, para datos que ya viven dentro del worker, o para payloads que se pueden transferir en vez de clonar: ordenar a lo bestia, joins, parsear ficheros, criptografía.

Para dar escala: un frame son 16,7 ms. 0,014 ms es el 0,08% de uno.

**Qué hice.** Cambié el ADR de *"usar un worker"* a *"solo si calcular cuesta más que transferir"*, con las dos cifras obligatorias antes de mover nada. Y borré los ficheros del worker.

---

## 6. Errores de instrumento que me encontré

- `set scrollTop` ocupando el 36% del perfil era mi propio script de carga, no la aplicación.
- Los dobles renders en desarrollo eran `<StrictMode>`, y al principio los tomé por dobles renders de verdad.
- Un INP de 3 segundos medido contra el servidor de desarrollo, que no dice nada de la build que se despliega.
- Edité el script de carga entre una ejecución "después" y una "antes", lo que invalidó esa comparación. Repetida con el instrumento quieto.

---

## 7. Qué quedó montado

Un directorio `qa/` con scripts ejecutables, un banco de microbenchmarks y write-ups de medición fechados, uno por sesión, con las cifras que justificaron cada cambio. El método vive en el ADR que lo decidió; la carpeta guarda solo lo que se ejecuta y lo que produjo.

El mismo tratamiento se aplicó después a una segunda tabla — historial de auditoría, revertir cambios, y los componentes compartidos que salieron de hacerlo dos veces —, y a propósito después de que la primera estuviera correcta. La segunda pasada costó días y no semanas porque los mecanismos ya existían y solo eran nuevas las mediciones.

---

## Abierto, a propósito

- `Commit`, `Hit test` y `Layerize` son ahora el 55% del perfil de hover: composición y pintado, no cálculo de estilo. Rendimientos decrecientes; no está planificado mientras nadie se queje.
- Un INP de 232 ms al abrir el editor de celda **no es React**. Es una ida y vuelta de red pidiendo permisos antes de abrir la celda. Otra capa, se sigue aparte.

## Herramientas

Chrome DevTools (capturas de heap y vista de comparación, panel de Performance con self y total, Selector Stats, Live Metrics/INP) · `document.getAnimations()` · `MutationObserver` · React DevTools Profiler · scripts de carga y banco de microbenchmarks propios · ADRs
