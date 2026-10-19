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

**Modelo:** Opus 5.5 High Effort
**Herramienta:** Claude Code

```
Audita la feature "comentarios" de este proyecto.
Es decir, audita las funcionalidades (endpoints) para crear, editar, borrar y listar comentarios, en las 3 capas que le afectan:
 - endpoints: rutas de la API
 - tabla donde se almacena en la base de datos
 - qué se muestra en la web de esta tabla

 Encuentra los problemas que hay respecto a esta feature. Sólamente respecto a esta feature en las 3 capas mencionadas anteriormente. No audites problemas de otras features (posts, autores, categorías y etiquetas).

 Muestra una tabla con los hallazgos (una afirmación concreta sobre el sistema que se puede comprobar, si no se puede comprobar no es un hallazgo, es una impresión.)
 Cada linea de la tabla es un hallazgo.
 Por cada hallazgo tiene que haber 6 columnas:
  1. "Qué es". Una frase. Si necesitas tres, es que hay dos hallazgos.
  2. "Dónde está". Fichero y la línea que lo demuestra. Sin esta casilla, la fila no se escribe.
  3. "Qué se rompe, y quién lo nota". El efecto concreto y a quién le llega. "Nadie todavía" es una respuesta válida y buena.
  4. "Cuántos sitios hay que tocar". En cuantos sitios habría que cambiar código para corregirlo. Contados, no estimados.
  5. "¿Se deshace con un revert?"  Sí o no, y si es que no, por qué no. Si, significa que los cambios comiteados para subsanar el hallazgo, en una futura versión se deshacen con un revert de esos cambios. Es decir, la ejecución de la aplicacion con la corrección aplicada, no ha generado datos almacenados que al hacer el revert hiciesen fallar a la aplicación.
  6. "¿Bloquea o desbloquea a otro hallazgo?" si bloquea, nombra al otro hallazgo. "A ninguno" es una respuesta válida. Un hallazgo "A" bloquea (o desbloquea) a otro "B", cuando solucionar el hallazgo "A" es parte de la solución del "B". O dicho de otra manera, para solucionar "B" es también necesario solucionar "A"

Los hallazgos reportados, debe ser posible comprobar por un humano comprobar cada uno de los puntos de la tabla.

Ordena la tabla por los siguientes factores:
 1º Precedencia: Un hallazgo que bloquea a otro irá antes en la lista (mirar la columna 6 de la tabla)
 2º Daño dividido por el ratio: Evalúa el daño teniendo en cuenta la columna 2 (si nadie lo nota todavía no es ni siquiera un daño, es un riesgo, por lo estaría en lo más bajo de la tabla). A mismo daño el que tiene más ratio (más sitios donde afecta, columna 4). Va más abajo en la tabla.
Si un hallazgo tiene el daño más alto y la precedencia más baja a la vez, se parte en dos filas, una fila sería un hallazgo cuyo fix de contención que corta el daño hoy y es parte del arreglo total. El arreglo total sería la otra fila.

Si hay propuestas de mejora, que no son daños medibles y o revisables, van fuera de la tabla, y se indica que habría que medir para ver como mejorarlo (por ejemplo, una mejora de performance)

No escribas esta auditoría en el chat, crea el documento docs/auditoria/comentarios.md y escríbela ahí.


```

**Qué salió:** Funcionó
