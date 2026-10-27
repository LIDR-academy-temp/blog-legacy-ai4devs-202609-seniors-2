# Auditoría — feature de comentarios

Alcance: crear, editar, borrar y listar comentarios (API, tabla `comentarios`, web). Solo lectura;
no se ha tocado código. Rama de trabajo: `auditoria-ac` (commit `ad34f0e`).

## 1. Mapa de la feature

### Rutas de la API (todas en `backend/start/routes.ts`, handler inline, sin controlador)

| Operación | Verbo + path | Handler (archivo:línea) |
|---|---|---|
| Crear | `POST /posts/:slug/comentarios` | `backend/start/routes.ts:295` |
| Borrar | `DELETE /comentarios/:id` | `backend/start/routes.ts:339` |
| Editar | `PATCH /comentarios/:id` | `backend/start/routes.ts:366` |
| Listar | no hay ruta propia: van dentro de `GET /posts/:slug` | `backend/start/routes.ts:109` (consulta en `:152-155`, salida en `:171`) |
| Recuento (lectura) | dentro de `GET /posts` | `backend/start/routes.ts:74-76` |
| Recuento (lectura) | dentro de `GET /posts/:slug/relacionados` | `backend/start/routes.ts:222-224`, asignado en `:229` |

Secreto de moderación usado por el borrado: `backend/start/routes.ts:25`.

### Tabla

- Migración: `backend/database/migrations/1769000000005_create_comentarios_table.ts:12-21`
  (columnas `id, post_id, autor_nombre, autor_email, cuerpo, estado, creado_en, padre_id`).
- Esquema Lucid generado: `backend/database/schema.ts:73-92` (`ComentarioSchema`). No hay modelo en
  `backend/app/models/` para comentarios.
- Semilla: `backend/database/seeders/blog_seeder.ts:128` (insert) y `:479` (datos `COMENTARIOS`).

### Web (`frontend/src/componentes/`)

| Componente | Qué hace con comentarios |
|---|---|
| `Comentarios.jsx` | Pide `GET /posts/:slug` (`:19`), pinta hilo raíz + respuestas (`:50-74`), formulario que manda `POST` (`:30-34`), recuento propio (`:11`, `:39`, `:54`). |
| `DetallePost.jsx` | Recuento propio en cabecera (`:30`, `:38`, `:71`); monta `<Comentarios>` (`:123`). |
| `ListaPosts.jsx` | Muestra `num_comentarios` (`:57`). |
| `PostsRelacionados.jsx` | Muestra `num_comentarios` (`:32`). |

La web no tiene UI para editar ni borrar comentarios (grep de `PATCH`/`DELETE`/`comentarios/` en
`frontend/src`: solo aparecen las llamadas de `Comentarios.jsx:19` y `:30`).

### Tests

- `backend/tests/functional/comentarios.spec.ts`: un único test, camino feliz de crear.

## 2. Hallazgos

Notas de lectura de la tabla:

- «¿Se deshace con un revert?» se refiere a revertir **el arreglo**. Si el daño ya hecho (datos
  borrados, secretos publicados) no vuelve con el revert, se dice en la misma casilla.
- Los recuentos de la SQLite local se sacaron con `sqlite3 -readonly backend/tmp/db.sqlite3`
  (semilla cargada): post 1 `diez-tendencias-…` → listado 4 / detalle 2; post 6
  `que-mide-de-verdad-…` → listado 1 / detalle 0.
- Según `CLAUDE.md`, cualquier cambio de comportamiento exige antes un test de caracterización;
  por eso casi todo depende de #1.

