# Pipeline de CI: qué comprobaciones bloquean

> Write-up saneado: sin código propietario, sin credenciales y sin datos de cliente.

## Contexto

Un repositorio políglota: backend Java 17 con Spring Boot, frontend React con Capacitor, dos servicios Python y el esquema completo de PostgreSQL/PostGIS. Un desarrollador. Usuarios en producción.

De dónde partía: tres comprobaciones (escaneo de secretos, auditoría de dependencias de npm y una compilación con tests parcial), con cobertura desigual según el lenguaje, el esquema de base de datos fuera de todo control automático, y el despliegue hecho a mano.

Criterio de diseño: el objetivo es que cualquier commit de la rama principal sea desplegable sin intervención manual, no acumular escáneres. Las comprobaciones cuyo resultado no sería interpretable todavía quedan aplazadas:

- La cobertura se mide después de que el contexto de Spring arranque en CI. Si el conjunto de tests deja fuera el arranque, el porcentaje no habla del código que se despliega.
- Los tests de integración van después de tener migraciones versionadas, porque son las migraciones las que construyen el esquema que el test necesita.
- El análisis de estilo y bugs entra con una línea base del código existente, nunca sobre pizarra limpia.

---

## 1. Qué bloquea y qué no

| Comprobación | Modo | Por qué |
|---|---|---|
| Escaneo de secretos sobre todo el historial | tumba la ejecución | una credencial filtrada se rota, no se valora |
| Auditoría de dependencias de producción (npm) | tumba la ejecución | superficie pequeña y conocida; cada hallazgo es accionable hoy |
| Compilación y tests unitarios | tumba la ejecución | — |
| Arranque del contexto de Spring contra un PostGIS real | tumba la ejecución | solo fue posible después de versionar el esquema |
| SAST | hallazgos informativos, fallo de la herramienta fatal | ver 1.2 |
| Vulnerabilidades de dependencias Java | informativo, programado | aparecen avisos nuevos sin que el código cambie |
| Lint del frontend | informativo, temporal | 736 avisos preexistentes; endurece cuando estén triados |

### 1.1 Introducción de controles nuevos

Los controles nuevos entran en modo informativo y endurecen cuando su lista de hallazgos está vacía o justificada. Bloquear desde el primer día sobre un repositorio con historia detiene el trabajo durante el triaje del ruido inicial.

Cada estado informativo lleva asociada una fecha o una condición de endurecimiento. Sin eso se queda informativo para siempre y nadie vuelve a mirarlo.

Excepción: una comprobación limitada al diff de un pull request bloquea desde el principio, al no producir ruido histórico.

### 1.2 SAST: los hallazgos informan, la herramienta bloquea

El job de SAST no lleva `continue-on-error`. Los hallazgos no tumban la ejecución porque el escáner no se invoca en modo error; un fallo de ejecución del escáner, sí.

Con `continue-on-error` los dos casos salen igual y se ignoran igual, así que una mala configuración o un fallo de red pueden estar semanas sin verse mientras el pipeline dice que SAST pasa.

---

## 2. Excepciones que caducan

Un hallazgo de dependencias se puede aceptar, pero cada aceptación lleva fecha de caducidad, y pasada esa fecha la build vuelve a fallar. Y una excepción que ya no corresponde a ningún aviso vivo también tumba la build, así que la lista no puede acumular entradas muertas.

La política que va con eso: una excepción solo es admisible cuando se ha verificado que la superficie afectada no se usa. Si el aviso tiene arreglo, se arregla. Apagar el fuego, no desactivar la alarma.

El escaneo de secretos tiene la misma forma. Los hallazgos históricos se permiten **por commit**, cada uno anotado con qué credencial era y la fecha en que se revocó, de modo que escanear todo el historial no bloquee cada ejecución mientras sigue cazando lo nuevo. La lista de rutas permitidas contiene solo directorios que no son código del proyecto: dependencias, artefactos de build, ficheros del IDE. Nunca fuentes: un secreto encontrado en el código se rota y se quita del fichero, no se silencia.

---

## 3. Versionado del esquema

El esquema no estaba versionado: unos setenta ficheros `.sql` sueltos en la raíz del repositorio, aplicados a mano, sin registro en ninguna parte del orden de aplicación, de si eran idempotentes ni del estado de cada entorno. Eso dejaba el job de arranque del contexto en informativo, porque el test necesita una base de datos con el esquema ya construido y no había forma automática de construirla, así que estaba directamente excluido de la ejecución.

Línea base y verificación:

- Línea base generada con `pg_dump --schema-only` sobre una copia en contenedor, y después **verificada cargándola en una base de datos vacía**: 351 relaciones, 868 índices de usuario, 1.014 funciones, 424 restricciones, idénticas al origen y sin errores.
- De 1.264 tablas, **1.002 las crea la aplicación en tiempo de ejecución**: tablas de staging por proyecto, particiones diarias de historial. Esas no son esquema y quedan fuera; incluirlas haría que la línea base creciera sola. Las tablas particionadas padre sí entran.
- Los dos caminos verificados de punta a punta. Base vacía: la migración corre, construye el esquema y la validación de JPA lo acepta contra las entidades. Base existente: se registra una línea base y la migración **no** corre.
- La imagen del contenedor de test es PostGIS y no `postgres` a secas, porque la línea base crea extensiones espaciales. Los esquemas de esas extensiones se crean con `IF NOT EXISTS` porque vienen con la imagen; los esquemas de la aplicación se crean **sin** esa cláusula, a propósito, para que una colisión de verdad falle a la vista.

