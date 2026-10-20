# Auditoría priorizada — feature de comentarios

- **Alcance:** solo la feature de comentarios (crear, editar, borrar y listar) en sus tres capas: API
  (`backend/start/routes.ts`), base de datos (migración, esquema y semilla de `comentarios`) y web
  (`frontend/src/componentes/Comentarios.jsx` y lo que `DetallePost.jsx`, `ListaPosts.jsx` y
  `PostsRelacionados.jsx` enseñan de ella). Lo demás queda fuera, aunque se haya visto de pasada (por
  ejemplo, el correo del **autor del post** que también sale en `GET /posts/:slug`).
- **Contraste:** las reglas de negocio de `README.md` y `CLAUDE.md` (R1–R6) y la arquitectura objetivo de `CLAUDE.md`.
- **Fecha:** 2026-10-20 · **Agente:** Claude Opus 5.5 en Claude Code · rama `auditoria-jfc` (commit `ad34f0e`).
- **No se ha modificado código funcional.** La base de desarrollo (`backend/tmp/db.sqlite3`, ignorada por git)
  se creó con `make db` para las pruebas y **se volvió a sembrar al terminar** (9 comentarios, como en la semilla).

## Leyenda

- **[Comprobado]**: el agente abrió las líneas citadas **y** ejecutó la petición o consulta indicada
  en el registro de evidencias (E1–E14), o el efecto se sigue literalmente de las líneas leídas.
  Si solo se comprobó leyendo código, se indica «(código)».
- **[Propuesto]**: hipótesis razonable que el agente **no ha podido demostrar**. Sus casillas no
  verificadas se dejan en blanco.
- **Lugares** (columna 4): sitios concretos que habría que editar, contados uno por uno sobre las
  líneas citadas. En blanco si no se puede contar sin diseñar antes la solución.

## Criterio de prioridad

1. **Precedencia** (primero): qué tiene que estar hecho antes para poder corregir otra cosa sin romper
   las reglas del repo. `CLAUDE.md` exige caracterizar antes de tocar nada, así que la red de tests
   va primero. Después van los hallazgos que solo dependen de ella. Al final, los que dependen de otro hallazgo.
   - **Nivel 1:** red de seguridad (C-01).
   - **Nivel 2:** solo bloqueados por C-01.
   - **Nivel 3:** bloqueados además por otro hallazgo de nivel 2.
2. **Dentro de cada nivel: daño ÷ radio de impacto.** El daño va en una escala 1–5 que puse yo con este criterio:
   5 = pérdida irreversible de datos ajenos, 4 = exposición de datos personales o de un secreto,
   3 = contenido legítimo que desaparece o cifras públicas falsas, 2 = una persona usuaria recibe un
   comportamiento incorrecto, 1 = nadie lo nota todavía. El **radio de impacto** es el número de
   lugares de la columna 4. Si hay empate, gana el de más daño. Esta lectura de «radio de impacto» es
   una interpretación mía (ver la nota al final).

## Contención inmediata vs. corrección definitiva

La gravedad de C-02, C-03 y C-05 choca con la precedencia: su corrección definitiva tiene que esperar
a C-01 (tests de caracterización), pero el daño ya se puede provocar hoy. Por eso se separan:

| Hallazgo | Contención inmediata (no espera a C-01) | Corrección definitiva (después de C-01) |
|---|---|---|
| C-02 Cualquiera borra cualquier comentario | Desactivar `DELETE /comentarios/:id` (`routes.ts:339`) o exigir solo la clave de moderación, y hacer una copia de la tabla `comentarios` | Autorización real (token o propiedad verificada) en un servicio, con su test |
| C-03 El correo de quien comenta sale en público | Dejar de seleccionar `autor_email` en `routes.ts:153` | Transformer de salida para el detalle, con aserción negativa `notProperty(..., 'autor_email')` |
| C-05 Clave de moderación en el código | **Rotarla** (ya está en el historial de git) y leerla de una variable de entorno | Sacarla a `env` validado y separar el permiso de moderación de la sesión |

Estas contenciones son propuestas, **no se han aplicado** (el encargo prohíbe tocar código funcional).

## Tabla de hallazgos

