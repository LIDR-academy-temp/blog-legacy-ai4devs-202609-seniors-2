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


## Prompt 1 — intento fallido (sesión caducada)

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Antes de comenzar la auditoría, comprueba que puedes trabajar con este repositorio. Ejecuta `git log -5 --oneline` y localiza todos los archivos que contienen la palabra `comentario` (sin distinguir mayúsculas y minúsculas). Devuélveme un resumen breve de los últimos commits y las rutas de los archivos encontrados. No modifiques ningún archivo ni realices todavía la auditoría.
```

**Qué salió:** el primer intento falló porque la sesión de Claude Code había caducado y requirió `/login`.

## Prompt 2 — comprobación inicial

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Antes de comenzar la auditoría, comprueba que puedes trabajar con este repositorio. Ejecuta git log -5 --oneline y localiza todos los archivos que contienen la palabra comentario (sin distinguir mayúsculas y minúsculas). Devuélveme un resumen breve de los últimos commits y las rutas de los archivos encontrados. No modifiques ningún archivo ni realices todavía la auditoría.
```

**Qué salió:** funcionó a la primera: 3 commits (el repo no tiene 5) y 13 archivos con «comentario».

## Prompt 3 — auditoría de comentarios

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Realiza una auditoría priorizada exclusivamente de la funcionalidad de comentarios de este proyecto Blog Legacy, abarcando API, base de datos y frontend: creación, edición, eliminación y listado de comentarios.

Lee primero README.md y CLAUDE.md para identificar las reglas de negocio y las convenciones. Inspecciona el código y ejecuta comprobaciones cuando aporten evidencia.

Crea docs/auditoria/comentarios.md con una tabla de seis columnas: (1) qué es el hallazgo, (2) archivo y líneas exactas que lo demuestran, (3) qué se rompe y quién lo nota, (4) número contado de lugares que habría que modificar, (5) si se revierte con un revert y por qué, (6) qué otro hallazgo bloquea o desbloquea.

Prioriza primero por precedencia y, dentro del mismo nivel, por daño dividido entre radio de impacto. Si la gravedad y la precedencia entran en conflicto, separa contención inmediata y corrección definitiva.

Distingue claramente los hallazgos propuestos de los comprobados; no inventes evidencias, líneas ni verificaciones. Deja en blanco cualquier campo que no consigas comprobar.

Al final del archivo incluye exactamente las tres respuestas obligatorias de la Parte B: cuántos hallazgos identificaste y cuántos comprobé personalmente abriendo sus archivos; cuál fue el hallazgo cuya prioridad no supe decidir y por qué; y qué afirmación tuya no pude verificar. Cuando una respuesta dependa de mi comprobación o criterio, déjala pendiente y pregúntame.

No modifiques el código funcional ni corrijas errores. Registra este prompt literalmente en prompts.md, indicando el modelo Claude Opus 5.5 y la herramienta Claude Code. Conserva también el prompt de comprobación inicial y el intento fallido por sesión caducada.
```

**Qué salió:** 21 hallazgos en `docs/auditoria/comentarios.md` (18 comprobados, 3 propuestos). La Parte B queda pendiente de tu comprobación.

## Prompt 4 — comprobación inicial, relanzada

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Antes de comenzar la auditoría, comprueba que puedes trabajar con este repositorio. Ejecuta `git log -5 --oneline` y localiza todos los archivos que contienen la palabra `comentario` (sin distinguir mayúsculas y minúsculas). Devuélveme un resumen breve de los últimos commits y las rutas de los archivos encontrados. No modifiques ningún archivo ni realices todavía la auditoría.
```

**Qué salió:** los mismos 3 commits y 15 archivos (los 13 de antes más `docs/auditoria/comentarios.md` y `prompts.md`).

## Prompt 5 — revisión de C-03

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Muéstrame el hallazgo C-03 de la auditoría y el código fuente exacto que lo demuestra, con número de línea. No modifiques ningún archivo. Quiero comprobar personalmente si el correo electrónico del comentarista aparece en la respuesta pública.
```

**Qué salió:** la fila de C-03, `routes.ts:150-155` y `:166-174`, la migración `:16` y los comandos para comprobarlo.

## Prompt 6 — líneas 150-174 de routes.ts

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Abre backend/start/routes.ts y muéstrame las líneas 150 a 174 directamente del archivo, sin interpretarlas ni modificarlas.
```

**Qué salió:** las líneas tal cual, sin interpretación.

## Prompt 7 — DELETE /comentarios/:id

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Muéstrame directamente del archivo backend/start/routes.ts las líneas correspondientes a DELETE /comentarios/:id, con sus números de línea. No modifiques nada.
```

**Qué salió:** `routes.ts:336-361`.

## Prompt 8 — líneas 20-30 de routes.ts

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Muéstrame directamente del archivo backend/start/routes.ts las líneas 20 a 30, con sus números de línea. No modifiques nada ni ejecutes peticiones.
```

**Qué salió:** las líneas tal cual.

## Prompt 9 — SECRETO_SESION en el historial

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Ejecuta git log -S SECRETO_SESION --oneline -- backend/start/routes.ts y muéstrame únicamente los hashes y mensajes de los commits encontrados. No muestres el valor de la clave ni modifiques archivos.
```

**Qué salió:** `e0e0943 initial import`.

## Prompt 10 — cierre de la entrega

**Modelo:** Claude Opus 5.5
**Herramienta:** Claude Code

```
Finaliza la entrega del Módulo 6 sin realizar nuevas investigaciones ni modificar código funcional.

1. Actualiza docs/auditoria/comentarios.md con estas tres respuestas de la Parte B:

- El agente identificó 21 hallazgos. Yo comprobé personalmente 3, abriendo los archivos y revisando la evidencia: C-02, C-03 y C-05. En C-05 también comprobé que SECRETO_SESION aparece en el historial de Git mediante el commit e0e0943. No ejecuté personalmente pruebas HTTP.
- El hallazgo cuya prioridad me resultó difícil decidir fue C-04, porque describe una posible condición de carrera que no se reprodujo en las 40 peticiones concurrentes. No sé si priorizar el riesgo potencial por encima de defectos reproducibles.
- No pude verificar el comportamiento visual de los comentarios en el navegador, especialmente los comentarios que desaparecen y las diferencias en los contadores. La evidencia disponible procede del análisis del código, no de una comprobación personal de la interfaz.

2. En prompts.md registra literalmente todos los prompts utilizados, incluyendo este mensaje de cierre y el intento inicial que falló por sesión caducada. Conserva los datos reales de modelo y herramienta; no inventes versiones.

3. No alteres los 21 hallazgos existentes ni completes campos sin evidencia.

4. Verifica que únicamente estén modificados docs/auditoria/comentarios.md y prompts.md.

5. Muéstrame un resumen final de los cambios. No hagas commit ni push.
```

**Qué salió:** Parte B rellenada con tus tres respuestas y este registro completado. Sin commit ni push.