**Migración desacoplada del arranque.** Las migraciones están desactivadas por defecto porque el usuario de base de datos con el que corre la aplicación no tiene permisos DDL, y no debe tenerlos. Aplicar migraciones es un paso de despliegue con sus propias credenciales, no algo que ocurra cada vez que alguien arranca la aplicación contra un servidor compartido.

Dos hallazgos de esa verificación, ajenos a CI: una tabla particionada sin partición `DEFAULT`, y que el primer arranque en un entorno real escribiría una tabla de historial de migraciones. Los dos anotados antes de tocar nada.

---

## 4. Tests sobre los planes de ejecución

El contenedor que se montó para el test de arranque sirve además para que CI compruebe **cómo se ejecutan las queries**, no solo que la aplicación levanta.

Los tests siembran filas suficientes para que el planificador se tome los índices en serio, y después comprueban con `EXPLAIN` que cada query crítica sigue usando el índice para el que se diseñó. La eliminación accidental de un índice deja el pull request en rojo, en lugar de aparecer como degradación en producción.

Dos limitaciones:

- `SET enable_seqscan = off` **encarece** los scans secuenciales, no los prohíbe. Sobre un fixture pequeño, un índice que falta sigue produciendo un scan secuencial y no un error, así que la comprobación tiene que leer el plan y el fixture tiene que ser lo bastante grande para ser representativo.
- Estos tests son lentos y se seleccionan por tag de JUnit, así que la build rápida no los paga. Corren como job propio, y son deterministas para cada pull request, a diferencia de las mediciones de tiempo, que no lo son y van programadas.

El reparto es: planes en cada pull request, tiempos en la ejecución programada. Un `EXPLAIN` devuelve el mismo plan en cualquier máquina; un tiempo depende del runner, y en runners compartidos esos tests fallan de forma intermitente.

---

## 5. Sustituciones forzadas por el plan del repositorio

La fase de bajo coste se escribió dando por hecho que el escaneo de código, la revisión de dependencias y la protección de rama eran gratis. En un repositorio **privado** las tres están detrás del mismo plan de pago. Lo que desaparece no es el aviso, sino el bloqueo en el merge.

| Lo planeado | El sustituto | Qué se perdió |
|---|---|---|
| Escaneo de código nativo de GitHub | Semgrep OSS | nada relevante: cubre Java y JavaScript, corre en el runner, no depende de la API de GitHub y se invoca con métricas desactivadas |
| Revisión de dependencias | OWASP Dependency-Check | **no es equivalente**: escanea el árbol entero periódicamente en vez de bloquear el diff de un PR. El bloqueo previo al merge desaparece; a cambio, las dependencias transitivas de Maven tienen cobertura por primera vez |
| Renovate | Dependabot | agrupa peor en un repositorio políglota; a cambio es nativo, gratis en repos privados, y su agrupación por ecosistema cubre los seis frentes: dos módulos Maven, npm, dos servicios Python y las propias actions del workflow |

La protección de rama queda anotada como aplazada a la espera de un plan de pago, no descartada en silencio. Mientras no exista, "tumba la ejecución" significa que la ejecución se pone en rojo, no que el merge esté impedido mecánicamente.

---

## 6. Control de coste

- **Filtrado por rutas.** Un pull request que solo toca el frontend no compila Java. Implementado con salidas de job y condiciones `if`, y no con filtros de ruta a nivel de workflow: un job saltado por `if` cuenta como satisfecho para la protección de rama, mientras que uno saltado por filtro de ruta la deja esperando indefinidamente.
- **Concurrencia con cancelación en curso solo en pull requests.** En la rama principal y en la ejecución programada el job termina, porque su resultado *es* el historial de esa rama.
- **El escaneo lento va programado.** Descargar la base de datos de vulnerabilidades entera lleva casi una hora y no dice nada nuevo sobre un cambio de código. Se dispara en push, semanalmente por cron, y a mano cuando hace falta.
- **Se genera un SBOM en cada build** y se guarda como artefacto, así que si mañana hay que aplicar una política de licencias el inventario ya está hecho.

---

## Estado actual

Los cimientos y los controles de bajo coste están en verde y funcionando. Las fases aplazadas, en el mismo orden: tests de frontend empezando por la lógica que resuelve la autorización, lint y auditoría de dependencias para los servicios Python, cobertura, reglas de arquitectura para las decisiones que hoy solo están escritas en un ADR, entrega automatizada a un entorno de pruebas, y firma y procedencia de artefactos.

## Herramientas

GitHub Actions · gitleaks · Semgrep OSS · OWASP Dependency-Check · CycloneDX SBOM · Dependabot · Testcontainers (PostGIS) · Flyway · JUnit · ESLint · npm audit
