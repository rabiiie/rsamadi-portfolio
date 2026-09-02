# Documentación fotográfica: OCR, geocoding y borrado de marcas de agua

> Write-up saneado: sin código propietario, sin credenciales, sin nombres de cliente ni de localidad. Las mediciones son reales; los sitios están generalizados.

## Impacto

El proceso automático que trata las fotos que entregan las cuadrillas en campo leía 1 de cada
27, y cuando no encontraba una dirección rellenaba una ubicación falsa sin avisar. Todo eso
acababa en el material que se entrega al cliente.

- **Calidad del dato.** La lectura automática del rótulo pasa de 1 de 27 a 27 de 27 sobre la misma muestra, y de 6 de 21 a 21 de 21 sobre una segunda. Las fotos que antes había que completar a mano se resuelven solas.
- **Seguridad y datos verificados.** Corregidos dos fallos que no daban error: una comprobación de acceso que nunca casaba, y un geocoder que devolvía el centro del país a 140 km cuando no encontraba la dirección. Ahora cada coordenada lleva su precisión (portal, calle o pueblo) y la que no la tiene no se acepta.
- **Coste controlado.** El borrado de la marca de agua con Amazon Bedrock baja del 55% al 34% de superficie tapada, que es lo que impide que el modelo se invente el fondo. El OCR pasa a costar el doble por foto, 1,20 € cada 400 en vez de 0,60 €, y esa es la contrapartida de la precisión de arriba.

## Resumen

Las cuadrillas fotografían las instalaciones de fibra y cada foto lleva un rótulo estampado con
empresa, pueblo, nodo, coordenadas y dirección. Un microservicio en Python lee ese rótulo,
verifica la ubicación, la escribe en el EXIF, corrige el rótulo cuando no cuadra con la
dirección, renombra los ficheros según la nomenclatura de cada cliente y borra el rótulo para
producir el material que se entrega.

El entregable es la propia foto. Una coordenada con 140 km de error y una correcta son
indistinguibles en un listado de ficheros, y no hay ninguna etapa posterior que valide: o se
comprueba al producir el dato, o no se comprueba.

## Resultados

| | Antes | Después |
|---|---|---|
| Fotos con el rótulo leído entero (muestra de 27) | 1 | 27 |
| Segunda muestra, con el reparto de banda corregido (21) | 6 | 21 |
| Superficie de foto que tapa la máscara de borrado | 55% | 34% |
| Coste de OCR por cada 400 fotos | 0,60 € | 1,20 € |

- El geocoder devolvía una posición siempre, también cuando no encontraba la dirección. Ahora cada coordenada lleva su precisión (portal, calle o pueblo) y la posición sola no se acepta.
- Un trabajo de renombrado terminaba como *completado* sin haber renombrado nada cuando faltaba el fichero de referencia. Ahora corta antes de empezar.
- Cinco bugs de interfaz que se habían tratado por separado resultaron ser el mismo: el estado del trabajo vivía en tres sitios.
- La comprobación de acceso comparaba el nombre del cliente contra su código, así que nunca casaba y solo dejaba un `WARN`. En la práctica no había autorización.

## Decisiones de ingeniería

- **Dos llamadas de OCR por foto en lugar de una.** El rótulo se lee mal a escala completa, así que la banda inferior se recorta y se envía aparte. Duplica el coste de OCR y es lo que sube la lectura de 1 de 27 a 27 de 27.
- **La foto es la unidad, no la carpeta.** Una sola respuesta por carpeta habría sellado fotos de tres calles distintas con la misma coordenada, sin dar error.
- **Dos fuentes independientes para la misma dirección**, el nombre del fichero y el rótulo. Si coinciden, la foto se aprueba sola; si discrepan, decide el operador sobre la imagen.
- **La marca de agua se borra por renglones y no por rectángulo.** Con el 55% de la foto tapada el modelo se inventaba el fondo.
- **Residencia de datos escalada al negocio**, no resuelta por ingeniería: el modelo que borra el rótulo solo existe en Estados Unidos y las marcas de agua llevan dirección postal y GPS.

**Stack.** Python · FastAPI · Amazon Textract · Amazon Location Service · Amazon Bedrock ·
Pillow · PostgreSQL · Spring Boot · React · EXIF/GPS. Detalle al final.

**Cómo corre.** Microservicio Python sobre Windows Server on-premise, con una instancia del
módulo por hilo del pool. En la nube van OCR, geocoding, extracción de campos e inpainting; la
base de datos y los ficheros se quedan en casa. Lo consumen el backend de Spring Boot y la
interfaz React.

