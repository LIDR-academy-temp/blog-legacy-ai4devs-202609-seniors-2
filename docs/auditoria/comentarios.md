# Auditoría de la feature «comentarios»

Alcance: las cuatro rutas que tocan comentarios, la tabla `comentarios` y lo que la web enseña de ella.
Nada fuera de eso.

| Pieza | Dónde |
|---|---|
| Lectura (dentro del detalle del post) | `GET /posts/:slug` → `backend/start/routes.ts:152-155` |
| Alta | `POST /posts/:slug/comentarios` → `routes.ts:295-334` |
| Borrado | `DELETE /comentarios/:id` → `routes.ts:339-361` |
| Edición | `PATCH /comentarios/:id` → `routes.ts:366-393` |
| Recuentos (en otros listados) | `routes.ts:75-76` (listado), `routes.ts:222-225` (relacionados) |
| Tabla | `backend/database/migrations/1769000000005_create_comentarios_table.ts` |
| Web | `frontend/src/componentes/Comentarios.jsx`, y los recuentos en `DetallePost.jsx`, `ListaPosts.jsx`, `PostsRelacionados.jsx` |

Cómo se leyó: lectura del código, de la semilla (`blog_seeder.ts:479-580`) y del test existente. No se ha
modificado nada. Las cifras de «lo que ve el lector» salen de aplicar las consultas a los datos de la semilla,
no de una captura de pantalla.

## Cómo se ordenó

1. **Precedencia primero.** Nivel 0: lo que hace posible tocar cualquier otra cosa. Nivel 1: lo que
   desbloquea o abarata a otro hallazgo. Nivel 2: no depende de nadie ni bloquea a nadie. Nivel 3: espera a
   que se haga otra fila (la fila lo nombra).
2. **Dentro de cada nivel, daño ÷ radio.** Daño de 1 a 5 (5 = datos personales expuestos o pérdida
   irrecuperable). Radio = sitios a tocar, contados (columna 4). Empate: va antes el de más daño.
3. **Un hallazgo con el daño más alto y la precedencia más baja a la vez se parte en dos filas.** Aquí es la
   fuga del correo: es lo más grave (daño 5) y su arreglo de fondo exige las capas nuevas (nivel 3). Por eso
   está en la tabla dos veces: la **contención** (fila 2) corta la fuga hoy; el **arreglo de fondo** (fila 13)
   espera su turno.

| Fila | Nivel | Daño | Sitios | Daño ÷ radio |
|---|---|---|---|---|
| 1 | 0 | 4 | 1 | 4,0 |
| 2 | 1 | 5 | 1 | 5,0 |
| 3 | 1 | 3 | 3 | 1,0 |
| 4 | 1 | 3 | 4 | 0,75 |
| 5 | 1 | 2 | 3 | 0,67 |
| 6 | 2 | 3 | 1 | 3,0 |
| 7 | 2 | 4 | 4 | 1,0 |
| 8 | 2 | 2 | 2 | 1,0 |
| 9 | 2 | 1 | 2 | 0,5 |
| 10 | 3 | 3 | 3 | 1,0 |
| 11 | 3 | 2 | 2 | 1,0 |
| 12 | 3 | 2 | 2 | 1,0 |
| 13 | 3 | 5 | 6 | 0,83 |
| 14 | 3 | 2 | 5 | 0,4 |

Las filas 3 y 4 están en el nivel 1 aunque su cociente sea bajo: desbloquean a las filas 11 y 10.
La fila 1 es el nivel 0 por mandato de `CLAUDE.md` (caracterizar antes de refactorizar), no por su cociente.

---

