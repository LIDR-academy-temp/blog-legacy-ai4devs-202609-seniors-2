# Prompts

Aquí van **todos los prompts que lanzaste** para hacer el ejercicio, en el orden en que los
lanzaste, con el modelo y la herramienta de cada uno.

Esto no es papeleo. Lo que se revisa es **cómo pediste las cosas**, no solo lo que salió: un
resultado flojo con un prompt bueno y un resultado flojo con un prompt vago necesitan feedback
distinto, y sin este archivo no se distinguen.

## Cómo rellenarlo

- Un apartado `## Prompt N` por cada prompt.
- **Pega el prompt tal cual lo lanzaste**, dentro del bloque de código, aunque ocupe diez líneas
  y aunque tenga faltas. No lo reescribas para que quede bien: el que arreglaste mentalmente
  después no es el que lanzaste.
- Incluye también los que **no funcionaron**. Suelen ser los más útiles de leer.
- `Modelo` y `Herramienta` en todos. Si cambiaste de una a otra a mitad, se nota aquí.

Borra el ejemplo de abajo cuando escribas el primero.

---

## Prompt 1

**Modelo:** Opus 5.5 Medium Effort
**Herramienta:** Claude Code

```
Actua como un Staff Engineer y/o Architect de Software con 12 anos de experiencia en sistemas de arquitecturas distribuidas, legacy, monolito y microservicios.
  NEcesito que resumas los ultimos commits y busques en que archivos aparece una palabra cualquiera del dominio
```

**Qué salió:** Funciono a la primera: Menciono que hay 3 commits y me dio el detalle, hora y autor de cada uno. Ademas, del dominio eligio la palabra "borrador" y mostro que aparece en 5 archivos.

## Prompt 2

**Modelo:** Opus 5.5 Medium Effort
**Herramienta:** Claude Code

```
Dale ve con slung (pero explicame antes q significa esa palabra en el contexto del proeycto)
```

**Qué salió:** Funciono a la primera: Me explico el concepto y luego me mostro donde aparece slung (16 archivos)

## Prompt 3

**Modelo:** Opus 5.5 Medium Effort
**Herramienta:** Claude Code

```
Actúa como un staff engineer haciendo una auditoría de código. Audita SOLO la feature de
comentarios de este proyecto y deja el resultado escrito en docs/auditoria/comentarios.md.

Es una tarea de SOLO LECTURA: no modifiques código, no crees ramas, no hagas commits, no
arregles nada. El único archivo que escribes es docs/auditoria/comentarios.md.

Qué entra (la feature de punta a punta, en sus tres capas):
- API: POST /posts/:slug/comentarios, PATCH /comentarios/:id, DELETE /comentarios/:id, y cómo
  se listan y se cuentan los comentarios dentro de GET /posts/:slug y GET /posts
  (backend/start/routes.ts, y app/exceptions/handler.ts para los errores).
- BD: la tabla comentarios, su migración y su semilla.
- Web: frontend/src/componentes/Comentarios.jsx y lo que pide a la API.
- Tests: tests/functional/comentarios.spec.ts (qué cubre y qué no).
El proyecto tiene 5 features; no audites ninguna otra, aunque veas problemas en ellas.

La vara de medir son las reglas de negocio del README (sección "Reglas de negocio"). Cada
hallazgo tiene que incumplir una regla (cítala por número) o tener un efecto concreto y
observable. Las opiniones de estilo o de arquitectura sin efecto no son hallazgos.

Evidencia: abre cada archivo y cita archivo:línea con la línea de código copiada
textualmente. Si no puedes señalar la línea que lo demuestra, la fila no se escribe.

Formato no negociable: una tabla, una fila por hallazgo, con una columna # (H1, H2...) como
identificador y estas seis casillas:
1. Qué es. Una frase. Si necesitas tres, es que hay dos hallazgos.
2. Dónde está, con la línea que lo demuestra.
3. Qué se rompe, y quién lo nota. El efecto concreto y a quién le llega. "Nadie todavía" es válido.
4. Cuántos sitios hay que tocar. Contados (lista los archivo:línea), no estimados.
5. ¿Se deshace con un revert? Sí o no, y si es no, por qué no.
6. ¿Bloquea o desbloquea a otro hallazgo? Nombrando su # . "A ninguno" es lo más frecuente.

Orden de la tabla:
- Primero por precedencia: va antes lo que abarata o desbloquea a otros hallazgos.
- Dentro de cada nivel de precedencia, por daño multiplicado por radio.
  Daño: 3 = filtra datos o rompe una regla de negocio, 2 = comportamiento incorrecto,
  1 = inconsistencia sin efecto visible. Radio: 3 = cualquier visitante, 2 = quien comenta
  o el autor, 1 = nadie todavía.
- Si un hallazgo tiene el daño más alto y la precedencia más baja, pártelo en dos filas: una
  contención que corta el daño hoy y el arreglo de fondo que espera su turno.
- Las propuestas que nadie ha medido van FUERA de la tabla, en una sección aparte, con lo que
  habría que medir escrito al lado.

Escribe el archivo fila a fila según vayas confirmando cada hallazgo, empezando por los más
graves, para que si se corta a medias quede lo que ya está verificado.

Cuando termines, dime en el chat (NO en el archivo): cuántos hallazgos escribiste en la tabla
y qué afirmaciones hiciste que no pudiste verificar leyendo código (por ejemplo, cosas que
dependen de cómo se comporta en ejecución).
```

