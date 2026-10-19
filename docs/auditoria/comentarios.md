# Auditoría — feature de comentarios

Alcance: crear (`POST /posts/:slug/comentarios`), editar (`PATCH /comentarios/:id`), borrar
(`DELETE /comentarios/:id`) y listar (bloque `comentarios` de `GET /posts/:slug` y los recuentos
`num_comentarios`) en API, base de datos (migración + semilla) y web (`frontend/src`).
Solo lectura: no se ha modificado código ni tests.

Orden: primero por precedencia (lo que desbloquea o abarata otro hallazgo va antes) y, dentro de
cada nivel, por daño entre radio. Las casillas en blanco no se pueden confirmar leyendo el código.

| # | Qué es | Dónde está | Qué se rompe y quién lo nota | Cuántos sitios hay que tocar | ¿Se deshace con un revert? | ¿Bloquea o desbloquea a otro? |
|---|---|---|---|---|---|---|
| 1 | La única prueba de comentarios solo afirma el status 201 y el nombre, así que no hay red para tocar nada de la feature. | `backend/tests/functional/comentarios.spec.ts:41-42` `respuesta.assertStatus(201)` y `assert.equal(respuesta.body().comentario.autor_nombre, 'Lucía Ferrer')` (y `.claude/hooks/guard-tests.mjs:32` `const COBERTURA_MINIMA = 0`) | Nadie todavía; cualquier cambio en 2–10 puede romper el contrato sin que falle nada. | 1: `comentarios.spec.ts` (caracterizar POST, PATCH, DELETE y la lista de `GET /posts/:slug`, con aserciones negativas de `autor_email`). | Sí. | Desbloquea 2–10 (el repo exige test antes de cada cambio). |
| 2 | **Contención:** el detalle de un post devuelve el correo de cada comentarista a cualquier lector. | `backend/start/routes.ts:153` `"select * from comentarios where post_id = ? and estado = 'aprobado' ..."` devuelto tal cual en `:171` `comentarios: comentarios,`; la semilla incluye a la propia autora (`blog_seeder.ts:495` `autor_email: 'marta.ruiz@corriente.es'`) | Quien comenta (y la autora que responde): su correo es público para cualquiera que abra el post. | 1: `routes.ts:153` (columnas explícitas sin `autor_email`). | Sí el código; los correos ya servidos no se recuperan. | Abarata 6 (el correo es la credencial de borrado). Su arreglo de fondo es 7. No espera a 1: es contención y va hoy, aunque no haya tests. |
| 3 | `estado` es texto libre y la semilla guarda comentarios aprobados como `'Aprobado'` y `'ok'`. | `backend/database/migrations/1769000000005_create_comentarios_table.ts:18` `table.string('estado').notNullable().defaultTo('pendiente')`; `blog_seeder.ts:510` `estado: 'Aprobado'`, `:522` `estado: 'ok'`, `:575` `estado: 'Aprobado'` | Lector y quien comenta: los comentarios 3, 4 y 9, ya moderados, no aparecen en el detalle (`routes.ts:153` compara `= 'aprobado'` exacto). | 2: una migración nueva (normalizar valores + restricción) y la semilla (`:510`, `:522`, `:575`). | No: el `UPDATE` que normaliza pierde si el valor original era `'Aprobado'` u `'ok'`, y un `down()` no puede reconstruirlo. | Desbloquea 9. Lo bloquea 1. |
| 4 | La clave de moderación que permite borrar cualquier comentario está escrita en el código fuente. | `backend/start/routes.ts:25` `const SECRETO_SESION = 'dev-secret-cambiar'` usada en `:351` `const esModerador = clave === SECRETO_SESION` | Quien comenta: cualquiera con acceso al repo borra cualquier comentario enviando `x-clave-moderacion`. | 3: `routes.ts:25`, `backend/start/env.ts` (declarar la variable) y `.env` (darle valor). | Sí el código; la clave ya está en el historial de git y hay que rotarla. | A ninguno. No espera a 1: rotar la clave es contención y va hoy. |
| 5 | Crear un comentario devuelve la última fila de la tabla, no la que se acaba de insertar. | `backend/start/routes.ts:329-331` `'select * from comentarios order by id desc limit 1'` | Nadie todavía en la web (`Comentarios.jsx:37` solo mira si existe `d.comentario`); con dos envíos simultáneos, un cliente de la API recibe el comentario ajeno, correo incluido. | 1: `routes.ts:329-331`. | Sí. | A ninguno. Lo bloquea 1. |
| 6 | Para borrar un comentario basta con enviar el correo de quien lo escribió. | `backend/start/routes.ts:350-353` `const email = request.input('autor_email', '')` … `String(comentario.autor_email).toLowerCase().trim() === String(email).toLowerCase().trim()` | Quien comenta: mientras 2 siga abierto, cualquier lector puede leer el correo en el detalle y borrar el comentario. | 2 como mínimo: `routes.ts:350-353` (DELETE) y `:378-379` (PATCH), que usan la misma credencial. | | Lo abarata 2. Lo bloquea 1. |
| 7 | **Arreglo de fondo de 2:** ningún endpoint de comentarios usa un transformer; todos devuelven la fila cruda de `select *`. | `backend/start/routes.ts:171` `comentarios: comentarios,`, `:333` `return response.status(201).send({ comentario: nuevos[0] })`, `:392` `return response.send({ comentario: actualizados[0] })` | Quien comenta: `autor_email`, `estado` y `post_id` salen en las tres respuestas. | 4: las tres respuestas (`routes.ts:171`, `:333`, `:392`) y un transformer nuevo en `app/transformers/`. | Sí. | Lo bloquea 1. |
| 8 | Al enviar un comentario, el recuento de la sección sube en 1 aunque el comentario queda pendiente y no se muestra. | `frontend/src/componentes/Comentarios.jsx:39` `setTotal(total + 1)` (la cabecera, `DetallePost.jsx:38` `setNumComentarios((d.comentarios \|\| []).length)`, no se actualiza) | Quien comenta: ve «N+1 comentarios» sobre N visibles, y la cabecera del artículo muestra un número distinto. | 1: `Comentarios.jsx:39`. | Sí. | A ninguno. |
| 9 | El listado y los relacionados cuentan como comentarios todo lo que no es `'pendiente'`, mientras que el detalle solo muestra `'aprobado'`. | `backend/start/routes.ts:75-76` `c.estado <> 'pendiente'`, `:223` `estado <> 'pendiente'` frente a `:153` `estado = 'aprobado'` | Lector: el post 1 de la semilla anuncia 4 comentarios en el listado (`ListaPosts.jsx:57`) y al abrirlo enseña 2. | 3: `routes.ts:75-76`, `:153` y `:223` (una sola definición de «aprobado»). | Sí. | Lo bloquea 3 (hasta que los valores estén normalizados no hay comparación correcta). |
| 10 | Un correo sin `@` se rechaza con la clave `errores`, que el formulario no lee. | `backend/start/routes.ts:317` `return response.status(400).send({ errores: ['email no válido'] })`; el front solo lee `d.error`, `d.message` y `d.motivo` (`Comentarios.jsx:44`) | Quien comenta: ve «No se ha podido enviar.» sin saber que el problema es el correo. | 1: `routes.ts:317` o `Comentarios.jsx:44`. | Sí. | A ninguno. Lo bloquea 1. |

