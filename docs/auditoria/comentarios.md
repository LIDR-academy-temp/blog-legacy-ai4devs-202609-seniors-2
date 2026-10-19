# Auditoría: feature de comentarios

Alcance: rutas de comentarios y detalle del post en `backend/start/routes.ts`, migración `1769000000005_create_comentarios_table.ts`, `Comentarios.jsx` y `DetallePost.jsx`. Orden, citas y razonamiento están debajo de la tabla. «Sitios» = líneas o bloques contiguos que el arreglo edita. Las casillas en blanco no las he podido establecer leyendo el código (la de «revert» queda en blanco en todas: depende del historial y de los datos, que no he mirado).

| Nv · # | Qué es | Dónde | Qué se rompe y quién lo nota | Sitios | ¿Revert? | Bloquea / desbloquea | Daño ÷ radio |
|---|---|---|---|---|---|---|---|
| 1 · 2 | `POST` devuelve la última fila de la tabla, no la insertada | `routes.ts:330` | Con dos comentarios solapados, una persona recibe el comentario y el correo de la otra | 1 | si | Abarata el #1 (1 de sus 3 sitios) | 2 ÷ 1 = **2,0** |
| 1 · 4 | Clave de moderación escrita en el código | `routes.ts:25`, `:351` | Quien lea el repo borra cualquier comentario | 2 | si | Desbloquea el 3a | 3 ÷ 2 = **1,5** |
| 1 · 1 | Las respuestas devuelven la fila cruda, con `autor_email` | `routes.ts:153→171`, `:330→333`, `:389→392` | Cualquiera que llame a la API ve el correo de cada comentarista (regla 6) | 3 | si | No bloquea al 3b | 3 ÷ 3 = **1,0** |
| 1 · 7 | Validación desigual entre `POST` y `PATCH`; `padre_id` sin validar | `routes.ts:319`, `:384`, `:308→326` | Comentarios editados a 1 carácter; respuestas a padres inexistentes | 2 | si | Desbloquea el #10 (si se construye) | 2 ÷ 2 = **1,0** |
| 1 · 8 | `estado` es texto libre y no hay FK | migración `:14`, `:18`, `:20`; `routes.ts:153` | `'Aprobado'` y `'ok'` no salen en el detalle (sin confirmar en la base real) | 8 | si | Ninguno | 2 ÷ 8 = **0,25** |
| 1 · 6 | Cuatro formas de error en `POST` | `routes.ts:311`, `:314`, `:317`, `:320`; `Comentarios.jsx:44` | El cliente no sabe qué error esperar; un cuerpo corto vuelve como 200 | 5 | si | Ninguno | 1 ÷ 5 = **0,2** |
| 1 · 9 | Dos recuentos de comentarios en la web | `DetallePost.jsx:30,38,71`; `Comentarios.jsx:11,23,39,54` | Dos números distintos en pantalla tras comentar | 7 | si | Ninguno | 1 ÷ 7 = **0,14** |
| 2 · 3a | Contención del #3: `DELETE` y `PATCH` solo con clave de moderación | `routes.ts:350-353`, `:378-379` | Corta borrar o editar lo ajeno con solo un correo; los autores pierden esa vía hasta el 3b | 2 | el código se revierte, los correos ya servidos y los datos ya migrados no | Bloqueado por el #4 | 3 ÷ 2 = **1,5** |
| 2 · 10 | La web no crea respuestas ni pinta las de segundo nivel | `Comentarios.jsx:33`, `:50`, `:63-64` | El lector no puede responder | 3 | si | Tras el #7 (si se decide) | 1 ÷ 3 = **0,33** |
| 3 · 5 | Borrar y editar usan reglas de autoría distintas | `routes.ts:352-353`, `:379` | Un correo con otras mayúsculas borra pero no edita; el moderador no edita | 2 | el código se revierte, los correos ya servidos y los datos ya migrados no | Bloqueado por el 3a; abarata el 3b | 2 ÷ 2 = **1,0** |
| 4 · 3b | Arreglo de fondo del #3: identidad real en vez de correo | `routes.ts:350`, `:353`, `:378-379` | Devuelve a los autores editar y borrar lo suyo | 2 | el código se revierte, los correos ya servidos y los datos ya migrados no | Bloqueado por el #5 y por una decisión de producto | 2 ÷ 2 = **1,0** |

