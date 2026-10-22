# Prompts

---

## Prompt 1

**Modelo:** Opus 5.5
**Herramienta:** Claude Code

```
Resuma los últimos commits, y que busque en qué archivos aparece una palabra cualquiera del dominio. Si no puede ejecutar git o no puede buscar por el proyecto, resuélvelo antes: sin eso no se audita nada.
```

**Qué salió:** Funcionó a la primera.

---

## Prompt 2

**Modelo:** Opus 5.5
**Herramienta:** Claude Code

```
Con un agente, produce la auditoría priorizada de la feature de comentarios y déjala escrita en docs/auditoria/comentarios.md.
El formato no es negociable: una fila por hallazgo, con estas seis casillas.

**Contexto y Alcance (Feature a trabajar):**
- Una *feature* comprende el stack completo de punta a punta: rutas de la API, tabla en la base de datos y la interfaz web.
- El blog cuenta con cinco features: posts, comentarios, autores, categorías y etiquetas.
- **Tu enfoque exclusivo:** Trabaja **únicamente en la feature de comentarios** (crear, editar, borrar y listar) a través de sus tres capas. Es la misma evaluada por el mentor en el directo.
- **Restricción estricta:** Limítate exclusivamente a esta feature. No abordes el sistema entero, aunque sea técnicamente viable o aunque te ofrezcas a hacerlo.

**Criterios de Orden y Priorización:**
1. **Precedencia:** Lo que abarque o desbloquee a otro hallazgo va primero.
2. **Impacto:** Dentro de cada nivel, la prioridad se calcula como *daño dividido por el radio*.
3. **Casos especiales (Alto daño / Baja precedencia):** Si un hallazgo combina el daño más alto con la precedencia más baja, **pártelo obligatoriamente en dos filas**:
   - Una **contención** que detenga el daño de inmediato.
   - El **arreglo de fondo** que esperará su turno según corresponda.
4. **Propuestas sin métricas:** Aquellas propuestas que nadie haya medido todavía deben colocarse **fuera de la tabla principal**, acompañadas por escrito de la métrica que sería necesario evaluar.

Qué es. Una frase. Si necesitas tres, es que hay dos hallazgos.

Dónde está, con la línea que lo demuestra. Sin esta casilla, la fila no se escribe.

Qué se rompe, y quién lo nota. El efecto concreto y a quién le llega. "Nadie todavía" es una respuesta válida y buena.

Cuántos sitios hay que tocar. Contados, no estimados.

¿Se deshace con un revert? Sí o no, y si es que no, por qué no.

¿Bloquea o desbloquea a otro hallazgo? Nombrando al otro. "A ninguno" es la respuesta más frecuente.
```

**Qué salió:** Funcionó a la primera, hizo el analisis correspondiente y lo registro tal y como se le indicó en el prompt.