## Nivel 0 — lo que hace posible todo lo demás

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe y quién lo nota | 4. Sitios a tocar | 5. ¿Revert? | 6. ¿Bloquea o desbloquea? |
|---|---|---|---|---|---|---|
| 1 | La feature no tiene red de seguridad: un único test con una única aserción sobre el alta. | `backend/tests/functional/comentarios.spec.ts:42-70`; la única afirmación del contrato es `assert.equal(...autor_nombre...)` en la línea 69. Sin caso de email inválido, cuerpo corto, 404, DELETE, PATCH ni lectura. | Nada se rompe hoy; **no se enteraría nadie** si mañana se rompe. Lo nota quien refactorice: cualquiera de las filas 2-14 puede cambiar el comportamiento sin que ningún test se ponga rojo. | 1 archivo (`comentarios.spec.ts`), que cubre 4 rutas con la matriz de `CLAUDE.md` (feliz / inválida / 404 / frontera). | Sí: solo añade tests. | **Desbloquea a todas** (2-14): ninguna se arregla sin su test de caracterización en verde primero. Es la precedencia más baja del documento. |

## Nivel 1 — desbloquean o abarata a otro hallazgo

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe y quién lo nota | 4. Sitios a tocar | 5. ¿Revert? | 6. ¿Bloquea o desbloquea? |
|---|---|---|---|---|---|---|
| 2 | **Contención.** El detalle del post devuelve el correo de cada comentarista porque lee la fila entera. | `routes.ts:152-155` (`select * from comentarios`) y `routes.ts:171` (`comentarios: comentarios`). El front no usa `autor_email` en ningún punto (`Comentarios.jsx` solo lee `autor_nombre`, `creado_en`, `cuerpo`, `id`, `padre_id`), así que quitar la columna no cambia lo que se pinta. | Rompe la regla 6 de `CLAUDE.md`. **Lo nota cualquier lector**, sin autenticación: basta un `GET /posts/:slug` para recoger los correos de todos los comentaristas aprobados (en la semilla, Ana Cortés y Marta Ruiz en el post 1). Un raspador los junta sin esfuerzo. | 1 (`routes.ts:152-155`: nombrar las columnas en vez de `*`). | Sí el cambio. Lo que ya se haya servido no se recoge. | **Abarata la fila 3**: el correo deja de ser una llave pública. La fila 13 la sustituye por el arreglo de fondo. |
| 3 | El correo hace de contraseña para borrar: quien lo conozca borra el comentario de otro. | `routes.ts:349-357` (`esAutor` compara solo el correo, sin ninguna prueba de autoría); borrado físico en `routes.ts:359`. | Cualquiera que conozca o adivine el correo borra el comentario, y queda **sin rastro y sin papelera** (`delete from`). Hoy el correo es público (fila 2), así que es un `curl` de una línea. La web no llama nunca a DELETE (`Comentarios.jsx` solo hace GET y POST), por eso **nadie lo nota todavía por la interfaz**. | 3 (la ruta DELETE `routes.ts:339-361`; el POST `routes.ts:324-333`, que tendría que emitir la prueba de autoría; una migración con la columna que la guarda). | Sí el código. Los comentarios ya borrados no vuelven. | **Bloquea a la fila 11**: no se unifica la regla de autorización de DELETE y PATCH antes de decidir cómo se prueba la autoría. La fila 2 baja su daño. |
| 4 | `estado` es texto libre y la tabla ya tiene tres valores que significan lo mismo. | `1769000000005_create_comentarios_table.ts:18` (`table.string('estado')`, sin CHECK ni enum; el propio comentario de las líneas 3-7 lo admite). Datos: `blog_seeder.ts:510` (`'Aprobado'`, id 3), `:522` (`'ok'`, id 4), `:575` (`'Aprobado'`, id 9). | Tres comentarios ya moderados (ids 3, 4 y 9) **no salen en el detalle** (`routes.ts:152` compara con `= 'aprobado'`), pero sí cuentan en las tarjetas (fila 10). Lo nota el comentarista, cuyo comentario nunca aparece, y el moderador, que lo aprobó. | 4 (una migración nueva que normaliza y restringe; las 3 filas de la semilla). SQLite no añade un CHECK a una tabla existente: la migración tiene que reconstruirla. | **No.** El `down()` no puede saber cuáles eran `'ok'` y cuáles `'Aprobado'`: la distinción se pierde al normalizar. Hace falta copia previa de la tabla. | **Desbloquea la fila 10**: con tres valores sucios no hay un predicado correcto. Los tres comentarios ocultos se mostrarán al normalizar, y eso es un cambio de comportamiento que hay que decidir de forma explícita. |
| 5 | `padre_id` se acepta sin comprobar que exista, que sea del mismo post ni que sea un comentario raíz. | `routes.ts:308` (se lee) y `routes.ts:326` (se inserta tal cual); `1769000000005_...:20` (`padre_id` sin clave foránea). El pintado: `Comentarios.jsx:50` (solo raíces) y `:64` (respuestas solo de raíces de la misma lista). | Un comentario con un padre inexistente, de otro post o que sea él mismo una respuesta se guarda y, una vez aprobado, **no se pinta nunca**: desaparece sin error. **Nadie todavía por la web**: el formulario no envía `padre_id` (`Comentarios.jsx:33`). Solo lo provoca quien llame a la API a mano. | 3 (`routes.ts` POST; migración con la clave foránea; `Comentarios.jsx:64` si se quiere más de dos niveles). | Sí el código. La clave foránea exige reconstruir la tabla en SQLite (misma limitación que la fila 4). | **Desbloquea la fila 12**: la misma clave foránea resuelve el borrado en cascada. Conviene hacerla en la misma migración que la fila 4. |

