# Auditoría — feature de comentarios

Alcance: crear (`POST /posts/:slug/comentarios`), editar (`PATCH /comentarios/:id`), borrar
(`DELETE /comentarios/:id`) y listar (dentro de `GET /posts/:slug` y los recuentos
`num_comentarios`), en API + DB + web. Posts, autores, categorías y etiquetas quedan fuera.

Contraste: reglas 2 (pendiente oculto), 5 (recuento solo aprobados) y 6 (no exponer correos) del
README.

**Precedencia** usada para ordenar (de mayor a menor):

- **P1** Seguridad y privacidad (regla 6, autorización).
- **P2** Integridad de datos.
- **P3** Reglas de negocio (2 y 5).
- **P4** Contrato de API y lo que ve la web.
- **P5** Red de tests.

Dentro de cada nivel, el orden es daño ÷ radio: primero lo que hace más daño tocando menos sitios.
Cuando un daño alto choca con una precedencia o un coste que no encaja, el hallazgo se parte en
**[contención]** y **[arreglo]**.

Datos de la semilla que se citan: el post 1 tiene 5 comentarios.

| id | estado | línea de la semilla |
|---|---|---|
| 1 | `aprobado` | `blog_seeder.ts:487` |
| 2 | `aprobado`, respuesta a 1 | `blog_seeder.ts:498` |
| 3 | `Aprobado` | `blog_seeder.ts:510` |
| 4 | `ok` | `blog_seeder.ts:522` |
| 5 | `pendiente` | `blog_seeder.ts:532` |

## Tabla

