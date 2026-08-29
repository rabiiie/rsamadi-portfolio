# Documentación fotográfica — OCR, geocoding y borrado de marcas de agua

> Write-up saneado: sin código propietario, sin credenciales, sin nombres de cliente ni de localidad. Las mediciones son reales; los sitios están generalizados.

## De qué va

Las cuadrillas fotografían las instalaciones de fibra. Cada foto lleva un rótulo estampado con empresa, pueblo, nodo, coordenadas y dirección. El pipeline lee ese rótulo, verifica la ubicación, la escribe en el EXIF de la foto, borra el rótulo y produce el conjunto documentado que se entrega al cliente.

- **Microservicio**: Python con FastAPI, en un Windows Server on-premise, con una instancia del módulo por hilo del pool.
- **En la nube**: OCR, geocoding, extracción de campos e inpainting. La base de datos y los ficheros se quedan en casa.
- **Consumidores**: el backend de Spring Boot y la interfaz React.

Aquí lo que se entrega **es la foto**. Una coordenada con 140 km de error y una correcta se ven exactamente igual en un listado de ficheros, y no hay ninguna etapa posterior que rechace ninguna de las dos. O validas en el momento de producirla, o no validas.

---

## 1. El OCR perdía las dos últimas líneas del rótulo

**El problema.** En fotos de 3060×4080, `DetectDocumentText` devolvía las líneas de arriba (empresa, pueblo, nodo) y se dejaba sistemáticamente las dos últimas: coordenadas y dirección. Justo las dos que hacían falta. Lectura sobre una muestra de 27 fotos: 1 de 27.

**La causa.** Resolución relativa, no calidad del motor. A escala de foto completa esas líneas ocupan pocos píxeles.

**El arreglo.** Recortar la banda inferior y mandarla como una segunda llamada, fundiendo las líneas que devuelve con el texto completo. Sobre la misma muestra: **27 de 27**.

Y cuatro cosas que salieron midiendo, mientras lo implementaba:

**Lo que manda es el área del recorte, no el margen.** Medido sobre una foto: un recorte de 2017×1344 devolvía `748`; el mismo recorte a 2017×1018 devolvía `50.00748`. Por encima de aproximadamente 1,5 Mpx el servicio reduce la imagen y vuelve a comerse el primer carácter de cada renglón. El alto de la banda se calcula ahora contra ese presupuesto.

**El recorte necesita su propio margen**, 60 px. Un glifo que toca el borde de la imagen se corta: `50.01774` llegaba como `0.01774`, que ya no es una latitud europea, así que la foto se descartaba por ilegible. Una línea que no parsea se ve en el log; una línea que parsea a otro número con buena pinta, no.

**Se mandan las dos mitades de la banda.** El rótulo va pegado a un margen, y a cuál depende de la app de cámara. Deducir el lado por el centro de masas de la primera pasada **falla**: el texto impreso en otra parte del encuadre desplaza el centro al lado contrario y el recorte cae al lado del rótulo y no encima. Sobre una segunda muestra: 6 de 21 deduciendo el lado, 16 de 21 mandando las dos mitades, 21 de 21 aplicando además el presupuesto de área.

**En la fusión gana la lectura más larga, no la última.** El recorte derecho solo alcanza la cola de los renglones del rótulo izquierdo, y su `748` truncado estaba pisando el `50.00748` completo del otro.

**Un arreglo relacionado.** La función que extrae coordenadas usaba `.search()`, o sea solo la primera coincidencia. El OCR concatena números de renglones contiguos y forma pares falsos con buena pinta (`10.89201, 08.2026`) antes del par bueno. Cambiado a iterar y parar en la primera coincidencia **plausible**, que es otra cosa distinta de la primera coincidencia.

**Hasta dónde llega la banda.** La segunda pasada corre también sobre los bloques, no solo sobre el texto plano: sus cajas se traducen a coordenadas de la foto completa y se fusionan con las de la primera. De ellas dependen la localización de la marca de agua y la máscara de borrado.

**Coste.** Dos llamadas de OCR por foto y no una: unos 1,20 € por cada 400 fotos en lugar de 0,60 €. El borrado sigue siendo una sola llamada.

