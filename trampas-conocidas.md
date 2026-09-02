# Trampas conocidas

Problemas que costaron encontrar, con la causa y lo que hay que mirar para no repetirlos.
Agrupados por área, porque casi todos vuelven a aparecer cuando se construye algo parecido.

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
reglas de asignación de recursos y la navegación. Ninguno era un fallo de programación: los
nueve estaban bien escritos y funcionaban el día que se escribieron. El fallo es tener nueve
copias de la misma lista, porque solo se descubre que una se quedó atrás cuando alguien usa
justo ese camino.

**Solución.** Un catálogo como fuente única y un único punto que responde si un usuario entra a
un módulo de un cliente. Cada controlador declara su módulo en una anotación, así que la
comprobación viaja con el endpoint. Lo que no se puede unificar de golpe se sujeta con tests:
un contador de controladores sin declarar que solo puede bajar, y comprobaciones de que cada
nombre usado existe en el catálogo y de que cada módulo concedible lleva a una pantalla.

**Qué mirar la próxima vez.** Contar cuántas veces está escrita la misma lista antes de añadirle
una entrada. Y desconfiar de un permiso que se puede conceder: que la pantalla de administración
lo ofrezca no demuestra que ningún endpoint lo consulte. Un grant que no mira nadie es peor que
no tenerlo, porque parece concedido.

**Qué queda pendiente.** La migración va módulo a módulo. El contador dice cuántos controladores
siguen decidiéndose por la vía antigua.

### El permiso se comprobaba en un sitio y se pintaba desde otro

**Síntoma.** Un usuario con permiso de lectura sobre unas obras, y con un grupo de columnas
concedido, veía esas celdas editables. Escribía, guardaba, y recibía un 403. Desde fuera parecía
que el permiso de arriba —el grupo de columnas— mandaba sobre el de abajo —el nivel del recurso—,
cuando en realidad mandaba uno en lo que se veía y otro en lo que se podía.

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
ofrece tienen que salir del mismo cálculo. Un endpoint de capacidades que devuelve varias formas
de la misma respuesta —un flag, un mapa, una lista— es una invitación a que solo una de ellas se
mantenga al día, y la interfaz elige la que le viene bien.

**Por qué no se vio antes.** Todos los usuarios eran administradores, y un administrador pasa las
dos comprobaciones. Un permiso reducido no se prueba solo: hay que fabricarlo.

