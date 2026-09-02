# Agentes de IA sobre MCP: tools semánticas en lugar de text-to-SQL

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente. Los identificadores están generalizados.

## Contexto

Dos agentes de dominio sobre una plataforma FTTH, que responden preguntas de operación en lenguaje natural: homes pasados en un municipio, contratos cancelados con la obra ya ejecutada, elementos listos para activar.

- **Agente A**: superficie SQL restringida, preexistente. Se le añadió un control de admisión.
- **Agente B**: 19 tools a medida sobre Model Context Protocol, que envuelven las mismas vistas que leen los informes oficiales.
- **Orquestación**: Python con FastAPI, respuestas en streaming por SSE. El servidor de tools es Spring Boot con el MCP server de Spring AI sobre SSE.
- **Autenticación**: Token Relay con JWT de Keycloak.

---

## 1. Tools frente a text-to-SQL

Las métricas de la plataforma tienen definición previa. "Homes passed" es una función sobre códigos de estado, validada contra la normativa técnica del cliente; el informe semanal excluye una lista de proyectos; "facturado" significa que un campo concreto no está vacío.

Un modelo que genera su propio SQL no dispone de esas definiciones: ve tablas. El resultado es una cifra aritméticamente correcta e institucionalmente falsa, expresada con seguridad y destinada a un informe de cliente.

Criterio adoptado: las tools envuelven la lógica de negocio existente y no la reimplementan. El agente y el informe semanal resuelven contra la misma vista, de modo que no pueden dar cifras distintas.

19 tools parametrizadas: resolución de proyecto, resumen de hitos por proyecto y de cartera completa, contratos por estado, ficha de un home, consultas de elementos listos para la siguiente fase y detección de anomalías. Ninguna `query(sql)` genérica.

Efecto no previsto: los usuarios no utilizan códigos de proyecto, sino nombres de municipio. Con una tool de SQL genérica el modelo habría construido el join por su cuenta. Una tool dedicada traduce nombre a código o responde que no lo encuentra, fallo preferible a una consulta correcta sobre el proyecto equivocado.

Descartado en la revisión del primer diseño: un `REPLACE(home_id, '-', '_')` dentro de los joins, que anula el uso de índices. Se resuelve con una columna normalizada en la base de datos, no con una transformación de texto en la capa de tools.

---

## 2. Cobertura en cada agregado

Un agregado que descarta filas en silencio es asumible en un dashboard, donde el usuario aplica el escepticismo habitual ante una gráfica. Un agente responde "4.812 homes pasados" en una frase bien formada, y esa formulación afirma implícitamente que están todos.

Toda tool de agregación devuelve la cifra más un bloque `coverage`: qué se incluyó, qué se excluyó y con qué motivo, y las advertencias aplicables. Una consulta por periodo sobre hitos devuelve el conteo y, cuando procede, la advertencia de que N registros sin fecha quedan fuera del periodo. Sin exclusiones, la lista de advertencias va vacía.

En la misma respuesta viajan dos elementos más:

- **Procedencia.** Cada respuesta nombra la vista de origen del dato, identificada como la fuente del informe semanal oficial. Ante una pregunta sobre el origen, el modelo dispone del dato y no lo genera.
- **Relaciones entre métricas.** Cuando una respuesta devuelve dos métricas y una es subconjunto estricto de la otra, incluye literalmente `"HP+ es subconjunto de HP; no sumar"`.

Criterio: una advertencia necesaria para interpretar un número viaja en el payload, no en el prompt. Las instrucciones del prompt se aplican a la conversación y no acompañan a la cifra.

**Paginación.** Las tools de listado tienen un tope duro de filas. Al cortarse el resultado, la respuesta marca `has_more` y devuelve un mensaje para que el orquestador acote la consulta.

---

## 3. Autorización en tres capas

Requisito: acceso asignado para el uso de IA y restricción por dominio, de forma que el agente de un cliente no responda sobre otro.

**Capas uno y dos.** Se resolvieron con el mismo cambio. El endpoint del stream estaba protegido por un comodín de ruta y un rol general de la aplicación, accesible a cualquier usuario autenticado; además, el identificador del agente era un segmento de URL que el proxy reenviaba sin validar, de modo que bastaba escribir el nombre de otro agente.

Existe ahora un catálogo que mapea agente a dominio y un guard que lo comprueba en los dos proxies antes de reenviar. La authority es `ROLE_<dominio>_AGENT` y una sola asignación cubre ambos requisitos. Una petición no autorizada se rechaza con 403 sin llamar al modelo.

**Capa tres: scope de recurso por Token Relay.** El JWT del navegador llega al orquestador Python, que valida firma y caducidad y lo reenvía intacto al servidor MCP en Spring. Spring lo valida como resource server y aplica el mismo guard de scope por proyecto que la API REST. Python autentica y no autoriza: la decisión permanece en el servicio propietario del dato, con una única implementación.

**Defecto detectado.** El MCP server de Spring AI ejecuta las tools en un scheduler elástico de Reactor, no en el hilo de la petición HTTP, y el contexto de Spring Security está asociado al hilo. Al consultar una tool qué proyectos puede ver el usuario, el contexto estaba vacío: la identidad se validaba dos veces y se perdía entre el filtro y la tool.

El fallo era cerrado. El atajo aparente (suprimir la comprobación, dado que la llamada es interna y el token ya venía validado) habría dejado el scope por proyecto sin punto de aplicación, sin que ningún test lo detectara: en un test el usuario ejecutor lo ve todo.