---

## 2. El geocoder nunca dice que no ha encontrado la dirección

**El problema.** El servicio de búsqueda siempre devuelve una posición. Medido contra los datos de un pueblo:

| Consulta | Qué devuelve | Error |
|---|---|---|
| Dirección real | `PlaceType=PointAddress` | el portal correcto |
| Calle real, número inventado | `PlaceType=Street` | punto medio de la calle, ~860 m |
| Calle inventada | `PlaceType=Locality` | centro del pueblo |
| Sin coincidencia | la posición de sesgo | centro del país, ~140 km |

Las cuatro pasan el filtro de "esto cae plausiblemente en Europa". Una guarda por distancia contra el centro del pueblo caza las dos últimas y **falla justo en la que más duele**: un número de portal que no existe cae cómodamente dentro del radio del pueblo y se acepta como bueno.

**Por qué importa.** Esa coordenada se escribe en el EXIF de la foto y de ahí pasa al informe del cliente. Una dirección mal tecleada acaba documentada como ubicación real.

**El arreglo.** El error de diseño era pedirle al servicio una posición, cuando además devuelve un **tipo de lugar**, que es el servicio diciéndote cuánto de tu consulta ha casado de verdad. Ahora se conserva y se traduce a una precisión explícita (portal, calle o pueblo) que llega hasta la pantalla del operador. La posición sola no se acepta nunca. La guarda por distancia se mantiene: en un pueblo descartó 3 de 17 carpetas.

---

## 3. La unidad es la foto, no la carpeta

**El problema.** Las fotos sin coordenadas utilizables caen en una carpeta llamada `sin_coordenadas`. Al llevar un diálogo de escritorio al navegador era cómodo convertir una pregunta por foto en una sola respuesta por carpeta.

**Por qué descarté ese camino.** Comprobado contra datos reales: una de esas carpetas tenía fotos de tres calles distintas. La dirección está en el nombre de cada fichero, con el código de nodo como sufijo, no en el nombre de la carpeta, que es literalmente `sin_coordenadas`. Colapsar el diálogo habría sellado fotos de portales diferentes con una misma coordenada, sin dar ningún error.

**El arreglo.** Todo dato que se le pide al operador se modela por foto. Lo que no venga en la respuesta se salta, igual que dejar un campo en blanco en el diálogo original.

---

## 4. Estado del trabajo y resultados de lote

**Problema A: el lote no decía nada.** La capa de servicio devolvía una cadena fija de éxito y la función de debajo ni siquiera devolvía sus estadísticas. Una ejecución podía terminar, reportar que fue bien y haber escrito **cero** coordenadas, sin forma de distinguirla de una que las corrigió todas.

Un estado sin números detrás es peor que ningún estado, porque cierra la investigación.

**El arreglo.** Las operaciones de lote devuelven cuentas reales: cuántas miró, cuántas corrigió, cuántas siguen pendientes y por qué. Y la interfaz enseña una fila por foto, no un resumen.

**Problema B: el estado vivía en tres sitios.** La memoria del microservicio, una tabla de trabajos y el navegador, con dos bucles sincronizándolos. De esa única duplicación salían cinco fallos distintos:

- `Running` mostrado junto a `completed`
- trabajos correctos marcados como `Failed` al reiniciar el microservicio
- progreso clavado en 0 %
- un mensaje de error viejo pegado a la fila
- un 401 en el stream de eventos

Cinco caras, un solo fallo. Y cada cara se había mirado como si fuera un bug propio.

**El arreglo.** Un único dueño: el servicio que hace el trabajo escribe su estado y una línea de log numerada por evento; el otro servicio lee. Se retiró el sondeo entre servicios y el relay de eventos: el navegador pide las líneas posteriores a la última que tiene. De regalo, el log sobrevive a cerrar la pestaña, que era otra pérdida silenciosa que nadie había apuntado como bug.

---

## 5. La autorización que avisaba en vez de denegar

**El problema.** La comprobación anterior llamaba a una consulta de proyecto que pertenece a otra línea de producto, y que además no comprueba nada cuando el identificador llega nulo. En la práctica no había autorización.