## Propuestas sin medir (fuera de la tabla)

- **Índice en `comentarios.post_id`.** La migración lo admite (`1769000000005_create_comentarios_table.ts:6`
  «post_id no tiene índice ni clave foránea») y tres consultas filtran por esa columna
  (`routes.ts:75`, `:153`, `:223`). Hay que medir: `EXPLAIN QUERY PLAN` de esas consultas y
  la latencia de `GET /posts` y `GET /posts/:slug` con un volumen realista de comentarios.
- **N+1 al contar comentarios de los relacionados.** `routes.ts:222` lanza un `count(*)` por
  relacionado. Hay que medir: consultas por petición y latencia de `GET /posts/:slug/relacionados`;
  con `MAX_RELACIONADOS = 2` puede no compensar.
- **Doble petición del detalle.** `DetallePost.jsx:34` y `Comentarios.jsx:19` piden el mismo
  `GET /posts/:slug`. Hay que medir: peticiones y tiempo hasta pintar los comentarios en la pestaña
  de red del navegador.
- **Spam sin freno en `POST /posts/:slug/comentarios`.** No hay límite de envíos y la semilla ya
  tiene un comentario promocional pendiente (`blog_seeder.ts:551`). Hay que medir: volumen de
  comentarios pendientes y qué proporción es spam antes de decidir entre limitar envíos o añadir
  un captcha.

## Las tres líneas

1. El agente devolvió 13 hallazgos (10 en la tabla y 3 que dejó fuera por el tope de 10); comprobé 10 abriendo el archivo.
2. La fila 2 (los correos de quien comenta salen en el detalle del post): chocaban lo grave (los correos se ven hoy) contra lo que bloquea (sin tests, cualquier cambio es a ciegas). El agente la puso detrás de los tests; yo decidí que la contención va hoy.
3. En la fila 5 el agente afirma que, con dos envíos simultáneos, un cliente de la API recibe el comentario de otra persona, con su correo. Vi la consulta que lo permitiría (`order by id desc limit 1`), pero no pude provocar dos envíos a la vez, así que no sé si pasa de verdad.