---

## 1. El OCR perdía las dos últimas líneas del rótulo

En fotos de 3060×4080, `DetectDocumentText` devolvía las líneas de arriba (empresa, pueblo,
nodo) y se dejaba las dos últimas, que son las que hacen falta: coordenadas y dirección. Sobre
una muestra de 27 fotos, 1 de 27. No es calidad del motor sino resolución relativa: a escala de
foto completa esas líneas ocupan pocos píxeles.

La banda inferior se recorta y se manda como segunda llamada, fundiendo lo que devuelve con el
texto completo. Sobre la misma muestra, 27 de 27. Cuatro cosas que salieron al medirlo:

**Lo que manda es el área del recorte, no el margen.** Medido sobre una foto: un recorte de 2017×1344 devolvía `748`; el mismo recorte a 2017×1018 devolvía `50.00748`. Por encima de aproximadamente 1,5 Mpx el servicio reduce la imagen y vuelve a comerse el primer carácter de cada renglón. El alto de la banda se calcula ahora contra ese presupuesto.

**El recorte necesita su propio margen**, 60 px. Un glifo que toca el borde de la imagen se corta: `50.01774` llegaba como `0.01774`, que ya no es una latitud europea, así que la foto se descartaba por ilegible. Si la línea no parsea, queda en el log y se ve. Si parsea a otro número que también es una latitud válida, no se entera nadie.

**Se mandan las dos mitades de la banda.** El rótulo va pegado a un margen, y a cuál depende de la app de cámara. Deducir el lado por el centro de masas de la primera pasada **falla**: el texto impreso en otra parte del encuadre desplaza el centro al lado contrario y el recorte cae al lado del rótulo y no encima. Sobre una segunda muestra: 6 de 21 deduciendo el lado, 16 de 21 mandando las dos mitades, 21 de 21 aplicando además el presupuesto de área.

**En la fusión gana la lectura más larga, no la última.** El recorte derecho solo alcanza la cola de los renglones del rótulo izquierdo, y su `748` truncado estaba pisando el `50.00748` completo del otro.

**Un arreglo relacionado.** La función que extrae coordenadas usaba `.search()`, o sea solo la primera coincidencia. El OCR concatena números de renglones contiguos y forma pares falsos de aspecto válido (`10.89201, 08.2026`) antes del par correcto. Cambiado a `finditer`, comprobando el rango de cada par y parando en el primero que cae donde trabaja la cuadrilla.

**Hasta dónde llega la banda.** La segunda pasada corre también sobre los bloques, no solo sobre el texto plano: sus cajas se traducen a coordenadas de la foto completa y se fusionan con las de la primera. De ellas dependen la localización de la marca de agua y la máscara de borrado.

**Coste.** Dos llamadas de OCR por foto y no una: unos 1,20 € por cada 400 fotos en lugar de 0,60 €. El borrado sigue siendo una sola llamada.

---

## 2. El geocoder no avisa cuando no encuentra la dirección

El servicio de búsqueda devuelve una posición siempre. Medido contra los datos de un pueblo:

| Consulta | Qué devuelve | Error |
|---|---|---|
| Dirección real | `PlaceType=PointAddress` | el portal correcto |
| Calle real, número inventado | `PlaceType=Street` | punto medio de la calle, ~860 m |
| Calle inventada | `PlaceType=Locality` | centro del pueblo |
| Sin coincidencia | la posición de sesgo | centro del país, ~140 km |

Los cuatro casos superan el filtro de plausibilidad geográfica. Una guarda por distancia contra el centro del pueblo detecta los dos últimos y falla en el caso crítico: un número de portal inexistente cae dentro del radio del pueblo y se acepta.

Esa coordenada se escribe en el EXIF de la foto y de ahí pasa al informe del cliente. Una dirección mal tecleada acaba documentada como ubicación real.

El error de diseño era pedir únicamente una posición, cuando el servicio devuelve además un tipo de lugar que indica qué parte de la consulta ha casado. Ahora se conserva y se traduce a una precisión explícita (portal, calle o pueblo) que llega hasta la pantalla del operador. La posición sola no se acepta nunca. La guarda por distancia se mantiene: en un pueblo descartó 3 de 17 carpetas.

---

## 3. Un dato por foto, no por carpeta