**El fallo que apareció rehaciéndola.** La tabla de clientes guarda el nombre visible y el código en columnas distintas. La comprobación recibía el **nombre** y lo comparaba contra el **código**. Nunca casaban, así que se caía por el camino de en medio y lo único que quedaba era un `WARN` en el log. Tests en verde, funcionalidad operativa, control ninguno.

**El arreglo.** El código de cliente se resuelve en el servidor a partir del identificador, y nunca se acepta del navegador.

**Dos decisiones relacionadas:**

- **La raíz de ficheros fallaba abierta.** Sin una propiedad explícita de rutas permitidas, la lógica caía al recurso de red entero, incluidos los ficheros de otra línea de producto. Ahora esa política vive en un solo sitio y falla cerrada.
- **El fallback antiguo es la dirección peligrosa.** "Sin filas de scope, acceso completo" deja entrar a quien no tiene nada configurado, mientras que a quien ya tiene scopes de ese cliente por otro módulo lo deja fuera correctamente. El caso permisivo es el que no genera ticket, y por eso es el que sobrevive sin que nadie lo vea.

---

## 6. La decisión que no se arregla con ingeniería

**La restricción.** Borrar el rótulo es la función que más le importa al negocio, y el modelo que lo hace solo existe en regiones de Estados Unidos. El anterior, alojado en Europa, está en fin de vida y cerrado a clientes nuevos, y elegirlo habría repetido un fallo que ya está en producción, que es montar un pipeline sobre un modelo que luego retiran. Y recortar el trozo antes de enviarlo no resuelve nada: el recorte **es** la zona de la dirección.

**La decisión.** Se registró como lo que es, una decisión de residencia de datos y no un problema técnico: con el coste dicho (las marcas de agua llevan dirección postal, GPS, altitud y hora), tomada por quien es dueño de esa decisión, y con una condición escrita para revisarla si el producto se comercializa o se abre a más clientes. El resto de servicios se quedan en la región europea.

**Notas de operación.** El modelo se invoca por perfil de inferencia y no aparece en el listado de modelos, solo en el de perfiles. Una única llamada con una máscara que cubre todos los fragmentos sustituye a un bucle de una invocación por caja. Unos 32 s para un PNG de 12 MB.

**El fallo de la máscara.** Borrar el rectángulo que envuelve el rótulo tapaba el 55% del recorte. Con esa proporción cubierta el modelo se quedaba sin contexto alrededor y **se inventaba el fondo**: en una foto, un muro de arenisca y unos peldaños volvieron convertidos en hormigón liso. Enmascarando los renglones de texto por separado baja al 34% y el fondo real sobrevive. Cajas de línea y no de palabra, porque el rótulo va alineado a la derecha y con palabras sueltas quedaban sin tapar los finales de renglón. Verificado sobre 10 fotos: 10 limpias.

---

## Hasta dónde llega, dicho en voz alta

En un pueblo, 40 fotos pendientes: **18 resueltas**.

Las otras 22 no tienen coordenadas ni en la imagen ni en el EXIF: de las 40, **ninguna** traía GPS. Entre las cuadrillas circulan varias apps de cámara y solo algunas estampan posición: de los cuatro formatos de rótulo que aparecieron, dos llevan coordenadas y dos solo fecha o nombre del técnico. Ninguna cantidad de trabajo en OCR recupera un número que nunca se escribió.

Para esas queda el nombre de la carpeta, que **es** una dirección. Geocodificarlo da precisión de portal, no la del punto donde estaba el técnico: es otra medida con las mismas unidades. Está implementado y marcado con un origen distinto, para que no se mezcle jamás con una coordenada leída de la foto. Una llamada de geocoding por carpeta, no por foto.

Dos fallos de formato más que cayeron en la misma pasada: la expresión regular de coordenadas con hemisferio no tragaba el formato de una de las apps por el símbolo de grado entre número y letra, y la de fechas solo aceptaba un separador.

## Stack

Amazon Textract · Amazon Location Service · Amazon Bedrock · Python · FastAPI · Pillow · PostgreSQL · Spring Boot · React · metadatos EXIF/GPS