## Nivel 2 — independientes

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe y quién lo nota | 4. Sitios a tocar | 5. ¿Revert? | 6. ¿Bloquea o desbloquea? |
|---|---|---|---|---|---|---|
| 6 | La respuesta del alta devuelve «el último comentario de la tabla», no el que se acaba de insertar. | `routes.ts:329-331` (`select * from comentarios order by id desc limit 1`), con un `await` entre la inserción (`:324`) y la lectura. | Si dos personas comentan casi a la vez, **la segunda recibe el comentario de la primera, con su correo y su texto pendiente** (el contenido que la moderación aún no ha visto y que no debía salir). Lo nota quien comenta justo después de otro. Rara hoy; el daño crece con el tráfico. | 1 (`routes.ts:329-331`: leer la fila por el id que devuelve la inserción). | Sí. | A ninguno. (Comparte la causa «el correo sale en una respuesta pública» con las filas 2 y 13, pero es otra consulta.) |
| 7 | La clave de moderación está escrita en el código y es la misma que se anuncia como «dev». | `routes.ts:25` (`const SECRETO_SESION = 'dev-secret-cambiar'`, con un `TODO(2024-11)` encima) y `routes.ts:351` (`clave === SECRETO_SESION`). Commit `e0e0943`. | Quien tenga el repositorio, o la cabecera `x-clave-moderacion`, **borra cualquier comentario** de cualquier instalación que no la haya cambiado. **Nadie todavía**, si ningún despliegue la usa tal cual (no consta cuál). | 4 (`routes.ts`; `backend/start/env.ts`; `.env.example`; `.env.test`). | **No.** La clave queda en el historial de git; revertir no la vuelve secreta. Hay que rotarla. | A ninguno. |
| 8 | La web guarda dos recuentos de comentarios que dejan de coincidir al comentar. | `DetallePost.jsx:30` y `:38` (recuento de la cabecera, pintado en `:71`); `Comentarios.jsx:11` y `:39` (`setTotal(total + 1)`), pintado en `:54`. | La cabecera del artículo no se entera nunca de un comentario nuevo, y el `<h2>` sube en uno al enviar **aunque el comentario quede pendiente y no se vea**: «3 comentarios» con 2 en pantalla. Lo nota **cualquier lector que comenta**, inmediatamente. | 2 (`Comentarios.jsx`, `DetallePost.jsx`). | Sí. | **Va con la fila 9**: se arreglan juntas en los mismos dos archivos (el padre pasa los comentarios por props). No bloquea a ninguna. |
| 9 | Cada visita al detalle pide dos veces el mismo post al servidor. | `Comentarios.jsx:19` (`fetch(API + '/posts/' + slug)`) duplica el de `DetallePost.jsx:34`; cada una ejecuta las 7 consultas de `routes.ts:110-164`. | Doble carga en el servidor y en la red por cada artículo abierto. **Nadie todavía**: con la semilla no se aprecia. | 2 (los mismos que la fila 8). | Sí. | **Va con la fila 8**. |

