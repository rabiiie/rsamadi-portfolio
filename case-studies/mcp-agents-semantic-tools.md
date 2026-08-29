# Agentes de IA sobre MCP — tools semánticas en vez de text-to-SQL

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente. Los identificadores están generalizados.

## De qué va

Dos agentes de dominio sobre una plataforma FTTH, que responden preguntas de operación en lenguaje natural: cuántos homes hay pasados en tal municipio, qué contratos se cancelaron con la obra ya hecha, qué está listo para activar.

- **Agente A**: superficie SQL restringida. Ya existía, y le puse un control de admisión.
- **Agente B**: 19 tools hechas a medida sobre Model Context Protocol, que envuelven las mismas vistas que leen los informes oficiales.
- **Orquestación**: Python con FastAPI, respuestas en streaming por SSE. El servidor de tools es Spring Boot con el MCP server de Spring AI sobre SSE.
- **Autenticación**: Token Relay con JWT de Keycloak.

---

## 1. Por qué tools y no text-to-SQL

**La restricción.** Las métricas de esta plataforma ya tienen definición. "Homes passed" es una función sobre códigos de estado, validada contra la normativa técnica del cliente. El informe semanal excluye una lista de proyectos. "Facturado" quiere decir que un campo concreto no está vacío.

Un modelo que se escribe su propio SQL no ve nada de eso. Ve tablas. Y da un número **aritméticamente correcto e institucionalmente falso**, dicho además con una frase bien construida, a alguien que lo va a pegar en un informe que va al cliente.

**La regla que adopté.** Las tools envuelven la lógica de negocio que ya existe; nunca la reimplementan. El agente y el informe semanal resuelven contra la misma vista. No pueden contradecirse porque solo hay una definición y no la controla ninguno de los dos.

**Cómo quedó.** 19 tools parametrizadas: resolver un proyecto, resumen de hitos de un proyecto o de toda la cartera, contratos por estado, ficha de un home concreto, consultas de lo que está listo para la siguiente fase, detección de anomalías. Ninguna `query(sql)` genérica.

**Un efecto que no esperaba.** Los usuarios no dicen códigos de proyecto, dicen nombres de pueblo. Con una tool de SQL genérica el modelo se habría inventado el join. Una tool dedicada traduce nombre a código, o responde que no lo encuentra — que es un fallo mucho mejor que una consulta segura contra el proyecto equivocado.

**Y algo que descarté al revisar el primer diseño**: un `REPLACE(home_id, '-', '_')` dentro de los joins, que anula el uso de índices. Eso se arregla con una columna normalizada en la base de datos, no con una transformación de texto en la capa de tools.

---

## 2. Cada agregado viene con su cobertura

**El problema.** Que un agregado se deje filas fuera en silencio es aceptable en un dashboard: el usuario ve una gráfica y aplica el escepticismo normal ante una gráfica. Un agente no tiene esa ventaja. Dice *"4.812 homes pasados"* en una frase bien formada, y esa fluidez es en sí misma una afirmación implícita de que están todos.

**El diseño.** Toda tool de agregación devuelve la cifra **más un bloque `coverage`**: qué se incluyó, qué se excluyó y con qué motivo, y las advertencias que apliquen. Una consulta por periodo sobre hitos devuelve el conteo y, si toca, una advertencia diciendo que hay N registros sin fecha y que por eso quedan fuera del periodo. Si no se excluyó nada, la lista de advertencias va vacía — que también es una afirmación.

En la misma respuesta viajan dos cosas más, y las dos existen por cómo un modelo lee su propia salida:

- **Procedencia.** Cada respuesta nombra la vista de la que sale el dato, identificada como la fuente del informe semanal oficial. Si el usuario pregunta de dónde viene el número, el modelo ya tiene el dato en vez de improvisarlo.
- **Relaciones que no debe equivocar.** Una respuesta devuelve dos métricas donde una es subconjunto estricto de la otra, y lleva literalmente la frase `"HP+ es subconjunto de HP; no sumar"`. Dos números juntos en un payload son una invitación a sumarlos.

**La regla general:** si un número necesita una advertencia cuando lo dice una persona, esa advertencia tiene que viajar en el payload, no en el prompt. Las instrucciones del prompt se aplican a la conversación; no van pegadas a la cifra.

