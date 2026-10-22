# Auditoría: feature de comentarios

Alcance: API (`backend/start/routes.ts`), tabla `comentarios` y lo que enseña la web
(`frontend/src/componentes/Comentarios.jsx`). Nada más.
Método: lectura de código, sin ejecutar nada. Cada «dónde» es una línea abierta y leída.
Reloj: empezado 11:48, parado en cuanto la tabla estuvo escrita. Las casillas en blanco son lo que no llegué a mirar.

Orden: primero precedencia (lo que desbloquea o abarata a otro), dentro de cada nivel daño ÷ radio.
Fuga de correo partida en dos filas (2 y 8): contención hoy, arreglo de fondo después.

| # | Qué es | Dónde (la línea que lo demuestra) | Qué se rompe y quién lo nota | Sitios a tocar | ¿Revert? | ¿Bloquea / desbloquea? |
|---|---|---|---|---|---|---|
| 1 | Los comentarios no tienen tests de caracterización: solo hay uno, el camino feliz del POST. | `tests/functional/comentarios.spec.ts:15` (único `test(`) | Nada se rompe hoy; ningún cambio posterior (filas 2–11) se puede hacer como refactor sin red. | 1 archivo nuevo de tests | Sí | **Desbloquea** a 2–11 (CLAUDE.md exige caracterizar antes de tocar). |
| 2 | **Contención.** `GET /posts/:slug` devuelve el correo de quien comenta. | `routes.ts:152-153` (`select * from comentarios`) | Cualquier visitante lee el `autor_email` de cada comentario aprobado: viola la regla 6. | 1 consulta (listar columnas) | Sí | **Abarata** a 6: sin el correo público, el borrado por correo deja de ser trivial. |
| 3 | `estado` es texto libre y hay filas `'Aprobado'` y `'ok'`. | `blog_seeder.ts:510,575` (`'Aprobado'`), `:522` (`'ok'`); `migrations/..._comentarios_table.ts:18` (sin CHECK) | El detalle filtra por `= 'aprobado'` exacto (`routes.ts:153`): 3 comentarios de la semilla (2 `Aprobado` + 1 `ok`) no se muestran aunque estén aprobados. Lo nota el lector. | 1 migración de datos + 1 CHECK/enum (más la semilla) | **No**: normalizar pierde la información de qué valor tenía cada fila; hace falta un `down` que no puede reconstruirlo | **Desbloquea** a 4: el recuento no se puede corregir bien mientras el dato esté sucio. |
| 4 | El recuento de comentarios cuenta `<> 'pendiente'` en vez de solo los aprobados. | `routes.ts:75-76` y `routes.ts:223` | Regla 5: con `'ok'`/`'Aprobado'` el número de las tarjetas no coincide con los comentarios que se listan. Lo nota el lector en `ListaPosts.jsx:57` y `PostsRelacionados.jsx:32`. | 2 en backend (los 2 `count(*)`); front: 0 | Sí | **Bloqueado por** 3. |
| 5 | El POST devuelve «el último comentario de la tabla», no el que acaba de insertar. | `routes.ts:329-330` (`order by id desc limit 1`, sin `where`) | Con dos envíos casi simultáneos, A recibe el comentario de B, con su correo (otra fuga de la regla 6). Hoy: casi nadie; con tráfico, cualquiera. | 1 | Sí | A ninguno |
| 6 | Borrar un comentario ajeno basta con conocer su correo: el «autor» se identifica con un `autor_email` del cuerpo, sin sesión. | `routes.ts:350-353` | Quien lea un correo (fila 2) borra ese comentario con un `DELETE`. Lo notaría el autor al ver desaparecer su comentario. | 1 (+ rediseñar la identidad: sin contar) | Sí el código; **no** el comentario borrado (`delete` físico, `routes.ts:359`) | **Bloqueado parcialmente por** 2. |
| 7 | La clave de moderación está en el código, con un valor de ejemplo. | `routes.ts:25` (`'dev-secret-cambiar'`) y `:351` | Quien tenga el repo puede borrar cualquier comentario con la cabecera `x-clave-moderacion`. | 2 (la constante y su uso) | Sí, pero la clave ya ha estado en el historial de git: rotarla no se deshace con un revert | A ninguno |
| 8 | **Fondo.** El comentario sale como fila cruda, sin transformer. | `routes.ts:171` (detalle) y `:333`, `:392` (POST y PATCH devuelven la fila entera) | Hoy filtra correo (2, 5); mañana filtrará cualquier columna nueva. | 3 respuestas + crear transformer, validador, servicio y repositorio de comentarios | Sí | **Bloqueado por** 1; absorbe a 2. |
| 9 | `padre_id` se acepta sin comprobar que exista ni que sea del mismo post. | `routes.ts:308` y `:326` | Una respuesta con `padre_id` inexistente o de otra rama no se pinta nunca (`Comentarios.jsx:50,64`). Lo nota quien comenta. | 1 en backend; front: | Sí | |
| 10 | Un mismo POST falla de cuatro formas distintas: 400, 422, 400 con otro formato y `200 {ok:false}`. | `routes.ts:311,314,317,320` | Unificarlo rompe el front, que compensa con `d.error \|\| d.message \|\| d.motivo` (`Comentarios.jsx:44`). | backend: 4; front: 1; otros consumidores: | Sí | **Bloqueado por** 1 (hay que caracterizar `200 {ok:false}` primero). |
| 11 | El PATCH tiene otra regla que el DELETE y no exige el mínimo de 5 caracteres del POST. | `routes.ts:379` (email sensible a mayúsculas, sin moderador, solo pendiente) y `:384` | | | | |
| 12 | El total de la web sube al enviar aunque el comentario quede pendiente y no se vea. | `Comentarios.jsx:39` (`setTotal(total + 1)`) | | | | |

## Fuera de la tabla: nadie lo ha medido

| Propuesta | Qué habría que medir |
|---|---|
| Índice / clave foránea en `comentarios.post_id` (la migración no los tiene) | Filas reales de `comentarios` en producción y el `EXPLAIN` del detalle; la semilla tiene 9 filas, no dice nada. |
| N+1 en relacionados (un `count` por relacionado, `routes.ts:222`) | Latencia real de `/posts/:slug/relacionados` con el volumen de producción. |
| Antispam / límite de frecuencia en el POST | Cuántos comentarios basura llegan hoy; no hay logs en el repo. |

---

## Parte B: las tres líneas

1. **Números:** el agente devolvió 15 hallazgos (12 en la tabla y 3 fuera); yo comprobé 12 abriendo el archivo y la línea citada, y ninguno ejecutándolo.
2. **Sin decidir:** la fila 2 (contención de la fuga del correo) contra la fila 1 (tests primero): es lo más grave y lo más barato, pero el propio `CLAUDE.md` dice que un bug se caracteriza antes de corregirse; lo grave empujaba a poner la 2 la primera y lo que bloquea, la 1.
3. **No verificado:** que `dev-secret-cambiar` sea, o haya sido, el valor real en algún entorno desplegado; solo vi el literal en el código, y lo que haya en producción no está en el repo.