| # | (1) Qué es | (2) Dónde (archivo:líneas que lo demuestran) | (3) Qué se rompe y quién lo nota | (4) Lugares a tocar | (5) ¿Se deshace con un revert? | (6) Bloquea / desbloquea |
|---|---|---|---|---|---|---|
| **Nivel 1** | | | | | | |
| C-01 | **[Comprobado]** La feature tiene un único test, que solo prueba el camino feliz de crear un comentario. Editar, borrar, el listado y los errores no tienen ningún test. | `backend/tests/functional/comentarios.spec.ts:15-43`; `npm test` → `Tests 1 passed (1)` (E1) | Nada falla hoy. Lo notará quien desarrolle: cualquier corrección de C-02…C-21 puede romper otro comportamiento sin que la suite se entere | 1 (`comentarios.spec.ts`, ampliado con la matriz feliz / inválida / 404 / frontera para POST, PATCH, DELETE y los comentarios del detalle) | Sí: solo añade tests | **Desbloquea** la corrección definitiva de todos los demás (CLAUDE.md: caracterizar antes de tocar) |
| **Nivel 2** | | | | | | |
| C-02 | **[Comprobado]** Cualquiera puede borrar cualquier comentario: `DELETE` autoriza con un `autor_email` que llega en la propia petición, y ese correo se publica en `GET /posts/:slug`. | `backend/start/routes.ts:350-357` (la comparación de correos es la "autorización"); `:359` (`delete from` físico); el correo se obtiene de `:152-153,171` | Se borró el comentario aprobado id 1 (de otra persona) mandando `"ANA.CORTES@ejemplo.es "`, el correo publicado en el detalle → `200 {"borrado":true}` (E9). Lo notan quien comentó y los lectores, y nadie recibe aviso | 1 (`routes.ts:349-357`) | **No**: el borrado es físico (`:359`). Revertir el código no devuelve las filas | Bloqueado por C-01. C-03 reduce el riesgo pero no lo elimina (el correo se puede adivinar). **Bloquea** a C-20 |
| C-03 | **[Comprobado]** El correo de quien comenta sale en la respuesta pública del detalle del post (incumple R6). | `backend/start/routes.ts:152-153` (`select *`), devuelto tal cual en `:171` | `GET /posts/diez-tendencias…` devuelve `ana.cortes@ejemplo.es` y `marta.ruiz@corriente.es` dentro de `comentarios[]` (E2). La web no lo pinta, pero sale en cualquier `curl` o en las devtools. Afecta a quien comenta (privacidad) | 1 (`routes.ts:153`) | El código sí. **La exposición ya ocurrida no**: los correos servidos no se recuperan | Bloqueado por C-01 (contención: no). **Desbloquea** (mitiga) C-02 |
| C-04 | **[Propuesto]** `POST` devuelve «el último comentario de toda la tabla» en vez del recién insertado. Con peticiones concurrentes podría devolver el comentario ajeno, con su correo. | `backend/start/routes.ts:329-331` (`select * from comentarios order by id desc limit 1`) | | 1 (`routes.ts:329-331`) | | Bloqueado por C-01 |
| C-05 | **[Comprobado]** La clave de moderación está escrita en el código y versionada. Con ella se borra cualquier comentario. | `backend/start/routes.ts:25`, usada en `:349-351` | `DELETE /comentarios/10` con `x-clave-moderacion: dev-secret-cambiar` → `200 {"borrado":true}` (E10). La puede usar quien lea el repo. Hoy nadie lo nota | 2 (`routes.ts:25`, `:351`) | **No**: el secreto ya está en el historial (`e0e0943`); hay que rotarlo | Bloqueado por C-01 (contención: no). Relacionado con C-10 |
| C-06 | **[Comprobado]** `PATCH` no aplica la longitud mínima que sí exige `POST`: se puede dejar un comentario con 1 carácter. | `backend/start/routes.ts:383-386` frente a `:319` | `PATCH /comentarios/10` con `cuerpo:"a"` → `200` y el cuerpo queda en `"a"` (E6). Lo nota moderación al revisar pendientes | 1 (`routes.ts:384`) | El código sí; los cuerpos ya editados no | Bloqueado por C-01 |
| C-07 | **[Comprobado]** `estado` es texto libre, y los comentarios aprobados guardados como `'Aprobado'` u `'ok'` no aparecen en el detalle (R2/R5). | Migración `1769000000005_create_comentarios_table.ts:18` (sin CHECK); `routes.ts:153` (`= 'aprobado'` exacto); semilla `blog_seeder.ts:510,522,575` | El post 1 enseña 2 de 4 comentarios no pendientes (faltan los ids 3 y 4); el post 6 enseña 0 de 1 (E2, E3). Lo notan quienes comentaron (su comentario aprobado no sale) y los lectores. La BD tiene `Aprobado:2, aprobado:3, ok:1, pendiente:2` (E12) | 4 (filas de la semilla `:510`, `:522`, `:575` y 1 migración nueva de normalización + CHECK) | **No** del todo: normalizar `'ok'`/`'Aprobado'` a `'aprobado'` pierde el valor original si no se guarda antes | Bloqueado por C-01 y por una decisión de negocio (¿`'ok'` es aprobado?). **Desbloquea** C-18 |
| C-08 | **[Comprobado]** Al borrar un comentario padre, sus respuestas se quedan huérfanas y la web deja de pintarlas. | `backend/start/routes.ts:359` (no toca las respuestas); migración `:20` (`padre_id` sin FK); `frontend/src/componentes/Comentarios.jsx:50,63-64` (solo pinta respuestas de una raíz presente) | Tras borrar el id 1, el id 2 (aprobado, `padre_id:1`) sigue en la API con un padre que no existe (E9, E11). Por el código del front ya no se pinta (no visto en navegador). Lo nota quien respondió | 2 (`routes.ts:359`; migración nueva con FK `padre_id` + política de borrado) | **No**: el borrado es físico | Bloqueado por C-01. Comparte migración con C-09 y C-12 |
| C-09 | **[Comprobado]** `POST` acepta un `padre_id` que no existe. | `backend/start/routes.ts:308,326`; migración `:20` | `padre_id: 99999` → `201` y se guarda (E5). Aunque se apruebe, la respuesta no saldrá nunca en la web (`Comentarios.jsx:50,63-64`). Lo nota quien la escribió. No se probó un `padre_id` de **otro** post | 2 (`routes.ts:308`; FK en migración nueva) | Sí el código; las filas ya guardadas quedan | Bloqueado por C-01. Comparte migración con C-08 |
| C-10 | **[Comprobado]** `PATCH` y `DELETE` aplican reglas de autorización distintas: `PATCH` distingue mayúsculas en el correo, no recorta espacios, ignora la clave de moderación y solo permite editar pendientes. | `backend/start/routes.ts:349-353` frente a `:378-381` (el comentario de `:375-377` lo admite) | El correo `prueba@ejemplo.es` → 403 en PATCH (guardado `Prueba@Ejemplo.es`) (E7); la clave de moderación → 403 en PATCH (E8); el mismo correo en minúsculas sí borra (E9). Lo notan quien comenta y moderación | 2 (`routes.ts:349-353`, `:378-381`) | Sí | Bloqueado por C-01 y por C-02 (definir primero la autorización). **Bloquea** a C-20 |
| C-11 | **[Comprobado] (código)** La URL de la API está fija en el componente de comentarios («cuarta copia»). | `frontend/src/componentes/Comentarios.jsx:3-4` | Nadie todavía. Se rompe al desplegar en otro host o puerto | 1 (`Comentarios.jsx:4`, solo en esta feature) | Sí | Bloqueado por C-01 (front sin tests: ninguno) |
| C-12 | **[Comprobado]** `comentarios.post_id` no tiene clave foránea ni índice, y `estado` no tiene CHECK. | Migración `1769000000005_create_comentarios_table.ts:4-6,14,18`; `pragma foreign_key_list` e `index_list` vacíos (E12) | Nadie todavía (no hay endpoint para borrar posts). Cada listado hace un `count` por post sin índice (`routes.ts:75-76`) | 1 (migración nueva) | Sí, con un `down()` reversible | Bloqueado por C-01. Comparte migración con C-07, C-08 y C-09 |
| C-13 | **[Comprobado] (código)** Las listas de comentarios y respuestas usan el índice como `key`. | `frontend/src/componentes/Comentarios.jsx:57,66` | Nadie todavía: la lista se carga una vez y no se reordena | 2 (`:57`, `:66`) | Sí | A ninguno |
| C-14 | **[Comprobado] (código)** El detalle se pide dos veces: `Comentarios` vuelve a pedir el post entero solo para sacar los comentarios. | `frontend/src/componentes/Comentarios.jsx:17-26`; `DetallePost.jsx:32-43,123` | Una petición extra por visita. Ningún usuario lo nota | 2 (`Comentarios.jsx:17-26`; `DetallePost.jsx:123`, pasar los comentarios como prop) | Sí | **Desbloquea** C-19 (una sola fuente) |
| C-15 | **[Comprobado]** La validación de `POST` responde con códigos y formas distintos para cada error, y un cuerpo corto devuelve `200`. | `backend/start/routes.ts:310-321`; `frontend/src/componentes/Comentarios.jsx:44` | Sin nombre → `400 {error}`; sin email → `422 {message}`; email sin `@` → `400 {errores:[…]}`; cuerpo corto → `200 {ok:false}` (E4). La web no lee `errores`, así que un email inválido muestra «No se ha podido enviar.» (código, no visto en navegador). Lo nota quien comenta y cualquier cliente de la API | 5 (`routes.ts:310-312`, `:313-315`, `:316-318`, `:319-321`; `Comentarios.jsx:44`) | Sí, pero cambiar los códigos rompe el contrato actual | Bloqueado por C-01: cambia un contrato, así que primero se caracteriza |
| C-16 | **[Comprobado] (código)** Toda la feature vive en las rutas: SQL, validación, autorización y forma de la respuesta, sin controlador, validador, servicio, repositorio ni transformer. | `backend/start/routes.ts:150-155,295-334,339-361,366-393` | Nadie en producción; lo nota quien la mantiene. Es la causa de que C-03, C-10 y C-15 existan | 4 (los cuatro bloques citados) | Sí | Bloqueado por C-01. **Desbloquea** la corrección limpia de C-03 (transformer), C-15 (validador) y C-10 (servicio) |
| C-17 | **[Propuesto]** `POST` no tiene límite de frecuencia ni antispam. | `backend/start/routes.ts:295`; no aparece `limiter`/`throttle` en `backend/start` ni `backend/config` (E13). La semilla trae spam pendiente (`blog_seeder.ts:551`) | | | | |
| **Nivel 3** | | | | | | |
| C-18 | **[Comprobado]** El recuento `num_comentarios` cuenta todo lo que no sea `'pendiente'` en vez de solo los aprobados (R5), y no coincide con lo que enseña el detalle. | `backend/start/routes.ts:75-76`, `:223` | Con la semilla, el post 6 dice `num_comentarios: 1` en el listado y su detalle tiene 0 comentarios (E3). Tras las pruebas, el post 1 decía 3 y su detalle tenía 1 (E14). Cualquier estado nuevo (`'rechazado'`, `'spam'`) también contaría. Lo ven los lectores en portada y en relacionados (`ListaPosts.jsx:57`, `PostsRelacionados.jsx:32`) | 2 (`routes.ts:75-76`, `:223`) | Sí | Bloqueado por C-01 y **por C-07** (definir el valor canónico de «aprobado»). **Desbloquea** C-19 |
| C-19 | **[Comprobado] (código)** La web lleva dos contadores de comentarios distintos, y el de la sección suma 1 al enviar aunque el comentario quede pendiente. | `frontend/src/componentes/DetallePost.jsx:30,38,71`; `frontend/src/componentes/Comentarios.jsx:11,23,39,54` | Después de comentar, la cabecera dice N y la sección dice N+1 sin que se vea ningún comentario nuevo. Lo nota quien acaba de comentar (no visto en navegador) | 7 (`DetallePost.jsx:30`, `:38`, `:71`; `Comentarios.jsx:11`, `:23`, `:39`, `:54`) | Sí | Bloqueado por C-18 (cifra fuente) y C-14 |
| C-20 | **[Comprobado]** La web no permite editar ni borrar comentarios, aunque la API sí. | Sin `PATCH`, `DELETE` ni `x-clave` en `frontend/src` (E13); `Comentarios.jsx:76-88` solo tiene el formulario de alta | Quien comenta no puede corregir ni retirar su comentario desde la web | | Sí | Bloqueado por C-02 y C-10 (no tiene sentido exponer en la web una autorización rota) |
| C-21 | **[Propuesto]** La web solo pinta un nivel de respuestas; una respuesta a otra respuesta no saldría. | `frontend/src/componentes/Comentarios.jsx:50,63-64` | | 1 (`Comentarios.jsx:63-64`) | Sí | |