| # | Qué es | Dónde está | Qué se rompe y quién lo nota | Sitios a tocar | ¿Se deshace con un revert? | ¿Bloquea o desbloquea otro? |
|---|---|---|---|---|---|---|
| 1 | Los comentarios solo tienen un test (camino feliz de crear) y afirma un único campo. | `backend/tests/functional/comentarios.spec.ts:15` `test('deja un comentario en un post publicado', async ({ client, assert }) => {` y `:42` `assert.equal(respuesta.body().comentario.autor_nombre, 'Lucía Ferrer')` | Nadie todavía. Pero cualquier arreglo de esta tabla se hace a ciegas: no hay test de editar, borrar, listar, entrada inválida, 404 ni de lo que no debe salir (`autor_email`). | 1: `backend/tests/functional/comentarios.spec.ts` (ampliar; caracterizar `POST :295`, `PATCH :366`, `DELETE :339` y el bloque de comentarios de `GET /posts/:slug :152-171`). | Sí. | Desbloquea a todos (#2–#20). |
| 2 | **Contención:** el detalle de un post devuelve el correo de cada comentarista porque hace `select *`. | `backend/start/routes.ts:153` `"select * from comentarios where post_id = ? and estado = 'aprobado' order by creado_en asc",` (sale por `:171` `comentarios: comentarios,`) | Cualquier visitante o scraper que llame a `GET /posts/:slug` se lleva los correos de quien comenta (p. ej. `ana.cortes@ejemplo.es` en el post 1). Incumple la regla 6. Y ese correo es justo lo que pide `DELETE` para autorizar (#6): con él, cualquiera borra el comentario. | 1: `backend/start/routes.ts:153` (seleccionar `id, autor_nombre, cuerpo, creado_en, padre_id`, que son los campos que usa la web en `frontend/src/componentes/Comentarios.jsx:50-70`). | Sí el código. Los correos ya servidos no se recuperan. | Lo bloquea #1 (test del detalle). Abarata #6 (corta la vía fácil para conseguir el correo). El arreglo de fondo es el transformer de #4. |
| 3 | **Contención:** la clave de moderación que permite borrar cualquier comentario está escrita en el código. | `backend/start/routes.ts:25` `const SECRETO_SESION = 'dev-secret-cambiar'` (usada en `:351` `const esModerador = clave === SECRETO_SESION`) | Quien lea el repo (está en los remotos `origin` y `upstream`; no he comprobado si son públicos) borra cualquier comentario mandando `x-clave-moderacion: dev-secret-cambiar`. Lo nota el autor del comentario cuando desaparece. | 4: `backend/start/routes.ts:25`, `backend/start/routes.ts:351`, `backend/start/env.ts:21-23` (declarar la variable en el bloque App), `backend/.env.example:7-10`. | No del todo: el valor sigue en el historial de git de los dos remotos; hay que dar la clave por quemada y no reutilizarla. | Lo bloquea #1 (test de borrado como moderador). Abarata #6. |
| 4 | Toda la feature vive en handlers inline de `routes.ts` (SQL, validación, autorización y respuesta), sin controlador, validador, servicio, repositorio ni transformer. | `backend/start/routes.ts:295` `router.post('/posts/:slug/comentarios', async ({ params, request, response }) => {` (igual en `:339` y `:366`) | Nadie de fuera directamente. Es la causa de que las reglas se copien a mano y salgan distintas (#13, #14, #9) y de que se devuelvan filas crudas (#2). | 8: `backend/start/routes.ts:295-334`, `:339-361`, `:366-393`, `:152-155`, `:171`, `:74-76`, `:222-224`, `backend/package.json:21` (añadir alias `#repositories/*`). Más archivos nuevos generados con `node ace make:*`. | Sí. | Lo bloquea #1. Desbloquea/abarata #2 (fondo), #6, #9, #10, #13, #14, #15. |
| 5 | **Contención:** el recuento cuenta `estado <> 'pendiente'` y el detalle muestra solo `estado = 'aprobado'`, así que los números no cuadran. | `backend/start/routes.ts:75-76` `'select *, (select count(*) from comentarios c where c.post_id = posts.id and c.estado <> ' +` / `"'pendiente') as num_comentarios from posts" +`; `:223` `"select count(*) as total from comentarios where post_id = ? and estado <> 'pendiente'",`; frente a `:153` `... and estado = 'aprobado' ...` | Lectores: la portada (`ListaPosts.jsx:57`) dice «4 comentarios» en el post 1 y al entrar ven 2; el post 6 dice 1 y no muestra ninguno. Quien escribió los comentarios 3, 4 y 9 (aprobados como `'Aprobado'`/`'ok'`) no ve el suyo publicado. | 3: `backend/start/routes.ts:75-76`, `:153`, `:223` (usar el mismo predicado en los tres; cuál es una decisión de negocio). | Sí. | Lo bloquea #1. El arreglo de fondo es #7. |
| 6 | Borrar un comentario se autoriza comparando con el `autor_email` que manda el propio cliente, sin autenticación. | `backend/start/routes.ts:350` `const email = request.input('autor_email', '')` y `:352-353` `const esAutor =` / `String(comentario.autor_email).toLowerCase().trim() === String(email).toLowerCase().trim()` | Cualquiera que conozca el correo de quien comentó (lo regala #2) borra el comentario. Lo nota el comentarista y el lector. Es borrado físico (`:359`). | 3: `backend/start/routes.ts:350`, `:352-353`, `:355` (pasar a identidad autenticada; el kit ya tiene `middleware.auth()` en `:416`). La decisión de cómo identifica el blog a un comentarista anónimo es previa. | El código sí. Los comentarios ya borrados no: es `delete` físico sin copia. | Lo bloquean #1 y #4. #2 y #3 cortan el daño mientras tanto. Desbloquea #13. |
| 7 | `estado` es texto libre sin CHECK ni enum y la base ya tiene `'Aprobado'` y `'ok'` además de `'aprobado'`. | `backend/database/migrations/1769000000005_create_comentarios_table.ts:18` `table.string('estado').notNullable().defaultTo('pendiente')` | Es la raíz de #5. Hoy afecta a 3 filas (ids 3, 4, 9). Cualquier valor nuevo (`'rechazado'`, `'spam'`) se contaría como visible en el recuento por el `<> 'pendiente'`. | 4: migración nueva (normalizar datos + restricción), `backend/database/seeders/blog_seeder.ts:510` `estado: 'Aprobado',`, `:522` `estado: 'ok',`, `:575` `estado: 'Aprobado',`. | No: la normalización reescribe datos (se pierde el valor original) y en SQLite quitar un CHECK obliga a recrear la tabla. | Lo bloquean #1 y #5 (decidir qué valores cuentan como aprobado). Simplifica #5. |
| 8 | Crear un comentario devuelve «el último comentario de la tabla», no el que se acaba de insertar. | `backend/start/routes.ts:329-331` `const nuevos: any = await db.rawQuery(` / `'select * from comentarios order by id desc limit 1'` | Con dos envíos a la vez, a una persona le puede llegar el comentario de otra, con su correo y su estado. Nadie todavía (no lo he reproducido; se deduce del código: hay un `await` entre el insert de `:324` y este select). | 2: `backend/start/routes.ts:324-327`, `:329-331`. | Sí. | Lo bloquea #1. Lo abarata #4 (repositorio con Lucid devuelve el modelo creado). |
| 9 | Las validaciones de crear devuelven cuatro formatos de error distintos, y la web no lee uno de ellos. | `backend/start/routes.ts:311` `return response.status(400).send({ error: 'El nombre es obligatorio' })`, `:314` `return response.status(422).send({ message: 'Falta el email' })`, `:317` `return response.status(400).send({ errores: ['email no válido'] })`; la web en `frontend/src/componentes/Comentarios.jsx:44` `setAviso(d.error \|\| d.message \|\| d.motivo \|\| 'No se ha podido enviar.')` | Quien comenta con un correo sin `@` ve «No se ha podido enviar.» en vez de «email no válido», porque la web no mira `errores`. | 5: `backend/start/routes.ts:311`, `:314`, `:317`, `:320`, `frontend/src/componentes/Comentarios.jsx:44`. | Sí. | Lo bloquea #1. Lo abarata #4 (validador VineJS + `app/exceptions/handler.ts`). |
| 10 | Un comentario demasiado corto responde `200` con `{ ok: false }` en vez de un código de error. | `backend/start/routes.ts:320` `return response.send({ ok: false, motivo: 'comentario demasiado corto' })` | Un cliente de la API que mire el código HTTP cree que se guardó. La web no lo nota porque decide por `d.comentario` (`Comentarios.jsx:37`). | 2: `backend/start/routes.ts:320`, `frontend/src/componentes/Comentarios.jsx:44` (si cambia la forma del cuerpo de error). | Sí. | Lo bloquea #1 (caracterizar el `200` antes, como pide `CLAUDE.md`). Lo abarata #4. |
| 11 | Borrar un comentario que tiene respuestas deja las respuestas huérfanas, y la web las oculta. | `backend/start/routes.ts:359` `await db.rawQuery('delete from comentarios where id = ?', [params.id])`; la web solo pinta raíces sin `padre_id` en `frontend/src/componentes/Comentarios.jsx:50` `const raices = comentarios.filter((c) => !c.padre_id)` | Si se borra el comentario 1 del post 1, desaparece también de la vista la respuesta 2 (de la autora del post), sin que nadie la haya borrado. Lo nota quien respondió y el lector. | 1: `backend/start/routes.ts:359` (decidir: borrar en cascada, impedir el borrado o reasignar). | El código sí. El comentario padre ya borrado no vuelve. | Lo bloquea #1. Relacionado con #18 (FK de `padre_id`). |
| 12 | Tras enviar un comentario, la web suma 1 al recuento aunque el comentario queda pendiente y no se muestra. | `frontend/src/componentes/Comentarios.jsx:39` `setTotal(total + 1)` | Quien acaba de comentar ve «3 comentarios» con 2 en pantalla, y la cabecera del artículo (`DetallePost.jsx:71`) sigue diciendo 2: dos números distintos en la misma página. Contradice la regla 5. | 1: `frontend/src/componentes/Comentarios.jsx:39`. | Sí. | A ninguno. El fondo está en #17. |
| 13 | Editar usa una regla de autorización distinta a la de borrar: sin clave de moderación y comparando el correo con mayúsculas. | `backend/start/routes.ts:379` `if (comentario.autor_email !== email \|\| comentario.estado !== 'pendiente') {` frente a `:351-353` (borrar: moderador o correo en minúsculas y sin espacios) | Quien escriba su correo con otra capitalización puede borrar su comentario pero no editarlo. Nadie todavía desde la web: no hay UI de editar. Además hereda el mismo agujero que #6 (autoriza con un correo que manda el cliente), limitado a comentarios pendientes. | 2: `backend/start/routes.ts:378-379`, `:350-353` (una sola política para los dos). | Sí. | Lo bloquean #6 (decidir la identidad) y #4. |
| 14 | Editar acepta un cuerpo de 1 carácter, cuando crear exige al menos 5. | `backend/start/routes.ts:384` `if (!cuerpo) {` frente a `:319` `if (!cuerpo \|\| String(cuerpo).length < 5) {` | Nadie todavía (no hay UI de editar). Por API se salta la regla de longitud mínima. | 1: `backend/start/routes.ts:384`. | Sí. | Lo bloquea #1. Lo resuelve gratis #4 (mismo validador). |
| 15 | `padre_id` se guarda tal cual llega, sin comprobar que exista, que sea del mismo post ni que sea una raíz. | `backend/start/routes.ts:308` `const padre = request.input('padre_id', null)` | Nadie todavía desde la web: el formulario no manda `padre_id` (`Comentarios.jsx:33`). Por API se crean respuestas que, aun aprobadas, la web nunca pinta (ver `:50` y `:64`). | 1: `backend/start/routes.ts:308`. | Sí. | Lo bloquea #1. Lo abarata #4. Relacionado con #16 y #18. |
| 16 | La web solo pinta un nivel de respuestas: una respuesta a una respuesta no aparece. | `frontend/src/componentes/Comentarios.jsx:64` `.filter((r) => r.padre_id === c.id)` (solo se aplica sobre raíces) | Nadie todavía: la semilla no tiene segundo nivel y la web no permite responder. | 1: `frontend/src/componentes/Comentarios.jsx:63-72`. | Sí. | A ninguno (o se cierra desde #15 prohibiendo responder a respuestas). |
| 17 | La página de detalle pide `GET /posts/:slug` dos veces y guarda dos recuentos de comentarios independientes. | `frontend/src/componentes/DetallePost.jsx:34` `fetch(API + '/posts/' + encodeURIComponent(slug))` y `frontend/src/componentes/Comentarios.jsx:19` `fetch(API + '/posts/' + encodeURIComponent(slug))` | Dos peticiones por visita y dos números que divergen (#12). Lo nota el lector solo a través de #12. | 7: `frontend/src/componentes/DetallePost.jsx:30`, `:38`, `:71`, `:123`; `frontend/src/componentes/Comentarios.jsx:11`, `:17-26`, `:54`. | Sí. | Desbloquea #12 (lo hace desaparecer). |
| 18 | `post_id` y `padre_id` no tienen clave foránea. | `backend/database/migrations/1769000000005_create_comentarios_table.ts:14` `table.integer('post_id').notNullable()` y `:20` `table.integer('padre_id').nullable()` | Nadie todavía: no hay ruta que borre posts (revisado `routes.ts` entero). Sí permite que existan las huérfanas de #11 y #15. | 1: migración nueva. | No: en SQLite añadir/quitar FK obliga a recrear la tabla, y si ya hay huérfanas la migración falla hasta limpiar datos. | Lo bloquean #11 y #15 (decidir qué hacer con huérfanas antes de imponer la FK). |
| 19 | El correo se valida solo con «contiene `@`». | `backend/start/routes.ts:316` `if (String(email).indexOf('@') === -1) {` | Se aceptan `@` o `a@`. Nadie todavía. | 1: `backend/start/routes.ts:316`. | Sí. | Lo resuelve #4 (validador VineJS). |
| 20 | No hay longitud máxima para nombre ni cuerpo, ni al crear ni al editar. | `backend/start/routes.ts:310` `if (!nombre) {`, `:319` `if (!cuerpo \|\| String(cuerpo).length < 5) {`, `:384` `if (!cuerpo) {` | Un cuerpo enorme se guarda y luego viaja en cada `GET /posts/:slug`. Nadie todavía. Límite real del bodyparser: no comprobado. | 3: `backend/start/routes.ts:310`, `:319`, `:384`. | Sí. | Lo resuelve #4 (validador VineJS). |

### Comprobado y sin problema

- XSS en comentarios: la web pinta `{c.cuerpo}` y `{c.autor_nombre}` como texto de React
  (`frontend/src/componentes/Comentarios.jsx:59-61`, `:68-70`); no hay `dangerouslySetInnerHTML`.
- SQL de comentarios: todas las consultas de comentarios usan `?` con parámetros
  (`backend/start/routes.ts:153`, `:223`, `:325`, `:340`, `:359`, `:367`, `:388`).
- Comentar en un borrador: `POST` filtra `estado = 'publicado'` (`backend/start/routes.ts:297`).

## 3. Ideas sin evidencia (fuera de la tabla)

| Idea | Qué haría falta para que fuera real |
|---|---|
| El recuento del listado irá lento con muchos comentarios: es una subconsulta por post y `post_id` no tiene índice (`migración :14`). | `EXPLAIN QUERY PLAN` de la consulta de `routes.ts:74-80` y medir tiempos con miles de comentarios sembrados. |
| Spam: no hay límite de envíos por IP ni captcha; la semilla ya tiene un comentario de spam (id 7). | Ver si hay limitador en `start/kernel.ts`/middleware (no lo he revisado) y logs de producción con volumen de envíos. |
| La comparación de la clave de moderación con `===` (`routes.ts:351`) es sensible a ataques de tiempo. | Solo importa si la clave deja de estar publicada (#3); habría que medirlo sobre la red real. |
| `key={index}` en las listas de comentarios (`Comentarios.jsx:57`, `:66`) puede descolocar el estado de React al cambiar la lista. | Hoy los ítems no tienen estado propio; haría falta un caso donde la lista cambie en caliente y se vea el fallo. |

## 4. Fuera de alcance (una línea cada uno)

- `GET /posts` y `GET /posts/:slug` devuelven `posts.autor_email` por `select *` (`routes.ts:74`, `:111`), y `GET /posts/:slug` el correo de `autores` (`:119`).
- `GET /autores/:id` lista también los borradores del autor (`routes.ts:284-286`).
- El cuerpo del post se inyecta sin sanear con `dangerouslySetInnerHTML` (`frontend/src/componentes/DetallePost.jsx:92`).

## 5. Estado

**Completa** para el alcance pedido (crear, editar, borrar y listar comentarios: API, tabla y web).
Todas las filas tienen archivo:línea abierto y la línea copiada. Lo que no se comprobó está
dicho en su casilla: #3 (si los remotos son públicos), #8 (no reproducido, deducido del código),
#20 (límite del bodyparser). No se ha ejecutado la suite ni levantado la app.

## Parte B

- **Cuántos hallazgos devolvió el agente:** El agente devolvió 20 hallazgos y yo comprobé 4.
- **El hallazgo del que no supiste decidir la prioridad:** En que momento escribir los tests ya que hay algo mas prioritario de la fuga de correos.
- **Algo que el agente afirmó y no pudiste verificar:** Contención: el recuento cuenta estado <> 'pendiente' y el detalle muestra solo estado = 'aprobado', así que los números no cuadran.