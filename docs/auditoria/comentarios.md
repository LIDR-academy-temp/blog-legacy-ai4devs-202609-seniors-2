# Auditoría de la feature «Comentarios»

**Alcance.** Solo la feature de comentarios, en sus tres capas:

- **API**: `POST /posts/:slug/comentarios`, `PATCH /comentarios/:id`, `DELETE /comentarios/:id`, y
  las tres lecturas que exponen comentarios o su recuento (`GET /posts/:slug`, `GET /posts`,
  `GET /posts/:slug/relacionados`).
- **Base de datos**: la tabla `comentarios` (migración, esquema generado y semilla).
- **Web**: `Comentarios.jsx` y lo que `DetallePost.jsx`, `ListaPosts.jsx` y
  `PostsRelacionados.jsx` enseñan de los comentarios.

No se audita nada más de posts, autores, categorías ni etiquetas. Solo se entra cuando una relación
con comentarios lo exige.

**Contraste.** Las reglas de negocio del `README.md` (líneas 59-65), sobre todo la 2 (pendiente
oculto), la 5 (recuento solo de aprobados) y la 6 (sin correos en la respuesta pública).

**Cómo se comprobó.** Todos los hallazgos salen de leer el código con la línea citada. Los marcados
con **[API viva]** se comprobaron además contra la API levantada en `localhost:3401` con la semilla
cargada, solo con peticiones `GET` (no se escribió nada en la base de desarrollo). Los flujos de
escritura (crear, editar, borrar) se verificaron leyendo el código, no ejecutándolos.

**Prioridad.** Crítica = explotable hoy por cualquier visitante. Alta = viola una regla de negocio
con datos reales o deja la puerta abierta a terceros. Media = defecto real sin víctima inmediata.
Baja = deuda o calidad.

## Hallazgos

