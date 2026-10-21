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

**Modelo:** Sonnet 5.5 xHigh
**Herramienta:** Claude Code

```
Se necesita una auditoria de la feature de comentarios, en busca de fallos o incosistencias.

Crea una tabla y para cada hallazgo crea una fila con el siguiente formato de tabla:

1. Qué es. Una frase. Si necesitas tres, es que hay dos hallazgos.

2. Dónde está, con la línea que lo demuestra. Sin esta casilla, la fila no se escribe.

3. Qué se rompe, y quién lo nota. El efecto concreto y a quién le llega. "Nadie todavía" es una respuesta válida y buena.

4. Cuántos sitios hay que tocar. Contados, no estimados.

5. ¿Se deshace con un revert? Sí o no, y si es que no, por qué no.

6. ¿Bloquea o desbloquea a otro hallazgo? Nombrando al otro. "A ninguno" es la respuesta más frecuente.

I guárdalo en docs/auditoria/comentarios.md
```

**Qué salió:** funcionó a la primera 