**Paginación.** Las tools de listado tienen un tope duro de filas. Cuando el resultado se corta, la respuesta marca `has_more` y devuelve un mensaje para que el orquestador acote la consulta. Un LLM al que le das 50.000 filas no responde mejor: responde peor, más caro, o no responde.

---

## 3. Autorización, en tres capas

El requisito venía dicho con claridad: acceso asignado para usar IA, y restricción por dominio — si te dan el agente de un cliente, no contesta del otro.

**Capas uno y dos.** Resultaron ser el mismo arreglo. El endpoint del stream estaba protegido por un comodín de ruta y un rol general de la aplicación, así que entraba cualquiera autenticado. Y el identificador del agente era un trozo de la URL que el proxy reenviaba sin validar: un usuario podía escribir el nombre de otro agente y pasar.

Ahora hay un catálogo que mapea agente → dominio, y un guard que lo comprueba en los dos proxies **antes** de reenviar. La authority es `ROLE_<dominio>_AGENT` y una sola asignación cubre los dos requisitos. Una petición no autorizada se rechaza con 403 sin gastar una llamada al modelo.

**Capa tres: el scope de recurso, por Token Relay.** El JWT del navegador va al orquestador Python, que valida solo firma y caducidad, y lo reenvía intacto al servidor MCP en Spring. Spring lo valida como resource server y aplica el mismo guard de scope por proyecto que usa la API REST. **Python autentica, nunca autoriza.** La decisión se queda en el servicio que es dueño del dato, y hay una sola implementación de ella.

**Y el fallo que merece contarse.** El MCP server de Spring AI ejecuta las tools en un scheduler elástico de Reactor, no en el hilo de la petición HTTP. El contexto de Spring Security está atado al hilo. Así que cuando una tool preguntaba qué proyectos puede ver el usuario, el contexto estaba vacío: la identidad se había validado dos veces y se caía al suelo entre el filtro y la tool.

Falló cerrado, que es la dirección buena. Pero el atajo tentador era el peligroso: como el check falla, la llamada es interna y el token ya venía validado, quitas el check. Eso deja el scope por proyecto sin ningún punto de aplicación, y ningún test lo detecta, porque en un test tú eres el usuario que lo ve todo.

El arreglo real captura la autenticación en el momento de invocar la tool, la expone durante toda la llamada y la limpia en un `finally`. Lo de limpiar no es adorno: un hilo de un pool que se quede con la identidad de un usuario se la aplica al siguiente.

**La versión general:** cuando montas un framework encima de un runtime asíncrono, pregúntate cuáles de tus supuestos de seguridad estaban atados al hilo. No avisan; simplemente dejan de ser ciertos.

---

## 4. La superficie SQL del agente que ya existía

El agente A respondía por SQL generado desde antes del trabajo con MCP. En vez de reescribirlo le puse un control de admisión, y las decisiones de ahí merecen contarse porque la versión ingenua de cada una está mal.

- **Lista blanca de las tablas realmente referenciadas**, no lista negra de palabras peligrosas. Las listas negras se pierden. La pregunta contestable es si toda tabla que aparece tras `FROM` o `JOIN` está en la lista o es un CTE declarado en el propio `WITH`.
- **Los literales y los comentarios se eliminan antes de analizar.** Si no, un nombre de tabla permitido escrito dentro de una cadena o de un comentario avala una consulta que además toca otra que no lo está.
- **Los nombres sin cualificar también tienen que estar en la lista**, y esta es la que no es obvia. PostgreSQL los resuelve por `search_path`, que alcanza el esquema compartido donde viven las tablas de otros clientes. Admitirlos en bloque es una lectura entre clientes sin necesidad de exploit. Lo de ese cliente hay que escribirlo cualificado por esquema.
- **Un esquema entero sí está permitido**, porque pertenece por completo a un cliente. Es una excepción cómoda a propósito: si añadir una vista para ese cliente obligara a editar una clase de seguridad, la gente dejaría de añadir vistas — o peor, las pondría en el esquema compartido.
- `MAX_SQL_LEN` 8000, y ejecución de solo lectura.

