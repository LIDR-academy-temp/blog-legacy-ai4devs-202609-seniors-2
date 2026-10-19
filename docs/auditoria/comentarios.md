# Auditoría — feature «comentarios»

**Alcance:** crear (`POST /posts/:slug/comentarios`), editar (`PATCH /comentarios/:id`), borrar
(`DELETE /comentarios/:id`) y listar (los `comentarios[]` de `GET /posts/:slug` y los recuentos
`num_comentarios` de `GET /posts` y `GET /posts/:slug/relacionados`), en tres capas: rutas de la API,
tabla `comentarios` y lo que la web pinta de ella. Quedan fuera los posts, los autores, las
categorías y las etiquetas.

**Cómo se ha comprobado:** leyendo `backend/start/routes.ts`, la migración
`1769000000005_create_comentarios_table.ts`, la semilla, `Comentarios.jsx`, `DetallePost.jsx`,
`ListaPosts.jsx` y `PostsRelacionados.jsx`; con `curl` contra la API levantada en el 3401 con la base
sembrada; y con un test Japa temporal dentro de `withGlobalTransaction()` para las peticiones que
escriben (se revierte solo y el archivo se borró después). Las peticiones que reproducen cada
hallazgo están en [Cómo reproducirlo](#cómo-reproducirlo).

**Orden de la tabla:** primero los hallazgos que bloquean a otro (columna 6). Después, de más a menos
daño y, a igual daño, primero el que toca menos sitios. Los «Nadie todavía» son riesgos y van al
final. El daño más alto que no se ve hoy, «cualquiera puede borrar cualquier comentario», se parte
en dos filas: la contención (H2) y el arreglo total (H21).

La columna **#** solo da nombre a cada fila para poder citarla en la columna 6.

## Hallazgos

| # | Qué es | Dónde está | Qué se rompe, y quién lo nota | Cuántos sitios hay que tocar | ¿Se deshace con un revert? | ¿Bloquea o desbloquea a otro hallazgo? |
|---|---|---|---|---|---|---|
| H1 | `GET /posts/:slug` devuelve el `autor_email` de cada comentarista dentro de `comentarios[]` porque lee con `select *`. | `backend/start/routes.ts:153` (consulta) y `:171` (respuesta) | Cualquier visitante o scraper se lleva el correo de todos los comentarios aprobados. Hoy salen `ana.cortes@ejemplo.es` y `marta.ruiz@corriente.es` en el post 1. Incumple la regla 6. Lo sufren quienes comentan, aunque la web no lo pinte. | 1 (`routes.ts:153`: pedir solo `id, autor_nombre, cuerpo, creado_en, padre_id`, que es lo que usa `Comentarios.jsx:50-70`) | Sí. Solo cambia una lectura, no escribe datos. | Bloquea a **H21**: mientras el correo salga en público, es justo la «credencial» que piden DELETE y PATCH, y con `select `* también se publicaría la columna de token que añade H21. |
| H2 | `DELETE /comentarios/:id` borra el comentario si el cuerpo trae el correo de quien lo escribió, aunque ese correo es público (H1). | `backend/start/routes.ts:350-353` y `:355` | Nadie todavía, porque no hay registro de borrados. Pero cualquier visitante puede borrar cualquier comentario aprobado: comprobado, `DELETE /comentarios/1` con `{"autor_email":"ANA.CORTES@EJEMPLO.ES"}` da `200 {"borrado":true}`. Lo notaría quien vea desaparecer su comentario. **Contención:** quitar la rama `esAutor` y dejar el borrado solo a moderación. La web no tiene botón de borrar, así que nadie pierde funcionalidad. | 1 (`routes.ts:352-355`) | Sí. Solo cambia una condición, no escribe datos. | Bloquea a **H21**: el arreglo total también elimina esta comparación por correo. |
| H3 | La clave de moderación está escrita en el código (`'dev-secret-cambiar'`) y con ella se borra cualquier comentario, también los pendientes. | `backend/start/routes.ts:25` y `:351` | Nadie todavía. Quien pueda leer el repositorio, un fork o su historial de git borra lo que quiera con la cabecera `x-clave-moderacion: dev-secret-cambiar` (comprobado: `200 {"borrado":true}`). | 3 (`routes.ts:25`, `backend/start/env.ts` y `backend/.env.example`) | Sí. Es configuración, no escribe datos. El valor viejo queda en el historial de git, así que además hay que rotarlo. | Bloquea a **H4**, si la aprobación se protege con la misma clave que hoy usa DELETE: con la clave pública, cualquiera aprobaría spam como el comentario 7. |
| H4 | Ningún endpoint cambia `estado`: todo comentario nace `pendiente` y ni la API ni la web tienen forma de aprobarlo. | `backend/start/routes.ts:326` (el INSERT fija `'pendiente'`). Las únicas escrituras en la tabla son `:324` (INSERT), `:359` (DELETE) y `:388` (UPDATE, que solo toca `cuerpo`). | Ningún comentario enviado desde la web llega a publicarse, aunque el formulario promete «pendiente de moderación» (`frontend/src/componentes/Comentarios.jsx:38`). Lo nota cada persona que comenta y cada lector: solo se ven los comentarios de la semilla. | 1 (un endpoint nuevo de moderación que ponga `estado = 'aprobado'`) | Sí. Los aprobados quedan con `'aprobado'`, un valor que el código actual ya lee. | A ninguno. Necesita **H3** antes (ver H3). Ojo: arreglarlo hace visibles H10, H15, H16 y H18. |
| H5 | La columna `estado` es texto libre sin CHECK y en la tabla conviven `'aprobado'`, `'Aprobado'` y `'ok'`, que el detalle no muestra porque compara exacto con `'aprobado'`. | `backend/database/migrations/1769000000005_create_comentarios_table.ts:18`. Datos: `backend/database/seeders/blog_seeder.ts:510, 522, 575`. Filtro: `backend/start/routes.ts:153`. | Los comentarios 3, 4 y 9, aprobados por el panel viejo o por el script de importación, no salen en la web. El post 6 aparece sin comentarios (comprobado: `comentarios: []`). Lo notan quienes los escribieron y los lectores. | 4 (1 migración nueva que normalice los valores y añada el CHECK, más las 3 líneas de la semilla; sin ellas `make db` fallaría contra el CHECK) | Sí para la app: los valores normalizados son válidos para el código viejo. Pero el UPDATE no se deshace: el `down` no puede saber qué filas eran `'Aprobado'` y cuáles `'ok'`. Antes hay que decidir si `'ok'` significa aprobado. | A ninguno. |
| H6 | El listado y los relacionados cuentan los comentarios con `estado <> 'pendiente'`, y el detalle muestra solo los que tienen `estado = 'aprobado'`. | `backend/start/routes.ts:75-76` y `:223`, frente a `:153`. Se pinta en `frontend/src/componentes/ListaPosts.jsx:57` y `PostsRelacionados.jsx:32`. | Post 1: la tarjeta de la portada dice «4 comentarios» y el artículo enseña 2. Post 6: «1 comentarios» en la portada y en relacionados, y 0 en el artículo (comprobado con `curl`). Lo nota cualquier lector que entre desde la portada. Incumple la regla 5. | 2 (`routes.ts:76` y `:223`) | Sí. Solo cambia una lectura. | A ninguno. Arreglar H5 también iguala los números con los datos de hoy, pero el arreglo de H6 no depende de H5. |
| H7 | Al enviar un comentario, la web suma 1 a su recuento aunque el comentario queda pendiente y no se pinta. | `frontend/src/componentes/Comentarios.jsx:39`. La cabecera lleva su propio recuento, que no se entera: `DetallePost.jsx:30, 38, 71`. | Quien acaba de comentar ve «3 comentarios» con 2 visibles, y la cabecera del artículo sigue diciendo 2. Le dura hasta que recarga la página. | 1 (`Comentarios.jsx:39`) | Sí. Es solo frontend, no escribe datos. | A ninguno. |
| H8 | El POST devuelve el último comentario de la tabla (`order by id desc limit 1`) en vez del que acaba de insertar. | `backend/start/routes.ts:329-331` | Nadie todavía. Si dos envíos se intercalan entre el INSERT y el SELECT, una persona recibe el comentario de otra, con su correo. No lo he reproducido: se comprueba leyendo esas líneas. | 1 (`routes.ts:329-331`: leer por el id insertado) | Sí. | A ninguno. |
| H9 | DELETE borra el comentario pero deja sus respuestas apuntando a un `padre_id` que ya no existe, y la web deja de pintarlas. | `backend/start/routes.ts:359` (borra una sola fila). Web: `frontend/src/componentes/Comentarios.jsx:50` (raíces = sin `padre_id`) y `:63-64` (solo pinta hijos de una raíz). | Nadie todavía: la web no tiene botón de borrar y la semilla no tiene borrados. Comprobado en test: tras borrar el id 1, el detalle sigue devolviendo `[[2, 1]]` (id 2 con `padre_id` 1) y la web no lo pinta. Lo notaría quien escribió esa respuesta. | 1 (`routes.ts:359`: rechazar el borrado si tiene respuestas, o borrarlas en la misma operación) | Sí si se rechaza el borrado. Con cascada, la app tampoco falla tras el revert, pero las respuestas borradas no vuelven. | A ninguno. |
| H10 | El POST guarda `padre_id` sin comprobar que exista, que pertenezca al mismo post ni que sea un número, y la tabla no tiene clave foránea en esa columna. | `backend/start/routes.ts:308` y `:326`. Migración `:20` (`integer` nullable, sin FK). | Nadie todavía: la web nunca envía `padre_id` (`Comentarios.jsx:33`). Por API se aceptan con 201 `padre_id: 6` (comentario de otro post), `99999` (no existe) y `"hola"` (SQLite lo guarda como `text`). Esas respuestas no se pintarían nunca (`Comentarios.jsx:64`). | 1 (validación en el POST) | Sí. | A ninguno. |
| H11 | Si `cuerpo` llega como objeto JSON, el POST responde 500 con la sentencia SQL y la traza en el cuerpo de la respuesta. | `backend/start/routes.ts:319` (`String(cuerpo).length` deja pasar un objeto) y `:324-327`. `backend/app/exceptions/handler.ts:9` (`debug = !app.inProduction`). | Nadie todavía: la web siempre envía texto. En desarrollo, un cliente de la API ve la consulta y las rutas de ficheros. En producción la traza no sale, pero sigue siendo un 500 en vez de un 4xx. | 1 (validación de tipo en el POST) | Sí. | A ninguno. |
| H12 | Un cuerpo de menos de 5 caracteres responde `200 {"ok":false,"motivo":"comentario demasiado corto"}` en vez de un 4xx. | `backend/start/routes.ts:319-320` | Nadie todavía en la web, porque mira `d.comentario` (`Comentarios.jsx:37`) y enseña `d.motivo`. Un cliente de la API que se fíe del código de estado da por guardado un comentario que no existe. Cambiarlo es un cambio de contrato: va con su propio test, según CLAUDE.md. | 1 (`routes.ts:320`) | Sí. | A ninguno. |
| H13 | PATCH acepta un cuerpo de 1 carácter y se salta el mínimo de 5 que exige el POST. | `backend/start/routes.ts:384`, frente a `:319` | Nadie todavía: la web no edita. Comprobado: `PATCH /comentarios/5` con `cuerpo: "a"` da 200. | 1 (`routes.ts:384`) | Sí. | A ninguno. |
| H14 | PATCH compara el correo distinguiendo mayúsculas y DELETE no, así que la misma persona puede borrar su comentario pero no editarlo. | `backend/start/routes.ts:379` (`!==` exacto), frente a `:353` (`toLowerCase().trim()`) | Nadie todavía: la web no tiene botones de editar ni de borrar. Comprobado: PATCH con el correo en mayúsculas da 403 y DELETE con el correo en mayúsculas da 200. | 1 (`routes.ts:379`) | Sí. | A ninguno. Desaparece si se hace H21. |
| H15 | Ni el POST ni la tabla limitan la longitud de `autor_nombre` ni de `cuerpo`. | `backend/start/routes.ts:305-321` (ninguna comprobación de máximo). Migración `:15`: `string` es `varchar(255)`, que SQLite no hace cumplir. | Nadie todavía, porque todo queda pendiente (H4). Comprobado: un nombre de 5000 caracteres da 201. Cuando se arregle H4, lo verá cualquier lector. | 1 (validación en el POST) | Sí. Las filas largas que ya existan no rompen el código viejo. | A ninguno. |
| H16 | El POST da por válido cualquier correo que contenga `@` (por ejemplo `x@`). | `backend/start/routes.ts:316` | Nadie todavía. Comprobado: `autor_email: "x@"` da 201. Como el correo es la única prueba de autoría (H2, H14), quien lo escriba mal no podrá gestionar luego su comentario. | 1 (`routes.ts:316`) | Sí. | A ninguno. |
| H17 | La web solo pinta respuestas de primer nivel: una respuesta a otra respuesta nunca se muestra. | `frontend/src/componentes/Comentarios.jsx:63-64` (solo busca hijos de las raíces) | Nadie todavía: la web no permite responder (`Comentarios.jsx:33`) y la semilla solo tiene un nivel (id 2 → id 1). La API sí acepta como `padre_id` el id de una respuesta (H10). | 1 (`Comentarios.jsx:63-64`, o rechazarlo en el POST) | Sí. | A ninguno. |
| H18 | `creado_en` se guarda en UTC sin zona horaria y la web corta los 10 primeros caracteres, así que un comentario escrito cerca de medianoche muestra otro día. | `backend/start/routes.ts:323` (`toISOString()`) y `frontend/src/componentes/Comentarios.jsx:59, 68` (`substring(0, 10)`) | Nadie todavía, porque no se publica nada (H4). Comprobado: un comentario enviado el 20/10 a las 00:13 hora local quedó con `creado_en: "2026-10-19 22:13:45"`. Además, las fechas de la semilla están escritas en hora local, así que la columna mezcla dos zonas. | 1 (`routes.ts:323`) | Sí. La app no falla, aunque las filas creadas con la corrección se quedan con el otro formato. | A ninguno. |
| H19 | `comentarios.post_id` no tiene clave foránea. | Migración `:14`. `PRAGMA foreign_key_list('comentarios')` sale vacío. | Nadie todavía: la API no borra posts y el POST solo inserta tras encontrar el post (`routes.ts:296-303`). | 1 (migración) | Sí. | A ninguno. |
| H20 | Los tres rechazos 4xx del POST usan tres formas y dos códigos distintos: `400 {error}`, `422 {message}` y `400 {errores:[...]}`. | `backend/start/routes.ts:310-318` | Nadie todavía en la web, que lo tapa probando tres claves (`Comentarios.jsx:44`). Un cliente de la API tiene que conocer los tres casos. | 4 (`routes.ts:311`, `:314`, `:317` y `Comentarios.jsx:44`) | Sí. | A ninguno. |
| H21 | La autoría de un comentario no tiene más credencial que el correo de quien lo escribió (arreglo total de H2). | `backend/start/routes.ts:350-353` (DELETE) y `:378-379` (PATCH). Migración `:11-21`: no hay columna para un token. | Con H2 aplicado: nadie todavía (DELETE queda solo para moderación, y PATCH solo sirve sobre pendientes, cuyo correo no se publica). Sin H2: cualquier visitante borra cualquier comentario aprobado. | 4 (migración con la columna del token, POST `routes.ts:324-333` para emitirlo, DELETE `:350-353` y PATCH `:378-379` para exigirlo) | No, si la columna nueva es NOT NULL: el INSERT viejo (`routes.ts:325`) no la rellena y el POST fallaría tras el revert. Para que se pueda revertir, la columna tiene que ser nullable. | A ninguno. Necesita **H1** y **H2** antes. |

## Cómo reproducirlo

Con la API en el 3401 (`make api`) y la base recién sembrada (`make db`).

**Lectura (no escribe nada):**

```bash
# H1: correos de los comentaristas en la respuesta pública
curl -s localhost:3401/posts/diez-tendencias-de-diseno-que-pueden-cambiar-la-web-moderna \
  | python3 -c "import json,sys; [print(c['id'], c['autor_email']) for c in json.load(sys.stdin)['comentarios']]"

# H5 y H6: recuento del listado frente a lo que enseña el detalle
curl -s "localhost:3401/posts?pagina=2" | python3 -c "import json,sys; [print(p['id'], p['num_comentarios']) for p in json.load(sys.stdin)['datos']]"
curl -s localhost:3401/posts/que-mide-de-verdad-una-metrica-de-producto | python3 -c "import json,sys; print(len(json.load(sys.stdin)['comentarios']))"
sqlite3 backend/tmp/db.sqlite3 "select id, post_id, estado, padre_id from comentarios"

# H19
sqlite3 backend/tmp/db.sqlite3 "pragma foreign_key_list('comentarios')"
```

**Escritura:** estas peticiones modifican la base, así que lánzalas desde un test con
`testUtils.db().withGlobalTransaction()` o vuelve a sembrar después con `make db`. Todas se lanzan
contra `/posts/diez-tendencias-de-diseno-que-pueden-cambiar-la-web-moderna/comentarios` (POST) o
`/comentarios/:id`.

| # | Petición | Respuesta observada |
|---|---|---|
| H2 | `DELETE /comentarios/1` con `{"autor_email":"ANA.CORTES@EJEMPLO.ES"}` | `200 {"borrado":true}` |
| H3 | `DELETE /comentarios/2` con la cabecera `x-clave-moderacion: dev-secret-cambiar` | `200 {"borrado":true}` |
| H9 | `DELETE /comentarios/1` con la clave, y después `GET` del detalle | `comentarios` = `[[2, 1]]` (id, padre_id) |
| H10 | POST con `padre_id: 6`, `99999` y `"hola"` | `201` las tres veces; `typeof(padre_id)` = `integer`, `integer` y `text` |
| H11 | POST con `cuerpo: {"a":"xxxxxxxx"}` | `500` con `insert into comentarios …` y los frames en el cuerpo |
| H12 | POST con `cuerpo: "abc"` | `200 {"ok":false,"motivo":"comentario demasiado corto"}` |
| H13 | `PATCH /comentarios/5` con `{"autor_email":"tomas.iglesias@ejemplo.es","cuerpo":"a"}` | `200` con `cuerpo: "a"` |
| H14 | `PATCH /comentarios/5` con `autor_email: "TOMAS.IGLESIAS@ejemplo.es"` | `403` (el mismo correo en mayúsculas sí borra: ver H2) |
| H15 | POST con `autor_nombre` de 5000 caracteres | `201` |
| H16 | POST con `autor_email: "x@"` | `201` |
| H18 | Cualquier POST válido | `creado_en` en UTC (`2026-10-19 22:13:45` para un envío del 20/10 a las 00:13 hora local) |
| H20 | POST sin nombre / sin email / con email `"nope"` | `400 {error}` / `422 {message}` / `400 {errores:[…]}` |

## Comprobado y correcto

- Un comentario `pendiente` no sale en el detalle (`routes.ts:153`): los ids 5 y 7 no aparecen. Se cumple la regla 2.
- No se puede comentar en un post en `borrador`: el POST contra `guia-de-accesibilidad-para-equipos-pequenos` da 404 (`routes.ts:296-302`).
- La web pinta el cuerpo y el nombre del comentario como texto (`{c.cuerpo}`, `{c.autor_nombre}` en `Comentarios.jsx:59-70`). React los escapa, así que no se inyecta HTML.

## Mejoras (fuera de la tabla: no hay un daño que se pueda medir hoy)

| Propuesta | Dónde | Qué habría que medir antes de tocarlo |
|---|---|---|
| Comentarios no debería volver a pedir el post entero: hoy la vista de un artículo hace dos `GET /posts/:slug` | `Comentarios.jsx:19` y `DetallePost.jsx:34` | Peticiones y bytes por vista de artículo (pestaña Network). Hoy son 2 llamadas al detalle; el objetivo es 1. |
| El recuento de los relacionados hace una consulta por post | `routes.ts:217-225` | Consultas por petición (log de queries de Lucid) y latencia de `GET /posts/:slug/relacionados`. Con `MAX_RELACIONADOS = 2` hoy son 2 recuentos. |
| `post_id` no tiene índice y el listado hace un `count(*)` correlacionado por post | Migración `:14` y `routes.ts:75-76` | `EXPLAIN QUERY PLAN` de la consulta del listado y su latencia con un volumen realista (p. ej. 100 000 comentarios). Hoy, con 9 filas, no se puede medir. |
| No hay freno al spam: ni límite por IP, ni captcha, ni verificación del correo | `routes.ts:295-334` | Comentarios pendientes por día y qué proporción es spam (en la semilla, 1 de 2 pendientes, el id 7). |
| Las listas usan el índice como `key` | `Comentarios.jsx:57, 66` | Nada hoy: la lista no cambia después de cargarse. Revisarlo si se añade editar o borrar en vivo. |
| Toda la feature vive en `routes.ts`, sin validador, servicio, repositorio ni transformer | `routes.ts:292-393` | No es un daño: es el refactor objetivo de CLAUDE.md. Un validador VineJS resolvería a la vez H10, H11, H12, H15, H16 y H20, y un transformer, H1. |
| La única prueba cubre solo el camino feliz del POST y afirma un único campo | `backend/tests/functional/comentarios.spec.ts:41-42` | Cobertura con `npm run test:coverage`. Según CLAUDE.md, cada arreglo de la tabla empieza con su test de caracterización en verde. |

## Fuera de alcance (visto, pero de otra feature)

- `GET /posts/:slug` también devuelve `post.autor_email` (`routes.ts:111`, `select * from posts`). Incumple la regla 6, pero pertenece a la feature de posts.
- El cuerpo del post se inyecta sin sanear (`DetallePost.jsx:92`). También es de la feature de posts.


-------------------------
**Cuántos hallazgos devolvió el agente, y cuántos comprobaste tú abriendo el archivo**: 21/5
**El hallazgo del que no supiste decidir la prioridad**: No he decidido nada, simplemente he ido leyendo el orden de la tabla. Aparentemente la precedencia la respeta (por los hallazgos a los que hace referencia). A simple vista el H2 no lo hubiese puesto tan arriba.
**Algo que el agente afirmó y no pudiste verificar**: EL borrado, el H2. No es que no haya podido verificar. Pero no me he puesto a comprobar que el registro existía en BD, y a ejecutar el curl de borrado, para comprobar que sucede.
