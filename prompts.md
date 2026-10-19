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

**Modelo:** Sonnet 5.5 Medium
**Herramienta:** Claude Code

```
Audita SOLO la feature de comentarios de este blog, de punta a punta. Alcance exacto:
- API: las rutas de comentarios de backend/start/routes.ts (crear, editar, borrar) y la parte de comentarios del detalle del post.
- Base de datos: la migración de la tabla comentarios.
- Web: frontend/src/componentes/Comentarios.jsx y la parte de comentarios de DetallePost.jsx.
No audites posts, autores, categorías ni etiquetas.

Reglas:
- Máximo 10 hallazgos.
- Cada hallazgo lleva archivo:línea y el fragmento citado tal cual está escrito. Si no has abierto el archivo y no puedes citar la línea, el hallazgo no se escribe.
- Si el mismo defecto aparece en varios sitios, es UN hallazgo con la lista de sitios.
- No pongas severidad ni prioridad: la decido yo.
- Escribe el resultado en docs/auditoria/comentarios.md como una tabla con una fila por hallazgo y estas columnas: Qué es (una frase) | Dónde está (archivo:línea + cita) | Qué se rompe y quién lo nota | Cuántos sitios hay que tocar | ¿Se deshace con un revert? | ¿Bloquea o desbloquea a otro hallazgo?
- Si no puedes establecer una casilla con lo que has leído, déjala en blanco. No la rellenes por plausibilidad.
- No modifiques ningún otro archivo.
- Termina diciendo cuántos hallazgos has escrito y qué archivos has abierto de verdad.
```

**Qué salió:** 
He escrito 10 hallazgos en docs/auditoria/comentarios.md. No he tocado ningún otro archivo.

Archivos que he abierto de verdad

backend/start/routes.ts: las líneas 96-175 y 288-398, más una búsqueda de «comentario» y otra de SECRETO_SESION.
backend/database/migrations/1769000000005_create_comentarios_table.ts
frontend/src/componentes/Comentarios.jsx
frontend/src/componentes/DetallePost.jsx
Casillas en blanco

La columna «¿Se deshace con un revert?» está en blanco en las 10 filas. Depende del historial y de los datos, y no los he mirado.
«¿Bloquea o desbloquea?» solo está rellena en el hallazgo 1. Que GET /posts/:slug devuelva el correo de quien comenta desbloquea el hallazgo 3, porque ese correo es lo único que se pide para borrar o editar.
Lo que no he hecho

No he consultado la base real, así que no sé si existen filas con 'Aprobado' u 'ok'. Esa parte del hallazgo 8 sale solo del comentario de la migración.
No he ejecutado la app.
No he leído los tests.
Dejé fuera de alcance, y lo anoté al final del archivo, el recuento por SQL de los listados (routes.ts:75 y :223) y la constante API repetida.

## Prompt 2

**Modelo:** Sonnet 5.5 Medium
**Herramienta:** Claude Code

```
Para cada hallazgo de docs/auditoria/comentarios.md, rellena la casilla "Cuántos sitios hay que tocar" con un número contado, no estimado. Para cada uno enséñame el comando grep o la búsqueda que has usado y su salida literal, y lista los archivos:línea contados. Distingue sitios del código, de la base de datos (migración o datos sembrados) y de la web. Si un consumidor puede estar fuera de este repositorio, dilo explícitamente en lugar de contarlo.
```

**Qué salió:** 
Rellené la columna «Cuántos sitios hay que tocar» de los 10 hallazgos en docs/auditoria/comentarios.md. Cada cifra sale de las búsquedas de abajo. El archivo había cambiado en disco desde que lo escribí, pero las ediciones se aplicaron sin conflicto y no lo releí.

Búsquedas usadas
Las ejecuté con la herramienta Grep (ripgrep), salvo la última, que fue grep -r por Bash. Excluí node_modules, coverage, docs, .git y los archivos prompts.md, CLAUDE.md y README.md. La salida literal está más arriba, en las llamadas.

#	Búsqueda	Qué muestra
A	autor_email en todo el repo	Hits en routes.ts, el seeder, las migraciones, el test, Comentarios.jsx:33, el Makefile y CLAUDE.md
B	comentarios|comentario\b (sin distinguir mayúsculas)	Todos los usos de la tabla y del concepto
C	padre_id|SECRETO_SESION|x-clave-moderacion|numComentarios|setTotal|\btotal\b	Líneas de la clave, de padre_id y de los recuentos
D	grep -rnE "DELETE|PATCH|method:" frontend/src	Un solo hit: Comentarios.jsx:31 method: 'POST'
E	grep -rnE "SECRETO_SESION|x-clave-moderacion|dev-secret" .	Solo routes.ts:25, :349 y :351
F	Read de Makefile:40-60, schema.ts:70-100 y la búsqueda estado: en el seeder	Ver los hallazgos 1 y 8
Recuento por hallazgo
#	Total	Código	BD	Web	Sitios contados
1	3	3	0	0	routes.ts:153→171, :330→333, :389→392
2	1	1	0	0	routes.ts:330
3	2	2	0	0	routes.ts:350,353 (DELETE) y :378,379 (PATCH)
4	1 secreto	2 líneas	0	0	routes.ts:25 y :351
5	2	2	0	0	routes.ts:352-353 y :379
6	5	4	0	1	routes.ts:311,314,317,320; Comentarios.jsx:44
7	2	2	0	0	routes.ts:319 y :384; :308 y :326
8	8	4	4	0	Ver abajo
9	7 líneas	0	0	7	DetallePost.jsx:30,38,71; Comentarios.jsx:11,23,39,54
10	3	0	0	3	Comentarios.jsx:33,50,64
El hallazgo 8 se desglosa así:

BD (4):
la migración, que son tres columnas (:14, :18, :20);
tres filas sembradas con estado no canónico: blog_seeder.ts:510 ('Aprobado'), :522 ('ok') y :575 ('Aprobado').
Código (4): routes.ts:153, :326 y :379, más database/schema.ts:74-91, que parece generado.
Encontré cosas que no esperaba:

Hallazgo 3: la web no llama nunca a DELETE ni a PATCH (búsqueda D). Esas dos rutas solo las usaría un cliente externo.
Hallazgo 10: el seeder sí tiene una respuesta (padre_id: 1 en blog_seeder.ts:500, aprobada) y ninguna de segundo nivel. La parte de «segundo nivel no se pinta» sigue sin datos que la ejerzan.
Hallazgo 1: el Makefile:51-53 (make fuga) busca "autor_email" en el detalle del post. Ese hit viene del correo del autor del post, no del de quien comenta, y no lo cuento. Después de arreglar este hallazgo, make fuga seguiría dando positivo por el correo del autor, así que no sirve para comprobarlo.
Hallazgo 8: estado aparece también en routes.ts:75-76 y :223 (<> 'pendiente'). Es de posts y no lo cuento.
Consumidores fuera del repositorio
La API escucha en el 3401 y no hay autenticación. Cualquier cliente externo (scripts, apps móviles, herramientas de moderación) puede depender de los hallazgos 1, 2, 3, 4, 5, 6 y 7. Lo dejé escrito en cada celda y no los conté. Tampoco cuento la base ya creada en backend/tmp/db.sqlite3 ni otros entornos con datos (hallazgo 8). El hallazgo 9 es solo del frontend de este repo.

Qué no comprobé
Mis búsquedas respetan .gitignore, así que no habrían visto archivos ignorados como un .env real. La búsqueda E sí corrió sin ese filtro, y en el repo hay solo .env.example y .env.test, que no contienen la clave. No abrí backend/tmp/db.sqlite3.


## Prompt 3

**Modelo:** Sonnet 5.5 Medium
**Herramienta:** Claude Code

```
Reordena la tabla de docs/auditoria/comentarios.md con este criterio, en este orden:
1. Precedencia: un hallazgo que desbloquea o abarata a otro va antes que el otro. Nombra el hallazgo bloqueado en la última casilla.
2. Dentro de cada nivel de precedencia: daño dividido por radio (radio = casilla "sitios a tocar"). Explica el daño en una frase.
Si un hallazgo tiene a la vez el daño más alto y la precedencia más baja, pártelo en dos filas: una contención que corta el daño hoy y el arreglo de fondo que espera su turno.
Las propuestas que nadie ha medido todavía (rendimiento, paginación, antispam…) sácalas de la tabla y ponlas debajo, cada una con lo que habría que medir para decidirla.
No uses etiquetas de severidad. No añadas hallazgos nuevos. No cambies ninguna casilla que yo haya dejado en blanco.
```

**Qué salió:** 
Reordené la tabla de docs/auditoria/comentarios.md en tres niveles de precedencia y partí el #3 en dos filas. No ejecuté nada: es solo una reescritura del documento.

Orden resultante

Nivel 1 (no esperan a nadie): #2, #4, 3a, #5, #7, #8, #6, #9.
Nivel 2: #1 (va tras el #2) y #10 (va tras el #7).
Nivel 3: 3b, que espera al #1 y al #5.
Precedencias que usé. Salen de las líneas que ya había leído:

El #2 abarata el #1, porque el sitio routes.ts:330→333 es el mismo.
El #1 desbloquea el 3b, porque el correo expuesto es la credencial para borrar y editar.
El #5 abarata el 3b, porque son las mismas líneas.
El #7 desbloquea el #10, porque un botón de responder mandaría un padre_id que hoy no se valida.
Daño ÷ radio.

La nota de daño es mía. Es de 1 a 3, con la escala escrita al principio del documento, y no es una medida. Si cambias una nota, el orden dentro del nivel se recalcula con una división.
Radio del #4: lo conté como 1 (un secreto, aunque ocupe 2 líneas).
Empates: rompí los que había (#2/#4 en 3,0 y #5/#7 en 1,0) poniendo antes al que desbloquea a otro.
Fila partida. El #3 tiene daño 3 y es el último de la cadena.

3a (contención): deja DELETE y PATCH solo para quien traiga la clave de moderación. El coste es que los autores legítimos pierden temporalmente editar y borrar lo suyo. Eso es seguro para la web, que no llama a esas rutas, pero un cliente externo recibiría 403.
3b (arreglo de fondo): autoriza con una identidad real y la regla única del #5. Espera al #1 y al #5.
Propuesta sin medir. Saqué de la tabla el «sin índice» del #8, porque nadie ha medido nada. Está debajo, con lo que habría que medir:

el número de filas real;
el plan de consulta de routes.ts:153, :223 y :75-76;
el tiempo de GET /posts/:slug con ese volumen.
No había ninguna otra propuesta de rendimiento, paginación ni antispam en la tabla.

Qué cambié, aparte del orden.

Añadí una columna final «Daño ÷ radio», con la frase de daño y el cociente.
Reescribí la última casilla de bloqueos en las filas que la tenían.
En el #8 quité «ni índice» del título y de una cita.
No toqué la columna «¿Se deshace con un revert?», que sigue en blanco, ni ninguna otra casilla en blanco.
No añadí hallazgos nuevos.
Dejé al final tu línea «El primer prompt ha sacado 10 casos».


## Prompt 4

**Modelo:** Sonnet 5.5 Medium
**Herramienta:** Claude Code

```

```

**Qué salió:** 