## Registro de evidencias (comandos ejecutados por el agente)

Sobre la base recién sembrada (`make db`) y la API levantada en 3401. `S` = `diez-tendencias-de-diseno-que-pueden-cambiar-la-web-moderna` (post 1).

| Id | Comprobación | Resultado observado |
|---|---|---|
| E1 | `make test` | `Tests 1 passed (1)` |
| E2 | `GET /posts/S` → `comentarios[]` | `[1,'aprobado','ana.cortes@ejemplo.es'], [2,'aprobado','marta.ruiz@corriente.es']`; los ids 3 (`Aprobado`) y 4 (`ok`) no salen |
| E3 | `GET /posts?pagina=1` → `num_comentarios`; detalle del post 6 | post 6 → `1` en el listado; el detalle del post 6 → 0 comentarios |
| E4 | `POST /posts/S/comentarios` con datos inválidos | cuerpo `"hey"` → `200 {"ok":false,…}`; sin nombre → `400 {"error"}`; sin email → `422 {"message"}`; `xy.es` → `400 {"errores":[…]}`; slug inexistente → `404` |
| E5 | `POST` con `padre_id: 99999` | `201`, guardado con `padre_id: 99999` (id 10) |
| E6 | `PATCH /comentarios/10` con el correo exacto y `cuerpo:"a"` | `200`, `cuerpo:"a"` |
| E7 | `PATCH /comentarios/10` con el correo en minúsculas | `403` |
| E8 | `PATCH /comentarios/10` con `x-clave-moderacion` | `403`. `PATCH /comentarios/1` (aprobado) con el correo de su autora → `403`. 404 y cuerpo ausente → `404` / `400` |
| E9 | `DELETE /comentarios/1` con body `{"autor_email":"ANA.CORTES@ejemplo.es "}` | `200 {"borrado":true}`. Sin credencial → `403`; id inexistente → `404` |
| E10 | `DELETE /comentarios/10` con `x-clave-moderacion: dev-secret-cambiar` | `200 {"borrado":true}` |
| E11 | Detalle de `S` tras E9; consulta de huérfanos en SQLite | `comentarios: [[2, padre_id 1]]`; huérfano `{id:2, padre_id:1}` |
| E12 | `pragma foreign_key_list/index_list(comentarios)`; `group by estado` | `[]`, `[]`; `Aprobado:2, aprobado:3, ok:1, pendiente:2` (después de E5–E10) |
| E13 | `grep` de `DELETE`/`PATCH`/`x-clave` en `frontend/src`; de `limiter`/`throttle` en `backend/start` y `backend/config`; de `SECRETO_SESION` | Sin resultados en el front ni en `limiter`/`throttle`. `SECRETO_SESION` solo aparece en `routes.ts:25,351` |
| E14 | 40 `POST` concurrentes comparando el nombre enviado con el devuelto; recuento del post 1 | **0 discrepancias** (C-04 no se reproduce). Post 1 → `num_comentarios: 3`, detalle con 1 comentario |

