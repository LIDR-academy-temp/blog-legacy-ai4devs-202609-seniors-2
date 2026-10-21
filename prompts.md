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

**Modelo:** sonnet 5.5 xHigh
**Herramienta:** Claude Code

```
Con un agente, produce la auditoría priorizada de la feature de comentarios y déjala escrita en docs/auditoria/comentarios.md; **solo esa feature** y no el sistema entero.
El formato no es negociable: una fila por hallazgo, con estas seis casillas.
1. Qué es. Una frase. Si necesitas tres, es que hay dos hallazgos.
2. Dónde está, con la línea que lo demuestra. Sin esta casilla, la fila no se escribe.
3. Qué se rompe, y quién lo nota. El efecto concreto y a quién le llega. "Nadie todavía" es una respuesta válida y buena.
4. Cuántos sitios hay que tocar. Contados, no estimados.
5. ¿Se deshace con un revert? Sí o no, y si es que no, por qué no.
6. ¿Bloquea o desbloquea a otro hallazgo? Nombrando al otro. "A ninguno" es la respuesta más frecuente.
```

**Qué salió:** (opcional, una línea) funcionó a la primera / tuve que insistir / me inventó una ruta que no existe.
Ya lancé el agente. Está leyendo routes.ts, las migraciones, el seeder, el test y el frontend, y escribirá la auditoría en docs/auditoria/comentarios.md. Todavía no sé qué ha encontrado.

Le encargué esto:
- Alcance: solo la feature de comentarios. Lo que vea fuera de ella no entra en el archivo y me lo cuenta aparte.
- Formato: una tabla con las seis casillas que pediste. Las filas van ordenadas por prioridad, y cada una empieza con un identificador (H1, H2…) para poder nombrar otros hallazgos en la sexta casilla.
- Evidencia: sin línea de código citada y comprobada, la fila no se escribe.
- Solo diagnóstico: no modifica código, no crea ramas ni hace commits. Puede ejecutar npm test y hacer peticiones GET a la API, pero ningún POST contra la base de datos real.

Cuando termine, te resumo el resultado y lo que no haya podido verificar.

La auditoría está escrita en docs/auditoria/comentarios.md. Tiene 27 hallazgos (H1 a H27), una fila cada uno, ordenados por prioridad.

Qué comprobé yo
- Todas las filas tienen las seis casillas. Mi primer recuento dio seis filas mal, pero eran las que llevan \| escapado dentro de una celda. Al tratarlo como texto, las 27 cuadran.
- Las citas de routes.ts:25, :153 y :171 coinciden con el código real.
- El agente dice que comprobó las 43 citas con un script. Yo solo verifiqué esas tres.
- Los resultados de las pruebas de escritura los doy por buenos según el informe del agente. Las hizo en una copia desechable, y el repo solo muestra el archivo nuevo sin trackear, además de docs/make_test.md.

Los tres primeros
1. H1: el detalle del post devuelve el autor_email de cada comentario (routes.ts:153 y :171). Rompe la regla 6, y hoy salen 4 correos.
2. H2: DELETE y PATCH autorizan con el correo que manda el cliente, y ese correo es público por H1. En la copia, el agente borró un comentario usando el correo leído del GET público.
3. H3: la clave de moderación dev-secret-cambiar está en el código (routes.ts:25) y en el historial de git. Con ella se borra cualquier comentario.

Después vienen el estado libre sin restricción (H4), los recuentos (H5 a H7), la validación, los huecos de tests, las desviaciones de arquitectura y, al final, dos latentes (H26 y H27).

Lo que no está confirmado
- H27 (carrera en el select ... order by id desc limit 1) no se reprodujo con 1.500 POST concurrentes. Está marcada como no confirmada.
- Que Aprobado y ok signifiquen «aprobado» es una inferencia a partir de la migración y la semilla. Habría que confirmarlo con quien modera.
- H7, H9 y H11 se dedujeron del código de Comentarios.jsx, porque el agente no abrió el navegador.
- Con un solo test, routes.ts queda en 32% de cobertura. Los handlers de DELETE, PATCH y el detalle no se ejecutan nunca.
- Una prueba de mutación cambió tres comportamientos a la vez y el único test siguió verde.

Fuera de alcance
No están en el archivo. El agente las vio y me las pasó:
- GET /posts/:slug devuelve el correo del autor del post (routes.ts:166-168).
- GET /autores/:id lista también los borradores (routes.ts:284-285).
- DetallePost.jsx:92 usa dangerouslySetInnerHTML con el cuerpo del post sin sanear.
- posts.slug no es único.

Rama y commit
El agente avisó de que la rama actual es auditoria-cedl, no s6/start. Lo confirmé, y apunta al mismo commit (ad34f0e). No he hecho commit. Si quieres, lo hago en una rama docs/ con un commit docs: que añada solo ese archivo.