## Nivel 3 — esperan a otra fila

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe y quién lo nota | 4. Sitios a tocar | 5. ¿Revert? | 6. ¿Bloquea o desbloquea? |
|---|---|---|---|---|---|---|
| 10 | El recuento de comentarios cuenta «todo lo que no sea pendiente» y el detalle muestra «solo lo que es `aprobado`». | `routes.ts:75-76` y `routes.ts:223` (`estado <> 'pendiente'`) frente a `routes.ts:152` (`estado = 'aprobado'`). Pintado en `ListaPosts.jsx:57` y `PostsRelacionados.jsx:32`. | Rompe la regla 5. Con la semilla, la tarjeta del post 1 anuncia **4 comentarios y el artículo enseña 2**; la del post 6 anuncia 1 y enseña 0. Lo nota **cualquier lector** que entra desde el listado. | 3 consultas, en `routes.ts` (75-76, 152, 223). | Sí. | **Espera a la fila 4**: sin los valores normalizados, no hay un predicado único correcto. |
| 11 | La regla «quién puede tocar un comentario» está escrita dos veces y las dos versiones difieren. | `routes.ts:351-354` (DELETE: acepta clave de moderación, compara el correo sin mayúsculas ni espacios, vale en cualquier estado) frente a `routes.ts:378-381` (PATCH: solo el autor, correo exacto, solo pendiente). El comentario de `routes.ts:375-377` lo reconoce. | Quien escribió su correo con otra capitalización puede borrar su comentario pero no editarlo; un moderador puede borrar pero no editar. **Nadie todavía por la web** (no llama a PATCH ni a DELETE). | 2 (las dos rutas). | Sí. | **Espera a la fila 3**: la regla única depende de cómo se pruebe la autoría. |
| 12 | Borrar un comentario con respuestas las deja huérfanas e invisibles. | `routes.ts:359` (`delete from comentarios where id = ?`, sin tocar las que apuntan a él por `padre_id`); invisibles por `Comentarios.jsx:50` (toda respuesta tiene `padre_id`, así que no es raíz) y `:64`. | Las respuestas ya aprobadas desaparecen sin aviso del hilo y quedan en la base. **Nadie todavía**: el front no borra. En la semilla, el comentario 1 tiene una respuesta (el 2). | 2 (`routes.ts:359`; la migración de la fila 5). | **No.** Los comentarios borrados no se recuperan, y un borrado en cascada borra también las respuestas. | **Espera a la fila 5**: la clave foránea con `ON DELETE` resuelve este hallazgo; sin ella habría que borrar a mano. |
| 13 | **Arreglo de fondo** de la fuga del correo: no existe una capa que decida qué campos de un comentario salen. | `routes.ts:152-155` y `:171`; el modelo `ComentarioSchema` existe en `backend/database/schema.ts:73` pero no hay `app/models/comentario.ts`, ni repositorio, ni transformer (el único ejemplo correcto es `app/transformers/user_transformer.ts`). | Misma fuga que la fila 2. Tras la contención solo queda el riesgo de que la próxima consulta vuelva a hacer `select *` y filtre: **lo notaría quien raspe la API, no el equipo**. | 6 (`routes.ts`; un controlador; un repositorio; el modelo; un transformer; el alias `#repositories/*` en `backend/package.json`). El endpoint de detalle se toca entero. | Sí. | **Absorbe a la fila 2** y espera a la fila 1. Comparte controlador y validador con la fila 14. |
| 14 | El alta responde los errores con cuatro formas distintas y el front solo entiende tres. | `routes.ts:311` (400 `{error}`), `:314` (422 `{message}`), `:317` (400 `{errores: [...]}`), `:320` (200 `{ok:false, motivo}`); el front lee `d.error \|\| d.message \|\| d.motivo` en `Comentarios.jsx:44` y **no lee `errores`**. | Con un correo sin `@` el visitante ve «No se ha podido enviar.», sin saber por qué; un cuerpo corto responde **200**, que cualquier cliente que mire el código toma por éxito. Lo nota **cualquier lector que comenta**. | 5 (las 4 ramas de `routes.ts`; la línea `Comentarios.jsx:44`). | Sí. | **Espera a la fila 13** (validador VineJS y manejador de errores). Arreglar la línea del front antes fijaría en el cliente un contrato que el refactor va a retirar. |

