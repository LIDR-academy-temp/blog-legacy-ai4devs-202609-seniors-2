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

**Modelo:** Opus 5.5
**Herramienta:** Claude Code

```
Necesito realizar la auditoría priorizada de la feature de comentarios de este proyecto.

INSTRUCCIONES Y REGLAS DE COBERTURA:
1. Delimita la auditoría EXCLUSIVAMENTE a la feature de comentarios en sus tres capas: rutas en la API (backend), tabla/modelo en la base de datos y su representación en la web (frontend). Ignora las demás funcionalidades del blog (posts, autores, categorías, etiquetas).
2. RESTRICCIÓN ESTRICTA (SOLO LECTURA): Esto es una auditoría de inspección. NO modifiques ningún archivo de código, configuración, base de datos ni tests del repositorio.
3. REGLA DE CITA OBLIGATORIA: Para cada hallazgo debes indicar el archivo y la LÍNEA EXACTA de código que lo demuestra. Si no puedes citar la línea concreta, el hallazgo NO se escribe.
4. LAS OPINIONES VAN FUERA: Las soluciones propuestas no medidas (por ejemplo, poner una caché) NO entran en la tabla de prioridades. Colócalas al final en un apartado independiente de "Propuestas / Experimentos" indicando qué habría que medir para evaluarlas.

Escribe el resultado directamente en el archivo `docs/auditoria/comentarios.md`.

FORMATO DE LA TABLA (Parte A):
Genera una tabla en Markdown con una fila por hallazgo verificado y exactamente estas 6 columnas:
| Qué es | Dónde está (Archivo y línea) | Qué se rompe y quién lo nota | Cuántos sitios hay que tocar (Radio) | ¿Se deshace con un revert? | ¿Bloquea o desbloquea a otro hallazgo? |

CRITERIO DE PRIORIZACIÓN DE LA TABLA:
- Ordena las filas primero por PRECEDENCIA (lo que desbloquea o abarata a otro hallazgo va antes).
- Dentro de cada nivel de precedencia, ordena por DAÑO DIVIDIDO POR RADIO.
- Si un hallazgo tiene el daño más alto pero la peor precedencia, PÁRTELO en dos filas: una "Contención" (radio mínimo para cortar el daño hoy) y un "Arreglo de fondo" (que espera a su turno de precedencia).

ESTRUCTURA DE LA PARTE B (Debajo de la tabla):
Al pie del archivo `docs/auditoria/comentarios.md`, incluye los encabezados para estas tres líneas finales:
1. Hallazgos devueltos por el agente: [X] | Hallazgos comprobados abriendo el código: [Y]
2. Hallazgo difícil de priorizar: [Indicar qué dos criterios chocaban]
3. Afirmación no verificable: [Indicar qué afirmó el agente que no se pudo comprobar]
```

**Qué salió:** Lista de hallazgos como se pidieron.
