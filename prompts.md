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

**Modelo:** Opus 5.5 (`claude-opus-5-5`)
**Herramienta:** Claude Code CLI v2.1.303, en WSL, dentro del repo (modo auto)

```
Audita la feature de comentarios de este repo: crear, editar, borrar y listar
comentarios, en sus tres capas: las rutas de la API (backend/start/routes.ts),
la tabla comentarios en la base de datos (migración y seeder) y lo que muestra
la web (frontend/src). Solo esa feature: no audites posts, autores, categorías
ni etiquetas aunque los veas.

No arregles nada: solo lee y audita. No modifiques código ni tests.

Escribe el resultado en docs/auditoria/comentarios.md (no en el chat), como una
tabla con una fila por hallazgo, numeradas, con estas 6 columnas:
1. Qué es: una frase. Si necesitas más, son dos hallazgos.
2. Dónde está: archivo:línea y el fragmento de código que lo demuestra. Si no
   hay una línea que lo demuestre, no escribas la fila.
3. Qué se rompe y quién lo nota (lector, autor, quien comenta... o "nadie todavía").
4. Cuántos sitios hay que tocar, contados (di cuáles).
5. ¿Se deshace con un revert? Sí o no, y si no, por qué.
6. ¿Bloquea o desbloquea a otro hallazgo? Nómbralo por su número, o "a ninguno".

Ordena las filas así: primero por precedencia (lo que abarata o desbloquea otro
hallazgo va antes); dentro de cada nivel, por daño dividido entre radio. Si un
hallazgo tiene el daño más alto pero la precedencia más baja, pártelo en dos
filas: una contención que corte el daño hoy y el arreglo de fondo que espera
su turno.

Las propuestas que nadie ha medido van fuera de la tabla, en una sección
aparte, con lo que habría que medir para confirmarlas.

Máximo 10 hallazgos. Si una casilla no la puedes confirmar leyendo el código,
déjala en blanco en lugar de suponer. Al terminar, dime en el chat cuántos
hallazgos escribiste.
```

**Qué salió:** funcionó a la primera: en unos 3 minutos escribió 10 hallazgos en el archivo (y mencionó 3 más que no cupieron por el tope de 10), más 4 propuestas sin medir. El prompt lo armó la sesión de Claude que me guiaba, a partir del enunciado, porque al principio no entendía el ejercicio; lo revisé y lo lancé tal cual. Fue el único prompt al auditor: la comprobación fila por fila y las 3 correcciones del archivo (la frase de la fila 1 y la casilla 6 de las filas 2 y 4) las hice yo después de comprobar, no con prompts.
