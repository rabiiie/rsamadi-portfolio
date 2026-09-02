# Trampas conocidas

Problemas que costaron encontrar, con su causa y la comprobación que los detecta. Agrupados
por área.

## GIS y caché

### El filtro CORS impedía cachear las teselas

**Síntoma.** El proxy de delante servía el 0% de las teselas desde su caché, aunque las
respuestas salían con `Cache-Control: public` y `max-age`.

**Causa.** Las teselas se piden siempre al mismo host que sirve el visor, así que no
necesitan CORS. Pero el `CorsFilter` de Spring Security añade una cabecera `Vary` a todas
las respuestas, antes incluso de mirar su configuración. Con `Vary` presente, IIS/ARR se
niega a cachear.

**Solución.** Un filtro registrado con `HIGHEST_PRECEDENCE`, por delante de la cadena de
Spring Security, que envuelve la respuesta solo en las rutas de teselas y descarta cualquier
`Vary`. El orden importa: el envoltorio tiene que estar puesto antes de que el `CorsFilter`
intente escribir la cabecera.

**Qué mirar la próxima vez.** Si algo que debería cachearse no se cachea, mirar las cabeceras
de la respuesta real, no la configuración. `Vary`, `Set-Cookie` y `Authorization` bastan para
que un proxy descarte una respuesta perfectamente cacheable.

**Qué queda pendiente.** La condición del filtro busca `/tiles/` en la ruta, y la ruta de
PMTiles es `/pmtiles/`, así que no entra. Es menor, porque las respuestas por rangos son 206
y ARR tampoco cachea 206.

## Autorización

### Nueve copias de la misma lista

**Síntoma.** Añadir un módulo al modelo de permisos no bastaba para poder concederlo. Según
por dónde se entrase, el módulo aparecía en la pantalla de administración pero no en el menú,
o entraba la pantalla y fallaba la petición de datos, o al revés. Cada arreglo destapaba el
siguiente.

**Causa.** La lista de módulos existía escrita a mano en nueve sitios: las reglas de rutas del
backend, el catálogo de permisos, el mapa de portadas, el de pestañas, la validación del panel
de administración, los grupos de roles del frontend, el registro de módulos de la interfaz, las
reglas de asignación de recursos y la navegación. Ninguna de las nueve estaba mal escrita. El
defecto es la duplicación: una copia desactualizada solo se detecta al recorrer ese camino
concreto.

**Solución.** Un catálogo como fuente única y un único punto que responde si un usuario entra a
un módulo de un cliente. Cada controlador declara su módulo en una anotación, así que la
comprobación viaja con el endpoint. Lo que no se puede unificar de golpe se sujeta con tests:
un contador de controladores sin declarar que solo puede bajar, y comprobaciones de que cada
nombre usado existe en el catálogo y de que cada módulo concedible lleva a una pantalla.

**Qué mirar la próxima vez.** Contar cuántas veces está escrita la misma lista antes de añadirle
una entrada. Que la pantalla de administración ofrezca un permiso no demuestra que
ningún endpoint lo consulte: un permiso que nadie lee aparenta estar concedido.

**Qué queda pendiente.** La migración va módulo a módulo. El contador dice cuántos controladores
siguen decidiéndose por la vía antigua.

### El permiso se comprobaba en un sitio y se pintaba desde otro

**Síntoma.** Un usuario con permiso de lectura sobre unas obras, y con un grupo de columnas
concedido, veía esas celdas editables. Escribía, guardaba, y recibía un 403. Aparentemente el grupo de
columnas prevalecía sobre el nivel del recurso; en realidad uno decidía lo visible y otro lo
permitido.

**Causa.** El endpoint que le dice al cliente qué puede hacer devuelve dos cosas: un flag de
cabecera y un mapa de booleanos por grupo de columnas. El flag cruzaba el grupo con el nivel del
recurso; el mapa se calculaba solo con el grupo. Y el que lee la interfaz para decidir si una
celda es editable es el mapa. La comprobación de escritura, por su parte, estaba bien: rechazaba
el guardado.

**Solución.** Cruzar el nivel también en el mapa, y dejar vacía la lista de columnas editables de
un grupo que no lo es, para no dejar una segunda vía. Tres tests fijan el resultado: nivel de
lectura no abre el grupo, nivel de edición sí, y el nivel de edición no concede un grupo que no
se tiene.