**Qué salió:** Funciono a la primera: 18 hallazgos y 6 propuestas fuera de la tabla, todas las citas existen.

## Prompt 4

**Modelo:** Opus 5.5 Medium Effort
**Herramienta:** Claude Code

```
 1. Ya termino el agente
 2. El resultado del otro agente q me devolvio:
 
  He escrito 18 hallazgos en la tabla (H1–H18) de docs/auditoria/comentarios.md, más una sección aparte con 6 propuestas sin medir, cada una con lo que habría que medir. No he tocado nada más: ni código, ni ramas, ni commits.

  Lo que conviene mirar primero
  - H1 / H5: GET /posts/:slug devuelve el correo de todos los que han comentado, porque los comentarios salen con select * (incumple la regla 6). Lo he partido en dos filas: H1 es una contención de una línea que corta la fuga hoy, y H5 es el arreglo de fondo con un transformer, que espera a que haya tests.
  - H6: para borrar un comentario basta con enviar el correo de quien lo escribió. Como ese correo es público por H1, cualquier visitante puede borrar cualquier comentario aprobado.
  - H2 → H4: el campo estado es texto libre y la semilla tiene comentarios moderados como 'Aprobado' y 'ok', que el detalle no enseña. El listado cuenta todo lo que no es 'pendiente' y el detalle solo cuenta 'aprobado', así que los números no coinciden (regla 5). Antes de unificar el recuento hay que decidir qué cuenta como aprobado, y eso es una decisión de producto.
  - H3: el único test de la feature comprueba un solo campo y ninguna ausencia. La suite está en verde con todos los demás hallazgos presentes.

  Afirmaciones que no pude verificar leyendo código (no he ejecutado la API ni los tests):
  1. Los números de la semilla (por ejemplo, «el post 1 dice 4 en el listado y 2 en el detalle», «el post 6 dice 1 y luego 0»). Los calculé leyendo el SQL y la semilla; supongo que la base local está cargada con esa semilla. Lo aviso también en la cabecera del archivo.
  2. H7, la condición de carrera del POST. Que una persona reciba el comentario de otra depende de que dos peticiones se crucen entre el insert y el select. Supongo que Lucid cede el control entre los dos await aunque better-sqlite3 sea síncrono. No lo he reproducido.
  3. H17, la clave de moderación. Que el radio pase a «cualquier visitante» depende de que el repositorio de GitHub sea público, y no lo he comprobado.
  4. H11 / H12, el formato de error tras el arreglo. En la columna 4 supongo que un validador VineJS devolvería por defecto un 422 con un formato de error distinto, y que por eso habría que tocar Comentarios.jsx:44. No lo he comprobado en el framework instalado.
  5. H10 solo se ve si falla la segunda petición del detalle. No lo he provocado.
  6. Regla 5: que 'Aprobado' y 'ok' signifiquen «aprobado» es una lectura de los comentarios de la semilla, no algo que diga el código.
     DOnde debo guardar esto?


3. Puedes verificarlo tu tambien y yo lo hago al tiempo ?  Debo verificar todos los hallazgos o N cantidad?
```

**Qué salió:** Funciono a la primera: Se validaron las citas de los 18 hallazgos (60 Citas) y me menciono que los hallazgos q alcanzara a validar en el tiempo del reloj