Al terminar se paró la API y se volvió a ejecutar `make db` (vuelven a ser 9 comentarios).

## Nota sobre el criterio

- La escala de daño 1–5 y la lectura de «radio de impacto» como número de lugares a tocar (columna 4)
  son decisiones mías. Si «radio de impacto» se entiende como **número de personas afectadas**, la
  fórmula daño ÷ radio pondría primero lo que afecta a pocos, y el orden del nivel 2 cambiaría
  (C-15 subiría y C-03 bajaría).
- El comentario de `routes.ts:21-24` dice que la clave «firma las sesiones». En el repo solo se usa
  en `:351` (E13), así que esa afirmación del comentario **no se cumple** en el código actual.

---

## Parte B

1. **Cuántos hallazgos devolvió el agente y cuántos comprobé yo abriendo el archivo:**
   El agente identificó 21 hallazgos. Yo comprobé personalmente 3, abriendo los archivos y revisando la evidencia: C-02, C-03 y C-05. En C-05 también comprobé que SECRETO_SESION aparece en el historial de Git mediante el commit e0e0943. No ejecuté personalmente pruebas HTTP.
2. **El hallazgo del que no supe decidir la prioridad, y por qué:**
   El hallazgo cuya prioridad me resultó difícil decidir fue C-04, porque describe una posible condición de carrera que no se reprodujo en las 40 peticiones concurrentes. No sé si priorizar el riesgo potencial por encima de defectos reproducibles.
3. **Algo que el agente afirmó y no pude verificar:**
   No pude verificar el comportamiento visual de los comentarios en el navegador, especialmente los comentarios que desaparecen y las diferencias en los contadores. La evidencia disponible procede del análisis del código, no de una comprobación personal de la interfaz.