**Qué mirar la próxima vez.** El dato que decide si algo se puede hacer y el que decide si se
ofrece deben salir del mismo cálculo. Un endpoint de capacidades que devuelve la misma respuesta
en varias formas (flag, mapa, lista) tiende a mantener actualizada solo una.

**Por qué no se vio antes.** Todos los usuarios eran administradores, y un administrador pasa las
dos comprobaciones. Un permiso reducido no se prueba solo: hay que fabricarlo.

### El frontend recalculaba permisos que el backend ya sabía

**Síntoma.** Fallaba en las dos direcciones a la vez: usuarios a los que se les negaban pantallas
que sí les tocaban, y enlaces de menú visibles que llevaban a un 403.

**Causa.** Los guards de ruta, la navegación y la lógica de a qué pantalla entra cada usuario
mantenían cada uno su propia unión de nombres de rol, escrita a mano. Cada vez que nacía una
familia de permisos, esas listas no se actualizaban. El mapa del servidor estuvo bien todo el
tiempo; los que se habían desviado eran sus consumidores.

**Solución.** El endpoint de sesión devuelve los módulos, las vistas, los informes y las acciones
del usuario, y la pantalla a la que entra. El frontend pinta eso y no deriva accesos por su
cuenta. Añadir una familia de permisos pasó a ser tocar un único mapa en el servidor.

**Qué mirar la próxima vez.** Una regla de autorización calculada en el cliente duplica lo que el
servidor ya resuelve, y es la copia que se desactualiza. Una unión de nombres de rol escrita a
mano en el frontend es un defecto pendiente de manifestarse.

### La columna existía y las consultas no la miraban

**Síntoma.** A un usuario se le asignaban unas obras en un módulo y otras distintas en otro, y
veía las dos listas en los dos sitios.

**Causa.** La tabla de permisos por recurso tenía una columna de módulo desde el primer día, con
su comentario en el esquema explicando que un valor nulo significa "todos los módulos de este
cliente". La pantalla de administración la escribía. Las consultas que leen esos permisos no la
filtraban, así que cualquier permiso de un cliente valía para todos sus módulos. El esquema iba
por delante del código, y el caso solo aparece cuando alguien tiene dos módulos con recursos
distintos, que es raro mientras todo el mundo es administrador.

**Solución.** Una única función decide si un permiso aplica a la pregunta, con dos reglas de
compatibilidad escritas a propósito: un permiso sin módulo vale para todo el cliente (situación
de todos los permisos anteriores a la columna, de forma que nada de lo asignado cambia), y una
pregunta que no distingue módulo cuenta cualquier permiso. La segunda regla es la crítica: si los
permisos por módulo fueran invisibles ahí, el usuario parecería no tener ninguno y la regla
antigua le abriría el cliente entero. Omitir el filtro amplía el acceso, no lo restringe.

**Qué mirar la próxima vez.** Cuando llega una columna nueva al esquema, buscar quién la lee, no
quién la escribe. Una columna que solo se escribe no es un control.

## Datos e integraciones

### El mismo cliente con dos números

**Síntoma.** Al llevar una aplicación de escritorio a la web, una inserción empezó a fallar por
clave foránea. Ese error resultó ser el caso menos grave.

**Causa.** El esquema heredado del escritorio numera sus propios clientes, aparte de los de la
plataforma. De tres clientes, uno tenía un id inexistente al otro lado (el que producía el error
visible) y los otros dos tenían el mismo número en ambas tablas apuntando a clientes distintos. Esos no fallan: entran, y el registro queda facturado al cliente equivocado. Buscando
esto apareció además el filtro por cliente de un informe comentado, con una nota al lado que
decía que de momento no se filtraba para no vaciar resultados.

**Solución.** El puente entre los dos espacios de numeración es el código del cliente, que es el
mismo texto en las dos tablas y único en la heredada. Nada que viva en el esquema heredado recibe
un identificador de la plataforma: se resuelve por código antes de entrar.

**Qué mirar la próxima vez.** Cuando dos sistemas se juntan, comprobar si sus identificadores
comparten espacio de numeración antes de pasarlos de uno a otro. Un id presente en ambos
lados con significados distintos no produce error: escribe en la fila equivocada. El fallo visible
puede ser el síntoma menor de otro silencioso y más antiguo.