Dos agentes, dos superficies, un principio: la superficie es tan ancha como el dominio pide y ni una tabla más.

---

## 5. Lo que se rompió: una sesión SSE reutilizada entre tareas

**El síntoma.** El stream del agente MCP se quedaba colgado unos treinta segundos y moría. Al cerrar, lanzaba *"attempted to exit a cancel scope in a different task than it was entered in"*.

**Cómo lo aislé, que es lo que vale.** El agente A — misma plataforma, mismo modelo, mismo camino de streaming, pero hablando por HTTP normal en vez de MCP — funcionaba perfectamente. Eso descartó de golpe el rate limit, la red, el modelo y la capa de streaming, y dejó al cliente MCP como único sospechoso.

**La causa.** El cliente abría la conexión SSE una vez y reutilizaba la sesión entre llamadas. El SDK de MCP se apoya en task groups de anyio, cuyos cancel scopes tienen que entrar y salir **en la misma tarea** — y la respuesta en streaming es un generador asíncrono, así que cada `yield` de vuelta al framework cruza una frontera de tarea. La tarea que leía el SSE quedaba huérfana y la respuesta de la tool no llegaba nunca.

**El arreglo.** Cada llamada abre, inicializa, usa y cierra su propia conexión, sin ningún `yield` por medio. Cuesta una reconexión por llamada. El pool de conexiones es una optimización real y está anotada como tal, pero encima de algo que ya funciona, no en lugar de arreglarlo.

---

## 6. Del pulgar abajo al caso de evaluación

**El problema.** La salida de un agente es prosa y los fallos son cualitativos, así que sin nada más un cambio de prompt se evalúa a ojo.

**El circuito de feedback.** Una valoración negativa en producción no se queda en un ticket. Se cruza con el turno de conversación que la produjo **y con las tool calls que hay debajo** — qué tool se ejecutó y con qué argumentos —, se clasifica de forma heurística en una categoría probable de fallo y entra como candidato pendiente, sin duplicar por id de feedback. Una persona confirma o corrige la categoría antes de que se convierta en caso permanente.

Lo de enganchar las tool calls es lo que hace que un fallo sea atribuible: eligió la tool equivocada, extrajo mal los parámetros, o tenía el dato bien y lo resumió mal. Son tres defectos distintos con tres arreglos distintos, y el texto de la queja no distingue ninguno.

**Las suites.** Seis, partidas por esas mismas líneas: enrutado de tools, planificación, resolución de nombres, síntesis de la respuesta y dos bancos de preguntas por dominio. Puntúa un modelo como juez, porque las respuestas son prosa y comparar cadenas solo mediría la redacción. Cada ejecución se guarda con su resultado por caso, así que un cambio de prompt da un número en vez de una impresión.

**Dos límites que conviene decir.** El juez es de la misma familia de modelos que el agente evaluado, que es una debilidad conocida de este método. Y el runner va secuencial para no chocar con el rate limit, así que la suite es lenta y se lanza a propósito, no en cada commit.

---

## Dónde está hoy

El camino MCP funciona de punta a punta. Queda un tema abierto, y lo digo tal cual: el modelo está en capa gratuita y el agente hace unas dos llamadas por pregunta, así que preguntas seguidas chocan con el límite. El backoff dura más que el timeout asíncrono del servlet, con lo que un rate limit se convierte en un stream muerto. Está aparcado por decisión: sacar el stream del servlet elimina la mitad del problema, y la otra mitad es cambiar de plan.

**El diseño se rehízo una vez.** La primera versión apuntaba a tablas crudas. Al pasarle siete preguntas reales de usuario se vio que la lógica que necesitaba ya estaba implementada en la capa semántica, incluida una lista de exclusión de proyectos que aplica el informe oficial y que las tools habrían contradicho. La segunda versión envuelve esa capa. Antes de implementarla resolví seis dudas con un script de profiling — valores reales de los enums, distribución de campos, cardinalidades — porque filtrar por un valor supuesto devuelve cero filas y ni un error.

## Stack

Model Context Protocol · MCP server de Spring AI (SSE) · Spring Boot · Spring Security OAuth2 Resource Server · Keycloak (Token Relay) · FastAPI · Python · anyio · Google Gemini · vistas semánticas en PostgreSQL