| # | Prio. | 1. Qué es | 2. Dónde está (línea que lo demuestra) | 3. Qué se rompe, y quién lo nota | 4. Sitios a tocar | 5. ¿Se deshace con un revert? | 6. ¿Bloquea / desbloquea a otro? |
|---|---|---|---|---|---|---|---|
| C1 | Crítica | El detalle público de un post devuelve el correo de cada persona que comenta. | `backend/start/routes.ts:153` — `"select * from comentarios where post_id = ? and estado = 'aprobado' …"` y se devuelve tal cual en `:171` (`comentarios: comentarios`). **[API viva]** `GET /posts/diez-tendencias-…` devuelve `autor_email: 'ana.cortes@ejemplo.es'`. | Viola la regla 6. Cualquier visitante, o cualquier scraper, obtiene el correo de todo el que ha comentado algo aprobado. Lo notan quienes comentan (spam), y el blog, que queda expuesto ante el RGPD. | 1 (la consulta de `:153`, con columnas explícitas o un transformer). | Sí: es solo código. Pero el revert no retira los correos ya servidos o cacheados. | **Desbloquea C2**: mientras el correo sea público, la autorización del DELETE no protege nada. Lo resuelve de raíz C15. |
| C2 | Crítica | Para borrar un comentario basta con conocer el correo de su autor, y ese correo es público por C1. | `routes.ts:350-353` — `const email = request.input('autor_email', '')` … `String(comentario.autor_email).toLowerCase().trim() === String(email).toLowerCase().trim()`. | Cualquier visitante lee el correo en `GET /posts/:slug` y llama a `DELETE /comentarios/:id` con ese correo. Así borra cualquier comentario aprobado de cualquier persona. Lo nota quien comentó, porque su comentario desaparece, y el blog pierde contenido. | 1 si solo se cierra C1. Son 4 si se introduce una prueba de propiedad real (token por comentario): migración nueva, guardarlo en el POST `:324-327`, comprobarlo en el DELETE `:350-353` y en el PATCH `:378-379`. | Si es solo código, sí. Con la migración del token, no limpiamente: el `down` borra los tokens ya emitidos. | **Bloqueado por C1** (mitigación inmediata). Va junto a **C9**, que es la misma regla de identidad. |
| C3 | Alta | Tras insertar, el POST devuelve «el último comentario de la tabla», que no tiene por qué ser el suyo. | `routes.ts:329-331` — `'select * from comentarios order by id desc limit 1'`. | Con dos envíos casi simultáneos, el 201 de uno devuelve la fila del otro: su correo, su texto y su `padre_id`, aunque esté pendiente. Nadie lo nota todavía (el tráfico es bajo), pero es una fuga de datos de un tercero (regla 6). | 1 (usar el id que devuelve el insert, o `insert … returning`). | Sí. | A ninguno. Queda aún más acotado con C15 (transformer). |
| C4 | Alta | La clave de moderación está escrita en el código fuente y versionada en git. | `routes.ts:25` — `const SECRETO_SESION = 'dev-secret-cambiar'`. Se usa en `:351` — `clave === SECRETO_SESION`. | Cualquiera con acceso al repo (que es un fork público del curso) borra cualquier comentario, incluidos los pendientes, enviando la cabecera `x-clave-moderacion: dev-secret-cambiar`. Nadie lo nota hasta que alguien lo usa. | 3: `routes.ts:25`, `routes.ts:351` y declararla en `start/env.ts`. | El cambio de código, sí. La clave ya filtrada en el historial **no**: hay que rotarla. | A ninguno. Nota: el comentario de `:21-23` dice que esta clave también firma las sesiones, y es falso: el cifrado usa `APP_KEY` (`config/encryption.ts:16`). |
| C5 | Alta | `comentarios.estado` es texto libre, sin CHECK ni enum, y en los datos ya conviven `'aprobado'`, `'Aprobado'` y `'ok'`. | Migración `1769000000005_create_comentarios_table.ts:18` — `table.string('estado').notNullable().defaultTo('pendiente')`. Semilla `blog_seeder.ts:510` (`'Aprobado'`), `:522` (`'ok'`), `:575` (`'Aprobado'`). **[API viva]** `sqlite3`: post 1 tiene 1 `Aprobado`, 2 `aprobado`, 1 `ok` y 1 `pendiente`. | Cada consulta interpreta el estado a su manera, y de ahí salen C6 y C7. Cualquier estado nuevo (`'rechazado'`, `'spam'`) se cuela en los recuentos. Lo notan quienes comentan (C6) y los lectores (C7). | 2 migraciones nuevas: normalizar los datos y añadir el CHECK. Además, las 3 filas de la semilla. | **No**: la normalización de datos no tiene vuelta atrás, porque el `down` no sabe qué fila era `'Aprobado'` y cuál `'ok'`. | **Desbloquea C6 y C7**: con el dato normalizado, las dos consultas se arreglan sin casos especiales. |
| C6 | Alta | El detalle solo muestra el estado `'aprobado'` exacto, así que tres comentarios aprobados legítimamente no salen nunca. | `routes.ts:150-153` — el propio comentario lo admite: «los que se guardaron como 'Aprobado' o como 'ok' no salen por aquí». **[API viva]** `GET /posts/que-mide-de-verdad-una-metrica-de-producto` devuelve 0 comentarios, cuando el post 6 tiene uno `Aprobado`. | Quienes comentaron no ven su comentario aprobado: Rubén Lima (id 3), Silvia Peña (id 4) y Paula Merino (id 9). Los lectores ven menos conversación de la que hay. | 1 (`routes.ts:153`). | Sí. | **Bloqueado por C5**: o se normaliza el dato, o la consulta tiene que tolerar las tres grafías. |
| C7 | Alta | El recuento de comentarios cuenta todo lo que no sea `'pendiente'`, en lugar de solo los aprobados, y no cuadra con lo que enseña el detalle. | `routes.ts:75-76` (listado) — `c.estado <> 'pendiente'`; `routes.ts:223` (relacionados) — `estado <> 'pendiente'`. **[API viva]** El post 1 marca `num_comentarios: 4` en `GET /posts?pagina=2`, pero su detalle muestra 2. El post 6 marca 1 y su detalle muestra 0. | Viola la regla 5. El lector ve «4 comentarios» en la portada, entra y encuentra 2. Lo nota cualquier lector. | 2 (`:75-76`, `:223`). | Sí. | **Bloqueado por C5**: si se cambia a `= 'aprobado'` sin normalizar, los recuentos bajan de golpe (el post 1 pasa de 4 a 2). **Desbloquea C16**. |
| C8 | Media | El PATCH acepta un cuerpo de 1 carácter, mientras que crear exige al menos 5. | `routes.ts:384` — `if (!cuerpo)`, frente a `:319` — `String(cuerpo).length < 5`. | Se puede saltar la regla de longitud mínima creando un comentario válido y editándolo después. Nadie lo nota todavía, porque la web no tiene edición (C17). | 1 (`:384`, o compartir un único validador VineJS con el POST). | Sí. | Se resuelve junto a C10 (un validador común). |
| C9 | Media | Editar y borrar usan reglas de autorización distintas para el mismo recurso. | `routes.ts:375-379` — `comentario.autor_email !== email \|\| comentario.estado !== 'pendiente'`: distingue mayúsculas, no admite la clave de moderación y solo deja editar pendientes. Frente a `:349-355`: no distingue mayúsculas y sí admite moderador. | Quien escribió su correo con otra capitalización puede borrar su comentario, pero no editarlo. El moderador puede borrar, pero no editar. Una vez aprobado, el autor ya no puede corregir una errata. Lo nota quien comenta. | 2 (`:349-355`, `:375-381`), o 1 si se extrae una única política. | Sí. | Va junto a **C2** (la misma decisión de identidad). |
| C10 | Media | Los errores de validación del POST salen en cuatro formatos y códigos distintos, y el cuerpo corto responde 200. | `routes.ts:310-321` — `400 {error}` (`:311`), `422 {message}` (`:314`), `400 {errores: [...]}` (`:317`), y `200 {ok: false, motivo}` (`:320`). | Un cliente de la API que mire el código HTTP cree que el comentario corto se guardó. La web lo enmascara leyendo los tres campos (`Comentarios.jsx:44`), así que solo lo notan los integradores. | 5: las 4 ramas de `:310-321` y `Comentarios.jsx:44`. | Sí. | Cambiar el `200` es un cambio de contrato: requiere antes su test de caracterización (C22). |
| C11 | Media | El correo se valida solo con la presencia de una `@`, y no hay longitud máxima en nombre ni cuerpo. | `routes.ts:316` — `String(email).indexOf('@') === -1`; `:319` solo comprueba el mínimo. SQLite no aplica el `varchar(255)` de `table.string` (migración `:15-16`). | Se aceptan correos como `@`, y textos de cualquier tamaño en nombre y cuerpo, que la web pinta enteros. Nadie lo nota todavía. | 2 (`:316`, `:319`). | Sí. | Se resuelve junto a C10. |
| C12 | Media | `padre_id` se guarda sin validar: puede apuntar a un comentario inexistente, de otro post, pendiente, o a una respuesta. | `routes.ts:308` — `request.input('padre_id', null)`, insertado sin comprobar en `:326`. La web solo pinta un nivel: `Comentarios.jsx:50` (raíces) y `:63-64` (hijos directos). | Una respuesta a una respuesta, o a un padre que no está aprobado, se guarda pero no aparece nunca en la web. Nadie lo nota todavía, porque la web no envía `padre_id` (`Comentarios.jsx:33`). | 1 (validar en el POST, `:308`). | Sí. | **Bloqueado por / desbloqueado con C13** (la FK valida la existencia). |
| C13 | Media | `post_id` y `padre_id` no tienen clave foránea ni índice, y el borrado es físico. | Migración `:14` y `:20` (`table.integer(...)` sin `references`). El propio comentario de `:6` lo confirma: «post_id no tiene índice ni clave foránea». `routes.ts:359` — `delete from comentarios where id = ?`. | Al borrar un comentario, sus respuestas quedan huérfanas y la web las oculta: no son raíz (`Comentarios.jsx:50`) y su padre ya no está. Ejemplo con la semilla: borrar el id 1 hace desaparecer la respuesta id 2. Lo nota quien respondió. | 2: migración nueva con las FK e índices, y `routes.ts:359` (borrado en cascada o lógico). | **No**: si ya hay huérfanos, la migración con FK obliga a limpiarlos antes, y esa limpieza no se deshace. | **Desbloquea C12**. |
| C14 | Media | No existe ningún endpoint para aprobar o rechazar un comentario: los pendientes solo salen de ese estado tocando la base a mano. | `routes.ts` define solo tres rutas de escritura: `:295` (POST), `:339` (DELETE) y `:366` (PATCH). Ninguna cambia `estado`. Migración `:4-6`: «'Aprobado' … y 'ok' desde un script». | El moderador no tiene flujo, y todo comentario nuevo se queda pendiente para siempre salvo intervención en SQL. Esos caminos manuales son los que crearon los estados raros de C5. Lo nota el moderador, y quien comenta y nunca ve su comentario. | 0 sitios existentes: hace falta una ruta nueva y su lógica. | Sí. | **Desbloquea C5**: un único camino de escritura del estado impide que vuelvan los datos sucios. |
| C15 | Media | Las tres respuestas que devuelven comentarios entregan la fila cruda de la base, sin transformer. | `routes.ts:171` (`comentarios: comentarios`), `:333` (`{ comentario: nuevos[0] }`) y `:392` (`{ comentario: actualizados[0] }`). | Salen `autor_email`, `estado` y `post_id` sin que el contrato los pida (CLAUDE.md, «Reglas que no se negocian»). Es la causa estructural de C1 y C3. Lo nota cualquier consumidor de la API. | 3 (`:171`, `:333`, `:392`) y un transformer nuevo. | Sí. | **Desbloquea C1 y C3**. |
| C16 | Media | La web mantiene dos recuentos de comentarios independientes que divergen entre sí y con el del listado. | `DetallePost.jsx:30`, `:38` y `:71` (cabecera); `Comentarios.jsx:11`, `:23`, `:39` y `:54` (sección). `ListaPosts.jsx:57` y `PostsRelacionados.jsx:32` usan el `num_comentarios` de la API. | Tras comentar, la sección suma 1 por un comentario pendiente que no se muestra, y la cabecera no cambia: el mismo artículo dice «2» y «3». La portada dice «4» (C7). Lo nota cualquier lector que comente. | 2 componentes (7 líneas: las 3 de `DetallePost.jsx` y las 4 de `Comentarios.jsx`). | Sí. | **Bloqueado por C7**: sin un recuento coherente en la API no hay una fuente única que compartir. |
| C17 | Baja | La web solo permite crear y listar: no ofrece editar, borrar ni responder. | `Comentarios.jsx:76-88`: un único formulario de alta. `:33` no envía `padre_id`. No hay ningún `fetch` con `PATCH` ni `DELETE`. | Quien comenta no puede corregir ni retirar su comentario desde la web. Los flujos de `PATCH` y `DELETE` solo existen por API. Lo nota quien comenta. | 1 componente (`Comentarios.jsx`). | Sí. | **Bloqueado por C2 y C9**: no conviene exponer un botón de borrar con la autorización actual. |
| C18 | Baja | El detalle del post se pide dos veces, una de ellas solo para sacar los comentarios. | `Comentarios.jsx:19` y `DetallePost.jsx:34`: el mismo `fetch(API + '/posts/' + …)`. | Doble carga en cada artículo, y es el origen de los dos estados de C16. Nadie lo nota todavía. | 2 (`Comentarios.jsx:17-26`, y pasar los datos por prop desde `DetallePost.jsx:123`). | Sí. | Se resuelve junto a C16. |
| C19 | Baja | Si falla la carga de comentarios, el error se traga y se muestra «0 comentarios». | `Comentarios.jsx:25` — `.catch(() => {})`. | Con la API caída, el lector ve un artículo sin comentarios en lugar de un aviso. Lo nota el lector, sin saberlo. | 1. | Sí. | A ninguno. |
| C20 | Baja | La URL de la API está escrita a mano en el componente. | `Comentarios.jsx:4` — `const API = 'http://localhost:3401'`. El propio comentario dice «Cuarta copia». | Desplegado fuera de local, los comentarios no cargan ni se envían. Nadie lo nota todavía. | 1 en la feature (hay 3 copias más fuera de su alcance, que no se cuentan). | Sí. | A ninguno. |
| C21 | Baja | `creado_en` se guarda en UTC sin marca de zona, y la web lo recorta a la fecha. | `routes.ts:323` — `new Date().toISOString().slice(0, 19).replace('T', ' ')`; `Comentarios.jsx:59` — `.substring(0, 10)`. | Un comentario enviado poco después de medianoche en España aparece con la fecha del día anterior. Lo nota quien comenta de madrugada. | 2 (`:323`, `Comentarios.jsx:59` y `:68`). | Sí. | A ninguno. |
| C22 | Baja | La feature tiene un solo test, que comprueba un único campo del camino feliz del alta. | `backend/tests/functional/comentarios.spec.ts:41-42` — `assertStatus(201)` y `assert.equal(…autor_nombre, …)`. No hay tests de PATCH, DELETE, listado ni recuentos. | Cualquier corrección de C1 a C16 se hace sin red de seguridad, y nadie lo nota hasta que algo se rompe. Lo nota el próximo que refactorice. | 1 archivo (ampliar el `.spec.ts`). | Sí. | **Desbloquea a todos**: CLAUDE.md exige caracterización en verde antes de tocar nada. |