| # | Qué es | Dónde (archivo:línea) | Qué se rompe / quién lo nota | Cuántos sitios hay que tocar | ¿Se deshace con un revert? | ¿Bloquea o desbloquea a otro? |
|---|---|---|---|---|---|---|
| 1 | **P1 [contención]** El detalle público de un post devuelve cada comentario con `select *`, y eso incluye `autor_email` (incumple la regla 6). | `backend/start/routes.ts:153`, `backend/start/routes.ts:171` | El correo de cada comentarista aprobado sale en la respuesta JSON. La web no lo pinta, pero lo ve cualquiera que abra la pestaña de red o llame a la API. Con la semilla, el post 1 expone `ana.cortes@ejemplo.es`. | 1: la consulta de `routes.ts:153` (columnas explícitas sin `autor_email`). La web no usa el campo (`Comentarios.jsx` solo lee `autor_nombre`, `creado_en`, `cuerpo`, `id` y `padre_id`). | El código sí. Los correos ya servidos no: pueden estar copiados o cacheados. | Desbloquea el #3: sin correos públicos, el borrado por correo deja de ser trivial. Se cierra del todo con el #12. |
| 2 | **P1** La clave de moderación está escrita en el código y en un repositorio público, y basta con enviarla en la cabecera `x-clave-moderacion` para borrar cualquier comentario. | `backend/start/routes.ts:25`, `backend/start/routes.ts:349-351` | Cualquiera que lea el repo (está en GitHub, y cada fork del cohorte la tiene) puede borrar cualquier comentario. Hoy no lo nota nadie, hasta que desaparecen comentarios. | 4: `routes.ts:25`, `routes.ts:351`, `backend/start/env.ts` (declarar la variable), `backend/.env.example`. | No. La clave sigue en el historial de git y en los forks. Hay que rotarla; un revert la vuelve a publicar. Los comentarios borrados no vuelven. | A ninguno. |
| 3 | **P1** El `DELETE` da por bueno al autor si quien llama conoce su correo, y ese correo es público por el #1. | `backend/start/routes.ts:352-353` | Cualquier visitante puede leer `autor_email` en `GET /posts/:slug` y borrar con él cualquier comentario aprobado. Lo nota el autor del comentario, que lo ve desaparecer. | **[contención]** 0 sitios propios: queda contenido al cerrar el #1. **[arreglo]** 1 bloque, `routes.ts:349-357`, con una prueba de autoría que no sea un dato que se puede adivinar (p. ej. un token devuelto al crear). Tocaría también `routes.ts:325` y la columna en una migración nueva. | El código sí. Los borrados no: es un `delete` físico (`routes.ts:359`). | Lo bloquea el #1, que tiene que ir antes o a la vez. Su arreglo tiene que ir con el #9 (la misma regla, escrita dos veces). |
| 4 | **P1** El `POST` no devuelve la fila que acaba de insertar sino «la última fila» de la tabla, así que con dos envíos simultáneos puede devolver el comentario de otra persona, correo incluido. | `backend/start/routes.ts:329-331` | Con peticiones concurrentes, una persona recibe en el 201 el comentario y el correo de otra. Hoy no lo nota nadie: hace falta concurrencia (no verificado, ver parte B). | 1: `routes.ts:324-331` (usar el id que devuelve el insert). | Sí. | A ninguno. |
| 5 | **P2** La tabla no tiene clave foránea en `padre_id` y el `POST` acepta cualquier valor: una respuesta puede colgar de un comentario que no existe o de otro post. | `backend/database/migrations/1769000000005_create_comentarios_table.ts:20`, `backend/start/routes.ts:308`, `backend/start/routes.ts:326` | Se guarda un comentario que la web no pinta nunca (`Comentarios.jsx:50`, `Comentarios.jsx:64`) pero que sí entra en los recuentos (`DetallePost.jsx:38`, `Comentarios.jsx:23`). Lo nota quien comenta: envía y su respuesta no aparece aunque se apruebe. | 2: validación en `routes.ts:308` y una migración nueva con la clave foránea. | Validación: sí. Migración: con `down()` sí, salvo que haya que limpiar filas huérfanas, y esas no vuelven. | Lo desbloquea el #17 (tests antes de tocar). |
| 6 | **P2** Al borrar un comentario se quedan huérfanas sus respuestas: no hay cascada ni clave foránea. | `backend/start/routes.ts:359`, `backend/database/migrations/1769000000005_create_comentarios_table.ts:20` | Si se borra el comentario 1 de la semilla, su respuesta (el 2, aprobada) desaparece de la web, porque solo se pintan las raíces y sus hijos (`Comentarios.jsx:50`). Pero sigue en el JSON y en el recuento de la cabecera (`DetallePost.jsx:38`). Lo nota el lector: la cabecera dice «1 comentarios» y no se ve ninguno. | 1 + 1: `routes.ts:359` (decidir si se borran las respuestas, se reasignan o se bloquea el borrado) y la migración del #5. | El código sí. Las filas borradas no. | Comparte migración con el #5. |
| 7 | **P3 [contención]** El listado y los relacionados cuentan los comentarios con `estado <> 'pendiente'`, pero el detalle solo muestra los que tienen `estado = 'aprobado'`, así que los números no cuadran (incumple la regla 5). | `backend/start/routes.ts:75-76`, `backend/start/routes.ts:223`, frente a `backend/start/routes.ts:153` | Con la semilla, el post 1 sale con «4 comentarios» en el listado (`ListaPosts.jsx:57`) y al entrar se ven 2. También contaría cualquier estado futuro (`rechazado`, `spam`). Lo nota el lector. | 3: `routes.ts:76`, `routes.ts:153`, `routes.ts:223` (los tres con el mismo criterio). | Sí. | Lo bloquea una decisión de producto: ¿`Aprobado` y `ok` cuentan como aprobados? Hasta que se tome, la contención es igualar los tres sitios a `= 'aprobado'`. Lo desbloquea el #17. |
| 8 | **P3 [arreglo]** La columna `estado` es texto libre, sin `CHECK` ni enum, y en la semilla ya conviven `aprobado`, `Aprobado` y `ok`. | `backend/database/migrations/1769000000005_create_comentarios_table.ts:18`, `backend/database/seeders/blog_seeder.ts:510`, `:522`, `:575` | Los comentarios 3, 4 y 9 de la semilla no salen en el detalle pero sí en el recuento del listado. Quien los escribió nunca los ve publicados. Lo nota el comentarista. | 3: una migración nueva (normalizar los datos y añadir el `CHECK`) y las 3 filas de la semilla. | **No.** El `UPDATE` que normaliza pierde el valor original (`ok` y `Aprobado` pasan a ser `aprobado`) y `down()` no puede reconstruirlo. | Lo bloquea la decisión del #7. Cuando entra, el #7 queda como único criterio. |
| 9 | **P3** El `PATCH` y el `DELETE` aplican reglas de autoría distintas: el borrado compara el correo sin mayúsculas ni espacios y acepta la clave de moderación; la edición lo compara exacto y no la acepta. | `backend/start/routes.ts:352-353`, frente a `backend/start/routes.ts:378-379` | Quien escribió `Lucia@ejemplo.es` y luego teclea `lucia@ejemplo.es` puede borrar pero no editar (403). El moderador puede borrar pero no editar. Lo nota el comentarista, o el moderador. | 2: los dos bloques (`routes.ts:349-354` y `routes.ts:378-379`). | Sí. | Va con el arreglo del #3 (una sola regla de autoría). |
| 10 | **P3** No hay ninguna ruta que cambie `estado`: un comentario nuevo nace `pendiente` y desde la API nunca puede pasar a aprobado. | `backend/start/routes.ts:325-326` (inserta `'pendiente'`). La única escritura posterior, `backend/start/routes.ts:388`, solo cambia `cuerpo`. | Ningún comentario creado desde la web llega a publicarse sin tocar la base a mano. Lo nota todo el que comenta. | 1 ruta nueva, con su controlador, validador y servicio, si se decide que la moderación va por API. | Sí. | Depende del #2 (cómo se identifica al moderador) y del #8 (qué estados existen). |
| 11 | **P4** El contrato de error del `POST` usa 4 formas distintas y 3 códigos, y un cuerpo demasiado corto devuelve `200`. | `backend/start/routes.ts:311` (400 `{error}`), `:314` (422 `{message}`), `:317` (400 `{errores:[]}`), `:320` (200 `{ok:false, motivo}`) | Un cliente de la API que mire el código HTTP da por bueno un comentario rechazado. La web no lee `errores` (`Comentarios.jsx:44`): con un correo sin `@` el usuario ve «No se ha podido enviar.» sin el motivo. Lo nota el comentarista. | 5: las 4 respuestas de `routes.ts:310-321` y `Comentarios.jsx:44`. | Sí. | Lo bloquea el #17: hay que caracterizar los 4 casos antes. Cambiar el contrato obliga a cambiar `Comentarios.jsx:44` en el mismo cambio. |
| 12 | **P4 [arreglo del #1]** Las tres respuestas de comentarios devuelven la fila cruda, sin transformer: `autor_email`, `estado` y `post_id` salen también en la creación y en la edición. | `backend/start/routes.ts:171`, `backend/start/routes.ts:333`, `backend/start/routes.ts:392` | En la creación y la edición el correo vuelve a quien lo envió, así que hoy no lo nota nadie. Pero cualquier columna nueva de `comentarios` saldría en las tres respuestas sin que nadie lo decida. | 4: un transformer nuevo y las 3 respuestas. | Sí. | Cierra del todo el #1. Lo desbloquea el #17. |
| 13 | **P4** La web lleva dos contadores de comentarios separados, y el de la sección sube al enviar aunque el comentario quede pendiente. | `frontend/src/componentes/DetallePost.jsx:30`, `:38`, `:71`, `frontend/src/componentes/Comentarios.jsx:11`, `:23`, `:39`, `:54` | Tras comentar, la sección dice «3 comentarios», la cabecera sigue en 2 y se ven 2: la UI cuenta un pendiente (contra el espíritu de las reglas 2 y 5). Lo nota quien acaba de comentar. | 7 líneas en 2 archivos. | Sí. | Mejor después del #7: entonces el número correcto puede salir de la API (`num_comentarios`) y no de `length`. |
| 14 | **P4** El `PATCH` solo comprueba que el cuerpo no esté vacío, mientras que el `POST` exige al menos 5 caracteres. | `backend/start/routes.ts:384-386`, frente a `backend/start/routes.ts:319` | Un comentario pendiente se puede editar a `"a"`. Lo nota el moderador. | 1, o 0 si se reutiliza el validador del `POST` cuando exista. | Sí. | Se resuelve con el validador VineJS compartido tras el #11. |
| 15 | **P4** La validación del `POST` es mínima: para el correo solo busca una `@`, y el largo del cuerpo se mide sin recortar espacios y sin máximo. | `backend/start/routes.ts:316`, `backend/start/routes.ts:319` | Se aceptan `@` como correo y `"     "` como cuerpo, y también cuerpos de cualquier tamaño. Lo nota el moderador, que recibe basura. | 2 (`routes.ts:316` y `routes.ts:319`), o 1 validador VineJS. | Sí. | Va con el #11 (mismo validador). |
| 16 | **P4** Si falla la carga de comentarios, el error se traga sin aviso. | `frontend/src/componentes/Comentarios.jsx:25` | Si la API no responde, la sección dice «0 comentarios» como si no hubiera ninguno. Lo nota el lector, que no sabe que falló. | 1. | Sí. | A ninguno. |
| 17 | **P5** Solo hay un test de comentarios, y comprueba el estado 201 y un campo: ni el `PATCH`, ni el `DELETE`, ni la lista, ni ningún caso de error. | `backend/tests/functional/comentarios.spec.ts:15-43` (aserciones en `:41-42`) | Hoy no lo nota nadie. Cualquier refactor o arreglo de los hallazgos #1-#16 puede romper el contrato sin que la suite se entere. | 1 archivo: `comentarios.spec.ts`. La matriz (feliz, entrada inválida, 404, frontera) para 3 rutas más la lista, con aserciones negativas de `autor_email`. | Sí. | **Desbloquea a todos.** Según `CLAUDE.md`, la caracterización va antes de tocar el #1, #3, #4, #5, #7, #9, #11 y #12. |

## Fuera de la tabla: opiniones y propuestas sin medir

- **`post_id` sin índice** (`1769000000005_create_comentarios_table.ts:14`). Puede hacer lento el
  subselect de `routes.ts:75` y la consulta por cada relacionado de `routes.ts:222`. *Para medir:*
  `EXPLAIN QUERY PLAN` en esas consultas y tiempo de `GET /posts` con 10⁴–10⁵ comentarios.
- **`Comentarios.jsx` vuelve a pedir el post entero** (`Comentarios.jsx:19`), una petición que
  `DetallePost.jsx:34` ya hace. *Para medir:* pestaña de red, número de peticiones y bytes por
  cada visita a un detalle.
- **`key={index}`** en las listas (`Comentarios.jsx:57`, `:66`). Podría causar re-renders
  incorrectos si cambia el orden. *Para medir:* reproducir borrando o insertando sin recargar. Hoy
  la lista no cambia en caliente, así que puede que no importe.
- **No hay límite de envío (anti-spam)** en el `POST`. *Para medir:* número de comentarios por IP
  y por minuto en los logs de producción, si existen.
- **El correo del `DELETE` se lee con `request.input`** (`routes.ts:350`), que también acepta
  query string. Si un cliente lo manda en la URL, el correo acabaría en los logs de acceso.
  *Para medir:* revisar qué envían los clientes reales y qué registra el proxy.
- **Visto pero no auditado (fuera de alcance):** el detalle devuelve también `post.autor_email`
  con `select * from posts` (`routes.ts:110-112`), lo que afecta a la regla 6. Pertenece a la
  feature de posts.

## Verificación de los hallazgos

Primera tanda: #1, #2, #3, #7 y #17. Segunda tanda: #4, #5, #6, #8, #9, #10, #11 y #12.

Cada archivo:línea citado se volvió a abrir.

- **#1 — confirmado.** `routes.ts:153`: `"select * from comentarios where post_id = ? and estado = 'aprobado' ..."`,
  y la tabla tiene `table.string('autor_email')` (`1769000000005_create_comentarios_table.ts:16`).
  `routes.ts:171` lo devuelve tal cual (`comentarios: comentarios`). El comentario 1 de la semilla
  es del post 1, que está publicado (`blog_seeder.ts:177`), y su correo es
  `ana.cortes@ejemplo.es` (`blog_seeder.ts:484`).
- **#2 — confirmado.** `routes.ts:25`: `const SECRETO_SESION = 'dev-secret-cambiar'`, y en
  `routes.ts:349` y `:351`: `request.header('x-clave-moderacion')` … `esModerador = clave === SECRETO_SESION`.
  `gh repo view` informa `"visibility":"PUBLIC"` (fork de `LIDR-academy-temp/...`).
- **#3 — confirmado.** `routes.ts:350`: `request.input('autor_email', '')`, y `routes.ts:353`:
  `String(comentario.autor_email).toLowerCase().trim() === String(email).toLowerCase().trim()`.
  El borrado es físico (`routes.ts:359`: `delete from comentarios where id = ?`).
- **#7 — confirmado.** `routes.ts:75-76`: `c.estado <> 'pendiente') as num_comentarios`, y
  `routes.ts:223`: `estado <> 'pendiente'`, frente a `routes.ts:153`: `estado = 'aprobado'`. En la
  semilla, el post 1 tiene los estados `aprobado`, `aprobado`, `Aprobado`, `ok` y `pendiente`
  (`blog_seeder.ts:487-532`): la cuenta da 4 y el detalle muestra 2.
- **#17 — confirmado, con las líneas corregidas.** Hay un único `test(` en
  `comentarios.spec.ts:15` y ningún otro archivo de tests en `backend/tests/` menciona
  comentarios. Las aserciones son `:41` `respuesta.assertStatus(201)` y `:42`
  `assert.equal(respuesta.body().comentario.autor_nombre, 'Lucía Ferrer')`. La fila citaba
  `:15-45` / `:43-44` y el archivo termina en la línea 44.
- **#4 — código confirmado; el efecto, no reproducido.** `routes.ts:324` hace el `insert` y
  `routes.ts:330` relee con `'select * from comentarios order by id desc limit 1'`, sin filtrar
  por el id insertado. Que dos peticiones se intercalen entre esos dos `await` no se ha probado
  con carga (ver Parte B, línea 3).
- **#5 — confirmado.** `routes.ts:308`: `const padre = request.input('padre_id', null)` pasa
  directo al insert (`routes.ts:326`), y en `routes.ts:310-321` no hay ningún `if` sobre `padre`.
  La migración (`:20`) declara `table.integer('padre_id').nullable()` sin `.references()`.
  `Comentarios.jsx:50` (`filter((c) => !c.padre_id)`) y `:64`
  (`filter((r) => r.padre_id === c.id)`) solo pintan las respuestas cuyo padre está en la lista.
- **#6 — confirmado, con el efecto corregido.** `routes.ts:359` borra solo la fila
  (`delete from comentarios where id = ?`) y la migración no tiene `.references()` ni cascada
  (`:20`). La fila decía «la cabecera dice 2 y se ve 1». Tras borrar el comentario 1, los
  aprobados del post 1 se quedan en `[2]`, así que `DetallePost.jsx:38` (`(d.comentarios || []).length`)
  pinta 1. Y como el 2 tiene `padre_id: 1`, que ya no está, `Comentarios.jsx:50` no lo pinta: se ven 0.
- **#8 — confirmado.** Migración `:18`: `table.string('estado').notNullable().defaultTo('pendiente')`,
  sin `CHECK`. La semilla tiene `estado: 'Aprobado'` (`blog_seeder.ts:510`, comentario 3, post 1),
  `estado: 'ok'` (`:522`, comentario 4, post 1) y `estado: 'Aprobado'` (`:575`, comentario 9,
  post 6, publicado en `:343`).
- **#9 — confirmado.** `routes.ts:353` normaliza (`.toLowerCase().trim()`) y `:351` acepta la
  clave. En cambio `routes.ts:379`: `if (comentario.autor_email !== email || comentario.estado !== 'pendiente')`,
  con comparación exacta y sin clave de moderación en todo el bloque `:366-393`.
- **#10 — confirmado.** La única escritura sobre comentarios después del insert es
  `routes.ts:388`: `'update comentarios set cuerpo = ? where id = ?'`. `grep` no encuentra
  `comentarios` en `backend/app/` ni en `backend/commands/`. El insert fija `'pendiente'`
  (`routes.ts:326`).
- **#11 — confirmado.** `routes.ts:311`: `status(400).send({ error: ... })`; `:314`:
  `status(422).send({ message: ... })`; `:317`: `status(400).send({ errores: [...] })`; `:320`:
  `response.send({ ok: false, motivo: ... })`, que sale con 200. `Comentarios.jsx:44`:
  `d.error || d.message || d.motivo || 'No se ha podido enviar.'` no lee `errores`.
- **#12 — confirmado.** `routes.ts:171` (`comentarios: comentarios`, de un `select *`), `:333`
  (`{ comentario: nuevos[0] }`, de un `select *`) y `:392` (`{ comentario: actualizados[0] }`,
  de un `select *`). Ninguna pasa por un transformer.

## Parte B — las tres líneas

1. **Hallazgos que devolvió el agente:** 17 filas en la tabla (los #1/#12 y #7/#8 son pares
   contención/arreglo). **Comprobados abriendo el archivo:** 13 de 17 (#1–#12 y #17). En el #4
   se comprobó el código pero no el efecto de la carrera, y en dos filas había errores que se
   corrigieron: las líneas del #17 y el efecto del #6. Faltan por comprobar el #13, #14, #15 y
   #16.
2. **El hallazgo del que no supe decidir la prioridad:** el #10 (ninguna ruta aprueba
   comentarios). Puede ser una regla de negocio rota (P3) o una decisión deliberada de moderar
   fuera de la API, y en el repo no hay nada que diga cuál de las dos es.
3. **Algo que el agente afirmó y no pude verificar:** la carrera del #4. Que dos `POST` se
   intercalen entre el `insert` (`routes.ts:324`) y el `select ... order by id desc`
   (`routes.ts:330`) se deduce leyendo el código (dos `await` seguidos), pero no lo he reproducido
   con carga concurrente sobre `better-sqlite3`. Tampoco pude comprobar lo que dice el comentario
   de `routes.ts:21`, que `SECRETO_SESION` «firma las sesiones»: en el repo solo se usa en
   `routes.ts:351`, y la configuración de cifrado usa `APP_KEY` (`config/encryption.ts:16`).