Las fotos sin coordenadas utilizables caen en una carpeta llamada `sin_coordenadas`. Al llevar un diálogo de escritorio al navegador salía muy barato convertir una pregunta por foto en una sola respuesta por carpeta.

Lo comprobé contra datos reales antes de hacerlo: una de esas carpetas tenía fotos de tres calles distintas. La dirección está en el nombre de cada fichero, con el código de nodo como sufijo, y no en el de la carpeta, que es literalmente `sin_coordenadas`. Colapsar el diálogo habría sellado fotos de portales diferentes con la misma coordenada, sin dar ningún error.

Así que todo dato que se le pide al operador se modela por foto, y lo que no venga en la respuesta se salta, igual que dejar un campo en blanco en el diálogo original.

---

## 4. Corregir el rótulo y renombrar

Además de leer el rótulo, el servicio lo reescribe y renombra los ficheros. Las dos operaciones
escriben sobre el entregable, y un fallo produce una foto que parece correcta con datos de otra
ubicación.

**Dos fuentes independientes para la misma dirección.** El nombre del fichero lleva la
dirección, y el rótulo también. Se normalizan las dos y se comparan. Para coincidir estando mal
tendrían que haberse equivocado igual, así que cuando coinciden y no falta ningún campo la foto
se aprueba sola, sin preguntar. Medido sobre un lote con el rótulo cambiado: 7 de 10 coincidían,
y las 3 que discrepaban eran exactamente las que había que corregir.

**Cuando discrepan, decide el operador.** Se le enseña la foto con el recuadro del rótulo
marcado, las dos lecturas, el texto crudo del OCR y las ocho líneas que el rótulo va a pintar,
que son más que la dirección. Puede corregir los campos, mover el recuadro o girar la foto 90°:
una foto de lado deja el rótulo en el lateral y el recuadro no se encuentra, así que al girarla
se rehace la lectura desde el OCR y el operador marca sobre la imagen buena.

En las fotos aparecen metros y reglas, y sus números entraban en el texto que se parsea como
dirección, así que el recorte de esos objetos va antes del filtrado y no después.

Los campos se sacan en dos pasadas: primero expresiones regulares sobre el texto del rótulo, y
lo que quede sin resolver va a un modelo en Bedrock, sobre el texto ya leído y no sobre la
imagen.

El renombrado tiene cinco nomenclaturas, una por cliente. Dos cruzan contra un Excel de
direcciones, una contra una tabla de referencia en CSV, y dos se resuelven con lo que ya está
en el nombre del fichero.

**Fallo detectado en el renombrado.** Sin el Excel o el CSV, la tabla de referencia
se quedaba vacía, la función avisaba en su log y se salía, y el trabajo terminaba como
**completado sin haber renombrado nada**. Ahora la falta del fichero es un error que corta el
trabajo antes de empezar, y una operación que no devuelve resumen también, en lugar de darse
por hecha.

Cada foto deja su fila, en base de datos y en un CSV junto a las fotos: fecha de ejecución,
fichero original, calle y número del nombre, calle y número del rótulo, si coincidían, qué se
hizo con ella y dónde acabó. Las que no se pueden resolver van a una carpeta de revisión, no al
resultado.

Una ejecución interrumpida deja ficheros temporales de rotación en la misma carpeta. Se excluyen
del recuento: no son fotos del cliente y contarlas infla el total.

---

## 5. Estado del trabajo y resultados de lote

**El lote no devolvía cifras.** La capa de servicio devolvía una cadena fija de éxito y la función de debajo ni siquiera devolvía sus estadísticas. Una ejecución podía terminar, reportar que fue bien y haber escrito **cero** coordenadas, sin forma de distinguirla de una que las corrigió todas.

Ahora las operaciones de lote devuelven cuentas reales: cuántas miró, cuántas corrigió, cuántas siguen pendientes y por qué. Y la interfaz enseña una fila por foto, no un resumen.

**El estado vivía en tres sitios.** La memoria del microservicio, una tabla de trabajos y el navegador, con dos bucles sincronizándolos. De esa única duplicación salían cinco fallos distintos:

- `Running` mostrado junto a `completed`
- trabajos correctos marcados como `Failed` al reiniciar el microservicio
- progreso clavado en 0 %
- un mensaje de error viejo pegado a la fila
- un 401 en el stream de eventos

Cinco manifestaciones de un único defecto, tratadas hasta entonces como bugs independientes.

