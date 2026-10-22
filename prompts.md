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

**Modelo:** claude-opus-5-5
**Herramienta:** Claude Code

```
❯ Quiero un resumnen de los ultimos commits y que busques en que archivos podes encontrar la palabra "comentario" o "comentarios"
```

**Qué salió:** 
```
Me encontro 3 commits y en la busqueda y 15 arhivos con la palabra comentario incluidos algunos archivos de configuracion que olvide pedir que no los tenga en cuenta.
```

## Prompt 2

**Modelo:** claude-opus-5-5
**Herramienta:** Claude Code

```
Sos un auditor Senior y necesito que elabores un informe exclusivamente sobre la feature de comentarios. El analisis debe tener en cuenta rutas en la API, tabla en la base de datos y lo que la web enseña de la feature, los flujos de crear, editar, borrar y listar
  comentarios. No quiero que revises nada mas alla de "Comentarios" salvo que alguna relacion lo requiera.
```

**Qué salió:**
```
Me encontro 3 commits y en la busqueda y 15 arhivos con la palabra comentario incluidos algunos archivos de configuracion que olvide pedir que no los tenga en cuenta.
```