---

## Propuestas sin medir (fuera de la tabla)

Ninguna de estas tiene todavía una casilla 3 honesta: falta el dato que dice si el problema existe.

| Propuesta | Qué habría que medir antes de escribir la fila |
|---|---|
| Ruta de moderación (aprobar / rechazar). **No hay ninguna ruta que escriba `'aprobado'`** (se puede comprobar buscando `estado` en `routes.ts`: solo se lee y se inserta `'pendiente'`); la semilla menciona un «panel viejo» (`blog_seeder.ts:509`). | Cómo se aprueba un comentario hoy; cuántos pendientes hay; la antigüedad del más viejo. Una ausencia no tiene línea que la demuestre, y por eso no es fila. |
| Límite de frecuencia o antispam en el alta (la semilla trae un comentario promocional, id 7). | Comentarios por hora y por origen. Hoy no se guarda IP ni cabecera alguna, así que el dato no existe: habría que empezar a registrarlo. |
| Índice sobre `comentarios(post_id)`; la subconsulta correlacionada de `routes.ts:75-76` se ejecuta por cada post del listado. | `EXPLAIN QUERY PLAN` y tiempo de `GET /posts` con 10 000 y con 100 000 comentarios. La semilla tiene 9 filas: no se puede concluir nada. |
| Paginar o limitar los comentarios del detalle (`routes.ts:152-155` no tiene `limit`). | La distribución de comentarios por post. El máximo en la semilla es 4. |
| Validar tipo y longitud de los campos del alta y de la edición (`routes.ts:305-321`, `383`: solo se mira que no estén vacíos, y que el correo tenga una `@`). | Qué responde la API con un `autor_nombre` que no sea texto, con un cuerpo de 1 MB y con uno de solo espacios. **No se ha ejecutado**; el driver de SQLite rechaza algunos tipos al enlazarlos, pero el código HTTP resultante no está comprobado. |
| Fechas: `creado_en` se escribe en UTC (`routes.ts:323`) y la web corta los diez primeros caracteres (`Comentarios.jsx:59` y `:68`). | Si los datos reales mezclan UTC y hora local, como hace la semilla; y cuántos comentarios caen junto a la medianoche. |

## Revisado y sin hallazgo

- **Inyección SQL en comentarios:** las cuatro rutas y los recuentos usan parámetros (`?`), no concatenan la entrada.
- **XSS en el texto del comentario:** `Comentarios.jsx:61` y `:70` lo pintan como texto de React. El único `dangerouslySetInnerHTML` (`DetallePost.jsx:92`) es el del cuerpo del post, no el de los comentarios.
- **Comentar en un borrador:** el alta devuelve 404 si el post no está publicado (`routes.ts:296-302`). La regla 1 se cumple.
- **Comentarios pendientes en el detalle:** no salen (`routes.ts:152`). La regla 2 se cumple.


Cuántos hallazgos devolvió el agente, y cuántos comprobaste tú abriendo el archivo.  14,4 

El hallazgo del que no supiste decidir la prioridad, y en una frase, por qué. El correo hace de contraseña para borrar

Algo que el agente afirmó y no pudiste verificar. Ojo al matiz: lo de que trae l ultimo comentario y no el que se acaba de crear 