Ahora hay un único dueño: el servicio que hace el trabajo escribe su estado y una línea de log numerada por evento; el otro servicio lee. Se retiró el sondeo entre servicios y el relay de eventos: el navegador pide las líneas posteriores a la última que tiene. El log pasa además a sobrevivir al cierre de la pestaña, pérdida silenciosa que no estaba registrada como defecto.

---

## 6. Autorización sin efecto

La comprobación anterior llamaba a una consulta de proyecto que pertenece a otra línea de producto, y que además no comprueba nada cuando el identificador llega nulo. En la práctica no había autorización.

**El defecto que apareció al rehacerla.** La tabla de clientes guarda el nombre visible y el código en columnas distintas. La comprobación recibía el **nombre** y lo comparaba contra el **código**. Nunca casaban, así que se caía por el camino de en medio y lo único que quedaba era un `WARN` en el log. Los tests pasaban y la pantalla funcionaba, pero no había ningún control.

El código de cliente se resuelve ahora en el servidor a partir del identificador, y nunca se acepta del navegador.

**Dos decisiones relacionadas:**

- **La raíz de ficheros fallaba abierta.** Sin una propiedad explícita de rutas permitidas, la lógica caía al recurso de red entero, incluidos los ficheros de otra línea de producto. Ahora esa política vive en un solo sitio y falla cerrada.
- **El fallback antiguo abre en lugar de cerrar.** La regla "sin filas de scope, acceso completo" admite a quien no tiene nada configurado, mientras excluye correctamente a quien ya tiene scopes de ese cliente por otro módulo. El caso permisivo no genera incidencia y por eso permanece sin detectar.

---

## 7. Residencia de datos: decisión de negocio

Borrar el rótulo es la función que más le importa al negocio, y el modelo que lo hace solo existe en regiones de Estados Unidos. El anterior, alojado en Europa, está en fin de vida y cerrado a clientes nuevos, y elegirlo habría repetido un fallo que ya está en producción, que es montar un pipeline sobre un modelo que luego retiran. Y recortar el trozo antes de enviarlo no resuelve nada: el recorte **es** la zona de la dirección.

**La decisión.** Se registró como decisión de residencia de datos y no como problema técnico: con el coste dicho (las marcas de agua llevan dirección postal, GPS, altitud y hora), tomada por quien es dueño de esa decisión, y con una condición escrita para revisarla si el producto se comercializa o se abre a más clientes. El resto de servicios se quedan en la región europea.

**Notas de operación.** El modelo se invoca por perfil de inferencia y no aparece en el listado de modelos, solo en el de perfiles. Una única llamada con una máscara que cubre todos los fragmentos sustituye a un bucle de una invocación por caja. Unos 32 s para un PNG de 12 MB.

**Defecto de la máscara.** Borrar el rectángulo que envuelve el rótulo tapaba el 55% del recorte. Con esa proporción cubierta el modelo se quedaba sin contexto alrededor y generaba fondo sintético: en una foto, un muro de arenisca y unos peldaños se convirtieron en hormigón liso. Enmascarando los renglones de texto por separado baja al 34% y el fondo real sobrevive. Cajas de línea y no de palabra, porque el rótulo va alineado a la derecha y con palabras sueltas quedaban sin tapar los finales de renglón. Verificado sobre 10 fotos: 10 limpias.

---

## Límites del alcance

En un pueblo, 40 fotos pendientes: **18 resueltas**.

Las otras 22 no tienen coordenadas ni en la imagen ni en el EXIF: de las 40, **ninguna** traía GPS. Entre las cuadrillas circulan varias apps de cámara y solo algunas estampan posición: de los cuatro formatos de rótulo que aparecieron, dos llevan coordenadas y dos solo fecha o nombre del técnico. Si la app de cámara no estampó la posición, no hay nada que leer.

Para esas queda el nombre de la carpeta, que **es** una dirección. Geocodificarlo da la coordenada del portal, no la del punto donde estaba el técnico. Está implementado y guardado con un origen distinto, para que no se mezcle con una coordenada leída de la foto. Una llamada de geocoding por carpeta, no por foto.

Dos fallos de formato más que cayeron en la misma pasada: la expresión regular de coordenadas con hemisferio no tragaba el formato de una de las apps por el símbolo de grado entre número y letra, y la de fechas solo aceptaba un separador.

## Stack

Amazon Textract · Amazon Location Service · Amazon Bedrock · Python · FastAPI · Pillow · PostgreSQL · Spring Boot · React · metadatos EXIF/GPS