## Detalle por hallazgo

Citas tal cual están escritas, y lo que la tabla no deja decir.

### #2: `POST` devuelve la última fila de la tabla
- Cita: `routes.ts:330` `'select * from comentarios order by id desc limit 1'`
- Se rompe: si dos personas comentan casi a la vez (en cualquier post), una recibe el comentario de la otra, con su correo. Quien comenta no lo nota; lo nota la otra persona.
- Sitios: código 1; BD 0; web 0. Fuera del repo: posibles, no contados.
- Relación: abarata el #1 solo en parte, porque comparte el sitio `routes.ts:330→333` (1 de los 3 del #1). No es previo del #1.
- Daño 2: sería 3 si ocurre, pero solo ocurre con dos comentarios solapados y no lo he verificado.

### #4: clave de moderación escrita en el código
- Citas: `routes.ts:25` `const SECRETO_SESION = 'dev-secret-cambiar'`; `routes.ts:351` `const esModerador = clave === SECRETO_SESION`
- Se rompe: quien lea el repo o adivine el valor por defecto borra cualquier comentario con la cabecera `x-clave-moderacion`. Una rotación de la clave exige tocar el código.
- Sitios: código 2 (`:25` y `:351`); BD 0; web 0. Fuera del repo: quien envíe la cabecera `x-clave-moderacion` no está en el repo, no contado.
- Relación: desbloquea el 3a. La contención deja esta clave como única puerta de `DELETE` y `PATCH`, y mientras sea la de `:25` no corta nada.
- Daño 3: cualquiera que lea el repo borra cualquier comentario.

### #1: las respuestas devuelven la fila cruda, con `autor_email`
- Citas:
  - `routes.ts:153` `"select * from comentarios where post_id = ? and estado = 'aprobado' order by creado_en asc"` → `routes.ts:171` `comentarios: comentarios,`
  - `routes.ts:333` `return response.status(201).send({ comentario: nuevos[0] })`
  - `routes.ts:392` `return response.send({ comentario: actualizados[0] })`
- Se rompe: cualquiera que llame a `GET /posts/:slug` ve el correo de cada comentarista, lo que viola la regla 6 del dominio. El frontend no lo pinta, pero llega en la respuesta.
- Sitios: código 3 (las consultas de `:153`, `:330` y `:389`); BD 0; web 0. Consumidores fuera del repo: posibles, no contados.
- Relación: no bloquea al 3b. Reduce lo explotable que es el #3 si el correo solo se conoce por esta vía (no verificado).
- Daño 3: publica el correo de cada comentarista.

### #7: validación desigual entre `POST` y `PATCH`
- Citas:
  - `routes.ts:319` `if (!cuerpo \|\| String(cuerpo).length < 5) {` (POST exige 5)
  - `routes.ts:384` `if (!cuerpo) {` (PATCH acepta 1 carácter)
  - `routes.ts:308` `const padre = request.input('padre_id', null)` → `routes.ts:326` `[post.id, nombre, email, cuerpo, 'pendiente', creado, padre]` (`padre_id` se guarda sin comprobar)
- Se rompe: se puede editar un comentario hasta dejarlo en 1 carácter. Se puede crear una respuesta a un comentario inexistente o de otro post. Lo notaría quien vea comentarios que no salen o salen mal anidados (no lo he comprobado en datos).
- Sitios: código 2; BD 0; web 0. Fuera del repo: posibles, no contados.
- Relación: desbloquea el #10 solo si alguien construye el botón de responder (no decidido); la web enviaría `padre_id`, que hoy no se valida.
- Daño 2: comentarios de 1 carácter y respuestas a padres inexistentes.

