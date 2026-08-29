# Puntos de conversación — AppFibra

Si solo tengo 60 segundos para explicar lo que he construido:

Soy el único desarrollador de AppFibra, una plataforma SaaS para el despliegue de fibra FTTH en Alemania. Más de 200 municipios, más de 200.000 homes y tres clientes trabajando con ella en producción.

**Stack**: backend en Spring Boot, frontend en React, PostgreSQL/PostGIS para el dato espacial, FastAPI para la capa de agentes de IA, y Keycloak/OAuth2 para identidad.

**De qué soy responsable**: de todo, de punta a punta. Requisitos, arquitectura, pipelines de GIS, seguridad, agentes, despliegue, y mantenerlo en pie cuando algo se rompe en producción.

**Las partes de las que me gustaría hablar en una entrevista:**

- **GIS.** Vector tiles, resolvers de capa por cliente, tablas de estado de obra precalculadas.
- **Agentes de IA.** Sobre MCP, con una capa de tools semánticas y no text-to-SQL, y un circuito que convierte el feedback negativo en casos de evaluación, para no adivinar si un cambio de prompt ha servido de algo.
- **Modelo de seguridad.** Keycloak/OAuth2, RBAC, scopes de recurso hasta proyecto o ciudad, autenticación M2M entre los servicios Java y Python.
- **La columna de auditoría.** Detectar cuándo un sistema externo sobrescribe datos en silencio, con diff campo a campo y capacidad de revertir.
- **Rendimiento.** Cuatro hipótesis sacadas de leer el código, las cuatro falsas, y una decisión de arquitectura mía revertida por su primera medición.
- **Capacidad.** Cuántos usuarios concurrentes aguanta de verdad, contestado con una prueba de carga y un experimento controlado, no con una estimación. Y la segunda vez, la medición dijo que no construyera nada.
- **El pipeline de CI.** La pregunta de diseño no es qué escáneres ejecutar, sino cuáles pueden parar el trabajo.

**Por qué no es "otro CRUD más"**: dato espacial a escala, agentes de IA conectados a datos de operación reales y no a una demo, y un refuerzo de seguridad que tuvo que sobrevivir a una migración de identidad de verdad. Todo desplegado y funcionando, no sobre el papel.

---

## Si la conversación va a rendimiento

*[Write-up completo: [rendimiento de la tabla](../case-studies/measured-performance-diagnosis.md)]*

Las tablas de seguimiento manejan cientos de miles de registros con más de 40 columnas, y los usuarios decían que pasar el ratón y hacer scroll iba pesado.

Tenía cuatro hipótesis de leer el código. Las cuatro falsas. Los indicadores que parpadeaban, a los que culpé: `document.getAnimations()` decía que no había ninguna corriendo. Los selectores `:hover` caros: Selector Stats los dejó en el 0,1% del coste. El doble render: era StrictMode, solo en desarrollo. El virtualizador roto: estaba perfilando una tabla distinta de la que había tocado.

Lo que era de verdad: una transición de 0,1 segundos declarada en la **celda** y no en la **fila**, multiplicada por cuarenta columnas hasta 952 animaciones simultáneas. El recálculo de estilo bajó del 45,7% del perfil al 16,1%.

**Con lo que empezaría**: borré un Web Worker que exigía una decisión de arquitectura **escrita por mí**. Lo medí, y serializar los datos para mandarlos costaba entre 7 y 10 veces más que el propio cálculo, en todos los tamaños de 25 a 10.000 filas. Mandar datos a un worker no los quita del hilo principal: el hilo principal paga el clonado entero antes de soltar. Reescribí la decisión de "usar un worker" a "solo si calcular cuesta más que transferir", con las dos cifras obligatorias antes de mover nada más.

El hábito que hay debajo de todo esto: medir la variabilidad antes de medir la diferencia, y aislar una variable cada vez. Lo demás son herramientas.

## Si preguntan por escala, o "cuántos usuarios aguanta"

*[Write-up completo: [capacidad y base de datos](../case-studies/capacity-and-database-performance.md)]*

Nadie lo sabía, así que lo medí. Scripts de k6 que reproducen la mezcla real de gestos, con umbrales que tumban la ejecución. El codo está entre 150 y 300 usuarios concurrentes; a 300 se cruzaron los seis umbrales, a 187 req/s. Degradaron todos los gestos a la vez, lo que apuntaba a un recurso compartido y no a una query mala, y un experimento controlado separó el pool de conexiones (real, pequeño) de la CPU (dominante). Eso le dijo al negocio que comprara núcleos y no que me pagara por ajustar parámetros que no eran la restricción.

La mitad de base de datos: un guardado de 730 ms a 113 ms quitándole trabajo por fila, y un gráfico de historial que leía 477.000 filas en cada petición. Esa es la mejor historia: 283 ms suenan asumibles hasta que el plan enseña que cada petición reclutaba tres de las cuatro vCPU, que es por lo que con diez usuarios concurrentes la mediana era de 934 ms. **El paralelismo que ayuda a uno es lo que hunde a diez.**

**Con lo que empezaría aquí**: cuando llevé el mismo tratamiento a la segunda tabla, la medición dijo que no. 232 filas en la ventana, y planificar la query costaba cuarenta y cinco veces más que ejecutarla. Así que el resumen precalculado no se construyó, y en su lugar entró en el código el número y el umbral que invertiría esa decisión.

Y los planes son ahora tests: corren contra un contenedor PostGIS real y comprueban con `EXPLAIN` que cada query crítica sigue usando su índice. Planes en cada pull request, tiempos en ejecución programada: un plan es un hecho sobre la query, un tiempo es un hecho sobre la máquina, y comprobar tiempos en runners compartidos solo enseña a la gente a ignorar el pipeline.

## Si la conversación va a entrega y CI

*[Write-up completo: [pipeline de CI](../case-studies/ci-pipeline-what-blocks.md)]*

Un repositorio políglota (Java, React, dos servicios Python y el esquema PostGIS entero), conmigo como único desarrollador y usuarios reales en producción.

La regla con la que lo diseñé: **un pipeline maduro no se mide por cuántos escáneres ejecuta, sino por si cualquier commit de main se puede desplegar sin intervención manual.** Eso decidió el orden: la cobertura no significa nada antes de que la aplicación arranque en CI, y los tests de integración son imposibles antes de versionar el esquema, así que esas dos cosas fueron primero.

La distinción que de verdad me apetece discutir: **que los hallazgos sean informativos no es lo mismo que la herramienta sea informativa.** El job de SAST no falla cuando encuentra cosas, pero sí falla cuando el escaneo no llega a ejecutarse. Con un `continue-on-error` general los dos casos se ven igual, rojo e ignorado, y un escáner roto informaría de que "pasamos SAST" mientras no pasa nada. Un job que sale rojo pase lo que pase enseña a ignorar CI, que es peor que no tenerlo.

Lo mismo con las excepciones de dependencias: cada una lleva fecha de caducidad, así que no puede volverse permanente por olvido, y una que ya no corresponde a ningún aviso vivo también tumba la build, así que la lista no acumula entradas muertas. Apagar el fuego, no desactivar la alarma.

Y la parte honesta: la mitad del plan se topó con un muro de pago, porque el escaneo de código y la protección de rama necesitan plan de pago en repos privados. Sustituí lo que pude, escribí **qué se perdía en cada sustitución**, y dejé la protección de rama anotada como aplazada, y no la quité en silencio. Dentro de seis meses, nadie debería poder confundir este pipeline con uno que hace algo que no hace.