El arreglo captura la autenticación en el momento de invocar la tool, la expone durante la llamada y la limpia en un `finally`. La limpieza es necesaria: un hilo de pool que conserve la identidad de un usuario la aplica al siguiente.

Al montar un framework sobre un runtime asíncrono hay que revisar qué supuestos de seguridad estaban ligados al hilo.

---

## 4. Superficie SQL del agente preexistente

El agente A respondía mediante SQL generado desde antes del trabajo con MCP. En lugar de reescribirlo se le añadió un control de admisión. La versión ingenua de cada decisión es incorrecta:

- **Lista blanca de las tablas referenciadas**, no lista negra de palabras peligrosas. La comprobación es que toda tabla tras `FROM` o `JOIN` esté en la lista o sea un CTE declarado en el propio `WITH`.
- **Literales y comentarios se eliminan antes del análisis.** En caso contrario, un nombre de tabla permitido escrito dentro de una cadena o un comentario avala una consulta que toca otra no permitida.
- **Los nombres sin cualificar también deben estar en la lista.** PostgreSQL los resuelve por `search_path`, que alcanza el esquema compartido donde viven las tablas de otros clientes; admitirlos en bloque permite lectura entre clientes sin necesidad de exploit. Las tablas del cliente se escriben cualificadas por esquema.
- **Un esquema completo sí está permitido**, por pertenecer íntegramente a un cliente. Excepción deliberada: si añadir una vista obligara a editar una clase de seguridad, las vistas acabarían en el esquema compartido.
- `MAX_SQL_LEN` 8000 y ejecución de solo lectura.

Dos agentes y dos superficies, con el mismo criterio: la superficie abarca el dominio y ninguna tabla más.

---

## 5. Sesión SSE reutilizada entre tareas

**Síntoma.** El stream del agente MCP quedaba bloqueado unos treinta segundos y terminaba con el error *"attempted to exit a cancel scope in a different task than it was entered in"*.

**Aislamiento.** El agente A (misma plataforma, mismo modelo y mismo camino de streaming, pero sobre HTTP y no sobre MCP) funcionaba correctamente. Eso descartó rate limit, red, modelo y capa de streaming, y dejó al cliente MCP como único candidato.

**Causa.** El cliente abría la conexión SSE una vez y reutilizaba la sesión entre llamadas. El SDK de MCP se apoya en task groups de anyio, cuyos cancel scopes deben entrar y salir en la misma tarea; la respuesta en streaming es un generador asíncrono, de forma que cada `yield` de vuelta al framework cruza una frontera de tarea. La tarea que leía el SSE quedaba huérfana y la respuesta de la tool no llegaba.

**Corrección.** Cada llamada abre, inicializa, usa y cierra su propia conexión, sin `yield` intermedios. Coste: una reconexión por llamada. El pool de conexiones queda anotado como optimización posterior, sobre un camino que ya funciona.

---

## 6. Del feedback negativo al caso de evaluación

La salida de un agente es prosa y sus fallos son cualitativos, de modo que sin instrumentación un cambio de prompt se evalúa por impresión.

**Circuito de feedback.** Una valoración negativa en producción se cruza con el turno de conversación que la produjo y con las tool calls subyacentes (qué tool se ejecutó y con qué argumentos), se clasifica de forma heurística en una categoría probable de fallo y entra como candidato pendiente, sin duplicar por id de feedback. Una persona confirma o corrige la categoría antes de convertirlo en caso permanente.

Enganchar las tool calls es lo que hace atribuible el fallo: elección de tool incorrecta, extracción incorrecta de parámetros, o dato correcto mal resumido. Tres defectos con tres correcciones distintas, indistinguibles desde el texto de la queja.

**Suites.** Seis, divididas por esas mismas líneas: enrutado de tools, planificación, resolución de nombres, síntesis de la respuesta y dos bancos de preguntas por dominio. La puntuación la da un modelo como juez, al tratarse de respuestas en prosa donde la comparación de cadenas mediría la redacción. Cada ejecución se guarda con su resultado por caso.

**Dos límites.** El juez pertenece a la misma familia de modelos que el agente evaluado, debilidad conocida del método. Y el runner es secuencial para no chocar con el rate limit, de modo que la suite es lenta y se lanza de forma explícita, no en cada commit.

---

## Estado actual

El camino MCP funciona de extremo a extremo. Queda un tema abierto: el modelo está en capa gratuita y el agente realiza unas dos llamadas por pregunta, de forma que las preguntas seguidas alcanzan el límite. El backoff dura más que el timeout asíncrono del servlet, con lo que un rate limit produce un stream muerto. Aparcado por decisión: sacar el stream del servlet elimina la mitad del problema y la otra mitad es un cambio de plan.

El diseño se rehízo una vez. La primera versión apuntaba a tablas crudas. Al contrastarla con siete preguntas reales de usuario se comprobó que la lógica necesaria ya estaba implementada en la capa semántica, incluida una lista de exclusión de proyectos que aplica el informe oficial y que las tools habrían contradicho. La segunda versión envuelve esa capa. Antes de implementarla se resolvieron seis dudas con un script de profiling (valores reales de los enums, distribución de campos, cardinalidades): filtrar por un valor supuesto devuelve cero filas y ningún error.

## Stack

Model Context Protocol · MCP server de Spring AI (SSE) · Spring Boot · Spring Security OAuth2 Resource Server · Keycloak (Token Relay) · FastAPI · Python · anyio · Google Gemini · vistas semánticas en PostgreSQL