### #8: `estado` es texto libre y no hay FK
- Citas:
  - `1769000000005_create_comentarios_table.ts:18` `table.string('estado').notNullable().defaultTo('pendiente')`
  - `:14` `table.integer('post_id').notNullable()` (sin FK)
  - `:20` `table.integer('padre_id').nullable()` (sin FK)
  - `:4-6` el comentario de la migración dice que se colaron `'Aprobado'` y `'ok'`
  - `routes.ts:152-153` filtra con `estado = 'aprobado'` exacto
- Se rompe: según el comentario de la migración, hay comentarios con `'Aprobado'` u `'ok'` que el detalle nunca muestra. El seeder siembra 3 de 9 así (`blog_seeder.ts:510, 522, 575`); no he consultado la base real para confirmar que existen.
- Sitios: BD 4 (1 migración nueva + 3 filas sembradas); código 4; web 0. No cuento la base ya creada en `backend/tmp/db.sqlite3` ni otros entornos con datos.
- Daño 2: comentarios aprobados que no se ven (sin confirmar en la base real).

### #6: cuatro formas de error en `POST`
- Citas:
  - `routes.ts:311` `status(400).send({ error: ... })`
  - `routes.ts:314` `status(422).send({ message: 'Falta el email' })`
  - `routes.ts:317` `status(400).send({ errores: ['email no válido'] })`
  - `routes.ts:320` `return response.send({ ok: false, motivo: 'comentario demasiado corto' })` (200)
  - `Comentarios.jsx:44` `setAviso(d.error \|\| d.message \|\| d.motivo \|\| 'No se ha podido enviar.')`
- Se rompe: cualquier cliente nuevo tiene que adivinar cuál de las cuatro formas toca. Un cuerpo corto responde 200 y un cliente que mire solo el status lo da por bueno. Un error `errores` mostraría el aviso genérico (`Comentarios.jsx:44` no lo lee).
- Sitios: código 4; web 1; BD 0. Fuera del repo: clientes que dependan de status o forma, no contados.
- Daño 1: el cliente no sabe qué error esperar; un cuerpo corto vuelve como 200.

### #9: dos recuentos de comentarios en la web
- Citas:
  - `DetallePost.jsx:30` `const [numComentarios, setNumComentarios] = useState(0)`, `:38` `setNumComentarios((d.comentarios \|\| []).length)`, `:71` `{numComentarios} comentarios`
  - `Comentarios.jsx:11` `const [total, setTotal] = useState(0)`, `:39` `setTotal(total + 1)`, `:54` `<h2>{total} comentarios</h2>`
- Se rompe: tras comentar, la cabecera del artículo y el título de la sección muestran números distintos. El título sube en 1 aunque el comentario siga pendiente y no esté en la lista. Lo nota el lector.
- Sitios: web 7 líneas en 2 archivos (`DetallePost.jsx:30, 38, 71` y `Comentarios.jsx:11, 23, 39, 54`); código 0; BD 0.
- Daño 1: dos números distintos en pantalla tras comentar.

### 3a: contención del #3
- Citas:
  - `routes.ts:350` `const email = request.input('autor_email', '')`
  - `routes.ts:353` `String(comentario.autor_email).toLowerCase().trim() === String(email).toLowerCase().trim()`
  - `routes.ts:378-379` `const email = request.input('autor_email', '')` / `if (comentario.autor_email !== email \|\| ...`
