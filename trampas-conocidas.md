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
