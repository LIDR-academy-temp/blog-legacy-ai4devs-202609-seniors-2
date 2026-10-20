# Prompts

## Prompt 1 — Exploración (solo comentarios)

**Modelo:** Opus 5.5
**Herramienta:** Claude Code v2.1.296

```
Estoy en el repo Blog Legacy, rama auditoria-PC.

TAREA: explorar SOLO la feature de comentarios (crear, editar, borrar, listar) en API + DB + web. NO audites posts, autores, categorías ni etiquetas.

Hacé:
1. Leé las reglas de negocio del README.md (las 6).
2. Listá las rutas de comentarios en backend/start/routes.ts y leé esas secciones.
3. Leé la migración de comentarios y el test backend/tests/functional/comentarios.spec.ts.
4. Leé frontend/src/componentes/Comentarios.jsx.
5. Decime en una lista corta: qué archivos tocan comentarios y qué hace cada ruta.

NO escribas hallazgos todavía. NO modifiques archivos. Mostrámelo.
```

**Qué salió:** Listó rutas, migración (estado texto libre, sin FK), un solo test del POST feliz, frontend solo crea/lista. ~42 s.

---

## Prompt 2 — Auditoría priorizada

**Modelo:** Opus 5.5
**Herramienta:** Claude Code v2.1.296

```
Seguimos en Blog Legacy, rama auditoria-PC. Feature: SOLO comentarios.

TAREA: producí la auditoría priorizada en docs/auditoria/comentarios.md.

Reglas:
- Solo hechos y riesgos. Opiniones/propuestas sin medir van FUERA de la tabla, con qué habría que medir al lado.
- Orden: (1) precedencia, (2) dentro de cada nivel daño÷radio. Si daño alto y precedencia baja chocan, PARTÍ en contención + arreglo (dos filas).
- Cada hallazgo DEBE tener archivo:línea. Sin eso, no es hallazgo.
- Contrastá contra las reglas de negocio del README (comentario pendiente oculto, recuento solo aprobados, no exponer emails).

Formato de tabla (6 columnas, una fila por hallazgo):

| Qué es | Dónde (archivo:línea) | Qué se rompe / quién lo nota | Cuántos sitios hay que tocar | ¿Se deshace con un revert? | ¿Bloquea o desbloquea a otro? |
|---|---|---|---|---|---|

SOLO escribí docs/auditoria/comentarios.md. NO arregles código. NO hagas commit. NO abras PR.
```

**Qué salió:** 17 filas priorizadas (pares contención/arreglo #1/#12 y #7/#8), opiniones fuera de la tabla, Parte B propuesta. ~4 min. La verificación de hallazgos (abrir archivo:línea) la hice yo a mano después, no con otro prompt.
