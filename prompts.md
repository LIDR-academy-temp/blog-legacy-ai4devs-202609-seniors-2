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

**Modelo:** Opus 5.5 Medium
**Herramienta:** Claude Code

```

genera un resumen de los últimos 10 commits del repositorio

```

**Qué salió:** menciona que el repositorio solo tiene 3 commits, muestra resumen de cada uno (el primero sube todo el código, los otros 2 ajustan documentación relacionada al curso)

## Prompt 2

**Modelo:** Opus 5.5 Medium
**Herramienta:** Claude Code

```

Lee primero README.md, CLAUDE.md para identificar las reglas de negocio y las convenciones de este repositorio.

Realiza una auditoría exclusivamente de la funcionalidad de comentarios del proyecto Blog Legacy, incluyendo todas las capas: API, base de datos y frontend, y operaciones crear, editar, borrar y listar comentarios.
El formato es el de una tabla donde cada fila debe contener lo siguiente:
1. Qué es. Una frase. Si necesitas tres, es que hay dos hallazgos.
2. Dónde está, con la línea que lo demuestra. Sin esta casilla, la fila no se escribe.
3. Qué se rompe, y quién lo nota. El efecto concreto y a quién le llega. "Nadie todavía" es una respuesta válida y buena.
4. Cuántos sitios hay que tocar. Contados, no estimados.
5. ¿Se deshace con un revert? Sí o no, y si es que no, por qué no.
6. ¿Bloquea o desbloquea a otro hallazgo? Nombrando al otro. "A ninguno" es la respuesta más frecuente.

El orden del resultado: primero la precedencia (lo que abarata o desbloquea a otro hallazgo va antes), y dentro de cada nivel, el daño dividido por el radio. Si un hallazgo tiene el daño más alto y la precedencia más baja a la vez, se parte en dos filas, una contención que corta el daño hoy y el arreglo de fondo que espera su turno. Y las propuestas que nadie ha medido todavía van fuera de la tabla, con lo que habría que medir escrito al lado.
El resultado guardalo en "docs/auditoria/comentarios.md" 
No modifiques el código. 

```

**Qué salió:** generó el documento solicitado con 23 hallazgos y la info para cada uno de ellos que permitió completar el ejercicio.