- Qué es: dejar `DELETE` y `PATCH` solo para quien traiga la clave de moderación, sin la vía «correo coincide».
- Se rompe: corta hoy que quien conozca un correo borre o edite el comentario de esa persona, siempre que la clave ya no sea la de `routes.ts:25` (#4). Coste: los autores legítimos dejan de poder borrar o editar lo suyo hasta el 3b, y cualquier cliente externo que use esa vía recibe 403.
- Sitios: código 2 (el bloque de `DELETE` `:350-353` y el de `PATCH` `:378-379`); BD 0; web 0, porque el frontend no llama a DELETE ni a PATCH. Fuera del repo: posibles, no contados.
- Daño 3: borrar o editar lo ajeno con solo saber un correo.

### #10: la web no crea respuestas ni pinta las de segundo nivel
- Citas:
  - `Comentarios.jsx:33` `body: JSON.stringify({ autor_nombre: nombre, autor_email: email, cuerpo: cuerpo })` (no envía `padre_id`)
  - `Comentarios.jsx:50` `const raices = comentarios.filter((c) => !c.padre_id)`
  - `Comentarios.jsx:63-64` `comentarios.filter((r) => r.padre_id === c.id)`
- Se rompe: la API acepta `padre_id`, pero el formulario no lo manda: la web no tiene botón de responder. Una respuesta a una respuesta se queda en `comentarios` y no aparece en ningún sitio. El seeder trae una respuesta (`blog_seeder.ts:500`) y ninguna de segundo nivel. Lo nota el lector, si el dato existe.
- Sitios: web 3; código 0; BD 0.
- Daño 1: el lector no puede responder ni ve las respuestas de segundo nivel.

### #5: reglas de autoría distintas en borrar y editar
- Citas:
  - `routes.ts:352-353` (borrar: acepta moderador, compara sin mayúsculas y borra en cualquier estado)
  - `routes.ts:379` `if (comentario.autor_email !== email \|\| comentario.estado !== 'pendiente') {` (editar: sin moderador, sensible a mayúsculas, solo si está pendiente)
  - `routes.ts:375-377` el propio comentario del código lo reconoce: «la regla se volvió a escribir a mano y salió distinta»
- Se rompe: una persona que escribió su correo con otra capitalización puede borrar pero no editar. Un moderador puede borrar pero no editar. Tras el 3a quedaría solo lo que el 3a no toque (que `PATCH` exige estado `pendiente` y `DELETE` no).
- Sitios: código 2; BD 0; web 0. Fuera del repo: posibles, no contados.
- Relación: bloqueado por el 3a, que reescribe las mismas líneas. Abarata el 3b: deja una sola regla para heredar.
- Daño 2: quien tiene derecho no puede editar; sin pérdida de datos.

### 3b: arreglo de fondo del #3
- Citas: las mismas líneas que el 3a: `routes.ts:350`, `:353`, `:378-379`.
- Qué es: autorizar borrar y editar con una identidad real y no con un correo que manda el cliente, usando la regla única que deje el #5.
- Se rompe: devuelve a los autores legítimos la posibilidad de editar y borrar lo suyo, que el 3a les quitó.
- Sitios: código 2; BD 0; web 0. Fuera del repo: posibles, no contados.
- Relación: bloqueado por el #5 y por una decisión de producto que no está en el repo. Hoy comentar es anónimo (nombre y correo libres), así que hay que decidir qué es una «identidad real» de quien comenta.
- Daño 2: mientras dure el 3a, el autor legítimo no puede editar ni borrar lo suyo.

## Cómo está ordenada la tabla

1. **Precedencia.** Un hallazgo que desbloquea o abarata a otro va antes que el otro, pero solo cuenta la relación que se apoya en lo leído en el código. Una relación que depende de algo no verificado se anota en la última casilla y no mueve la fila de nivel. Las filas se agrupan en cuatro niveles (columna «Nv»): el nivel 1 no espera a nadie y cada nivel siguiente espera a una fila del anterior.
2. **Dentro de cada nivel: daño ÷ radio.**
   - **Radio.** Un sitio es una línea o un bloque contiguo que el arreglo edita. Una fila sembrada o una migración nueva cuenta 1. Esta unidad es la misma en todas las filas: antes el #4 contaba 1 «secreto» y el #9 contaba líneas, y eso inflaba el cociente del #4.
   - **Daño.** Es una nota mía de 1 a 3 por ocurrencia, no una medida. **3** = filtra datos personales o permite borrar o alterar contenido ajeno; **2** = contenido o comportamiento incorrecto para alguien, sin filtración ni pérdida; **1** = incoherencia visible o de contrato, sin pérdida. Si el defecto solo ocurre en una circunstancia que no he verificado, bajo la nota un punto y lo digo en el detalle.
   - **Empates.** Pasa delante el que desbloquea a otro por una relación verificada en el código. Si no hay ninguno, pasa delante el de más daño.
3. **Fila partida.** El #3 tiene daño 3 y su arreglo de fondo es de los últimos en llegar, así que va en dos filas: **3a**, una contención que corta el daño hoy, y **3b**, el arreglo de fondo, que espera su turno.

### Qué cambió respecto a la versión anterior
- El #4 pasa a contar 2 sitios (`routes.ts:25` y `:351`) y baja a 1,5.
- El #4 queda declarado como previo al 3a. Con la clave en `routes.ts:25` como única puerta, la contención no corta nada.
- El #2 baja a daño 2: solo ocurre si dos comentarios se solapan, y eso no está verificado.
- El #1 sube al nivel 1. Lo que lo retrasaba era que el #2 «lo abarata», pero eso solo cubre 1 de sus 3 sitios.
- El #5 pasa detrás del 3a. El 3a quita la vía del correo de las mismas líneas, y el #5 queda reducido a lo que el 3a no toca.
- El 3b ya no espera al #1. El #1 solo reduce lo explotable que es el #3, no lo bloquea. Lo que sí espera es una decisión de producto que no está en el repo.

## Propuestas sin medir

Salen de la tabla porque no hay ninguna medida detrás. Cada una dice qué habría que medir para decidirla.

- **Índice en `post_id` de `comentarios`** (salía en el #8 como «sin índice»). Hay que medir:
  1. cuántas filas tiene la tabla en el entorno real (el seeder solo trae 9);
  2. el plan de consulta (`EXPLAIN QUERY PLAN`) de `routes.ts:153`, `:223` y `:75-76`;
  3. el tiempo de `GET /posts/:slug` con ese volumen.

  Si la tabla es pequeña y el plan ya es aceptable, no se justifica.

En la tabla no había ninguna otra propuesta de rendimiento, paginación ni antispam, así que no hay más que sacar.

## Notas

- Fuera de alcance, no auditado: el recuento por SQL en listados y relacionados (`routes.ts:75-76` y `:223`), que usa `estado <> 'pendiente'` y no `= 'aprobado'`. Pertenece a posts.
- Fuera de alcance, no auditado: `Comentarios.jsx:4` y `DetallePost.jsx:7` repiten la constante `API`; el comentario del propio código habla de «cuarta copia».
- No he comprobado la base de datos real, no he ejecutado la app y no he leído los tests.


El primer prompt ha sacado 10 casos


Línea 1: los dos números
Devueltos: 10. Comprobados abriendo el archivo: 8.

Línea 2: el hallazgo que no supe priorizar
No supe decidir el orden del #2 frente al #4/3a: el #2 sale primero por radio 1 (daño ÷ radio = 2,0), pero su daño no está verificado; el #4/3a permiten borrar lo ajeno y sí se demuestran leyendo el código, y quedan en 1,5. Se contradicen lo barato con lo demostrable.

Línea 3: lo que no pudiste verificar
Consumidores fuera del repo de DELETE/PATCH con correo o con x-clave-moderacion. El 3a les devolvería 403 y no hay forma de saber si existen.
Que el ok y los Aprobado existan en la base real. El seeder los siembra, pero tmp/db.sqlite3 y otros entornos no los has mirado.
Que dos envíos solapados crucen comentarios (#2).