### Comprobaciones sin hallazgo

Lo que se revisó y está bien. Se deja escrito para que no se vuelva a auditar:

- **Regla 2 (pendiente oculto) en el detalle**: se cumple, `routes.ts:153`. **[API viva]** El post 1
  no devuelve el id 5 (pendiente).
- **Comentar en un borrador**: rechazado con 404, `routes.ts:296-302` (filtra `estado = 'publicado'`).
- **XSS en comentarios**: el cuerpo se pinta como texto JSX (`Comentarios.jsx:61` y `:70`), no con
  `dangerouslySetInnerHTML`.
- **Inyección SQL en la feature**: todas las consultas de comentarios usan parámetros `?`.

### Fuera de alcance (anotado, no auditado)

- CSRF desactivado (`config/shield.ts:33`) y CORS abierto en desarrollo (`config/cors.ts:21`).
  Afectan a los tres endpoints de escritura, pero son configuración global, no de la feature.
- La respuesta de `GET /posts/:slug` también expone `post.autor_email` (el correo del autor del
  post). Es la regla 6, pero en la feature de posts.

## Parte B: las tres líneas

1. **Hallazgos que devolvió el agente:** 22 (C1-C22). De ellos, 4 los comprobó además el agente
   contra la API viva (C1, C5, C6 y C7); el resto, solo leyendo el código. **Comprobados por mí
   abriendo el archivo:** _pendiente de rellenar_.
2. **Hallazgo sin prioridad clara:** _pendiente de rellenar_. Candidato propuesto: **C4** (clave en
   el código). Es crítico si el fork es público, pero solo permite lo mismo que ya permite C2 con
   menos esfuerzo, así que su impacto marginal depende de que C1 y C2 se cierren antes.
3. **Algo que el agente afirmó y no pude verificar:** _pendiente de rellenar_. Candidato propuesto:
   la carrera de **C3**. Se deduce del código (`:329-331`), pero no se reprodujo con dos peticiones
   concurrentes, y depende de cómo serialice SQLite las escrituras en `better-sqlite3`.

---

## Anexo: notas previas

### Primera pasada de tests: Toda en verde
functional / Comentarios (tests/functional/comentarios.spec.ts)
✔ deja un comentario en un post publicado (33.21ms)

PASSED

Tests  1 passed (1)
Time  45ms
