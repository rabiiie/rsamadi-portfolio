# Capacidad y base de datos — pruebas de carga y trabajo de queries

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente.

## De qué va

Plataforma FTTH multicliente, con una tabla de referencia de unos cientos de miles de filas y cuarenta columnas. Y una pregunta del negocio que no tenía respuesta medida: cuánta gente puede trabajar a la vez, y qué máquina hace falta. La máquina de producción tenía cuatro vCPU, que es lo que le habían dado, sin ninguna medición detrás.

La otra mitad del mismo problema, la del navegador, está aparte: [rendimiento de la tabla](measured-performance-diagnosis.md).

---

## 1. El instrumento descartaba mis escenarios en silencio

**El síntoma.** Las primeras ejecuciones daban cifras estupendas. La salida nombraba un escenario llamado `default`, que el script no define. Los escenarios del script mezclan gestos reales (listar, filtrar, editar, exportar, abrir historial) en las proporciones de producción.

**La causa.** `K6_VUS`, `K6_DURATION` y `K6_ITERATIONS` **son opciones propias de k6**, también cuando las pasas como `-e K6_VUS=300`, que es exactamente la sintaxis de pasarle una variable a tu script. k6 se las queda, descarta todos los escenarios que el script declara y ejecuta uno plano por defecto. La mezcla de gestos no se ejecutó nunca.

Una medición de la cosa equivocada es peor que no medir, porque viene con un número puesto.

Y el primer diagnóstico que di fue incorrecto: culpé a una variable de entorno perdida en la máquina, cuando era el flag escrito en la línea de comandos. Lo corregí al ver que el flag era el sospechoso obvio.

**El arreglo.**

- Los scripts leen `LOAD_VUS` y `LOAD_DURATION`, que k6 no reclama.
- El script comprueba que el escenario en ejecución es el esperado y falla con un error que nombra la causa probable. Al siguiente le costará un minuto, no una tarde.

---

## 2. El punto de saturación

Lo que significa algo es el throughput en peticiones por segundo. Los usuarios virtuales son la presión que aplicas, no un resultado.

| Carga | Resultado |
|---|---|
| 150 concurrentes | Todos los umbrales en verde salvo el p95 del export (539 ms) |
| 300 concurrentes | Los seis umbrales cruzados, 187 req/s |

El codo está entre los dos: a partir de ahí el throughput se aplana mientras la latencia sube.

Y degradaron **todos los gestos a la vez**. Un solo endpoint lento apunta a una query; que se degrade todo al mismo tiempo apunta a un recurso compartido. Dos candidatos: el pool de conexiones o la CPU.

---

## 3. Separar el pool de la CPU

Lo tentador es agrandar el pool, porque es cambiar una línea. Hice la versión controlada: cambiar el tamaño del pool, no tocar nada más, volver a medir.

**El resultado.** El pool aportaba algo, y era real. La CPU dominaba. Cuatro vCPU era el techo y ninguna cantidad de tuneo del pool lo mueve.

Es una conclusión aburrida y es de las útiles: le dijo al negocio que comprara núcleos y no que me pagara por ajustar unos parámetros que no eran la restricción.

---

## 4. Lo que estaba haciendo la base de datos

**El guardado: de 730 ms a 113 ms.** Había trabajo por fila ejecutándose en cada guardado. El arreglo fue quitarlo, no paralelizarlo.

**El histograma de historial: 477.000 filas leídas por petición.** Recalculaba en cada petición unas cifras que ya no podían cambiar. `EXPLAIN (ANALYZE, BUFFERS)` sobre la ventana de 30 días:

```
Parallel Index Scan using idx_..._timestamp
  rows=159009 loops=3        <- 477.027 filas leídas
Workers Launched: 2
Execution Time: 283 ms
```

283 ms sueltos parecen asumibles. No lo son, y el plan explica por qué: cada petición reclutaba **tres de las cuatro vCPU**. Con diez usuarios concurrentes la mediana se fue a 934 ms. El paralelismo que ayuda a un usuario es justo lo que hunde a diez.

La tabla solo recibe altas y siempre se sella con la hora actual, así que de los 30 días del gráfico 29 son inmutables. Ahora lee un resumen diario de unas pocas decenas de filas.

Dos decisiones de implementación, las dos con una alternativa tentadora y equivocada:

- **Un job programado, no un trigger.** Mantener el resumen desde el trigger de auditoría le cobraría el coste a cada guardado, y el guardado acababa de bajar de 730 ms precisamente por quitarle trabajo. El camino de escritura no sabe que el resumen existe.
- **Recalcular dos días enteros, sin marca de agua por `id`.** Una secuencia no garantiza el orden de commit: la transacción que tiene el id 100 puede confirmar antes que la del 99, y una marca de agua sobre `id` se saltaría esa fila para siempre. Recalcular por día es idempotente: lo que llegue tarde entra en la siguiente pasada.

**También:** paginación por keyset y no por `OFFSET`, e índices GIN de trigramas para las búsquedas de texto que estaban haciendo scan.

---

## 5. Que no se vuelva a romper

Un índice borrado sin querer es invisible hasta que producción va lenta. Así que los planes de ejecución se convirtieron en tests: corren contra un contenedor PostGIS real y comprueban con `EXPLAIN` que cada query crítica sigue usando el índice para el que se diseñó. Si alguien borra uno, el pull request se pone en rojo.

El mecanismo es por tabla, no específico de la primera. Eso fue a propósito: la segunda tabla lo necesitaba antes de estar escrita.

Cómo están construidos esos tests, y la trampa que tiene la forma obvia de escribirlos, está en el [write-up del pipeline](ci-pipeline-what-blocks.md#4-tests-sobre-los-planes-de-ejecución).

---

## 6. El resumen que decidí no construir

**La propuesta.** Copiar el mecanismo del resumen a la segunda tabla.

**Medir primero.**

```
Index Only Scan
  rows=232          <- 11.232 filas en toda la tabla
Planning Time:  25.137 ms
Execution Time:  0.560 ms
```

232 filas, frente a 477.000 en la primera tabla. Y **planificar la query cuesta 45 veces más que ejecutarla**: en ese punto la query no es el problema, porque medio milisegundo es el presupuesto entero.

**La decisión.** No construirlo. Un resumen precalculado no podía ganar nada y habría añadido una tabla, un job programado y un minuto de desfase a cambio. La medición y el umbral de volumen que invertiría la decisión están escritos en el código, para que el siguiente herede el razonamiento y no solo la conclusión.

---

## 7. Un fallo que introduje yo en este trabajo

Añadir la IP del cliente al contexto de auditoría parecía limpio. Pero `current_setting('x', true)` devuelve una cadena vacía, no NULL, en cuanto se ha fijado una vez en la sesión. Sobre conexiones de un pool, la cadena de `COALESCE` guardaba `''` para siempre. Lo cazó un test que escribí para ese mismo cambio, antes de publicarlo, y por eso es una nota al pie y no un incidente de calidad de datos.

Lo generalizable: **en una conexión de pool, el estado de sesión no es estado fresco.**

## Herramientas

k6 (escenarios propios, umbrales como contrato, `Trend`/`Rate`/`Counter`) · `EXPLAIN (ANALYZE, BUFFERS)` de PostgreSQL · Testcontainers y Flyway para los tests de plan · selección por tag de JUnit para mantenerlos fuera de la build rápida · GitHub Actions · ADRs
