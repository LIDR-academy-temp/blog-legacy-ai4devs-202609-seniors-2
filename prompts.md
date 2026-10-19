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

---

## Prompt 1

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
Objectivo: Produce la auditoría priorizada de la feature de comentarios y déjala escrita en "docs/auditoria/comentarios.md", no en el chat. (Se prioriza por precedencia, luego gravedad en relación al radio)

Contexto:
Una feature es una funcionalidad del blog de punta a punta: sus rutas en la API, su tabla en la
base de datos y lo que la web enseña de ella. Este blog tiene cinco (posts, comentarios, autores,
categorías y etiquetas), y la tuya es comentarios: crear, editar, borrar y listar comentarios,
en esas tres capas.

Restricciones:
Solo esa feature. No el sistema entero.

El formato no es negociable: una fila por hallazgo, con estas seis casillas.

1. "Qué es." Una frase. Si necesitas tres, es que hay dos hallazgos.
2. "Dónde está." con la línea que lo demuestra. Sin esta casilla, la fila no se escribe.
3. "Qué se rompe, y quién lo nota." El efecto concreto y a quién le llega. "Nadie todavía"
es una respuesta válida y buena.
4. "Cuántos sitios hay que tocar." Contados, no estimados.
5. "¿Se deshace con un revert?" Sí o no, y si es que no, por qué no.
6. "¿Bloquea o desbloquea a otro hallazgo?" Nombrando al otro. "A ninguno" es la respuesta
más frecuente.

Pregunta si tienes cualquier duda y no hagas commit, ni cambies rama ni PR, hasta que explicitamente se indique.
```

**Qué salió:** Funcionó a la primera. Escribió 19 hallazgos en `docs/auditoria/comentarios.md`, comprobó 3 de ellos contra la API en marcha y no hizo commit.

---

## Prompt 2

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
lista las routas del api
```

**Qué salió:** Sacó las 15 rutas con `node ace list:routes`.

---

## Prompt 3

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
cuantos comentarios hay ahora y en que blogs
```

**Qué salió:** Hay 9 comentarios en 4 posts. Dio el desglose por estado, con `aprobado`, `Aprobado`, `ok` y `pendiente` mezclados.

---

## Prompt 4

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
realmente el criterio de la tabla de revert esta mal se refiere si el arreglo del hallazgo se puede hacer un revert facil de git no el problema
```

**Qué salió:** Había interpretado mal la columna 5. Tuve que corregir el criterio y reescribió la columna.

---

## Prompt 5

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
curl para borrar mi comentario el ultimo
```

**Qué salió:** Vio que no había ningún comentario mío. Me dio los `curl` para crear uno y borrarlo, sin ejecutarlos.

---

## Prompt 6

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
Hay comentarios con respuestas ¿? es posible responder a comentarios desde la web o por la api
```

**Qué salió:** Hay una sola respuesta. Desde la web no se puede responder; por la API sí, con `padre_id`, que no se valida.

---

## Prompt 7

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
se puede editar comentarios desde la web?
```

**Qué salió:** Desde la web no se puede editar. Por la API sí, solo mientras el comentario está `pendiente` y con el correo exacto del autor.

---

## Prompt 8

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
desde la web se puede moderar y aprobar un comentario?
```

**Qué salió:** No se puede ni desde la web ni por la API, solo tocando la base de datos. Propuso añadirlo como hallazgo #20, y no lo añadí.

---

## Prompt 9

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
el hallazgo 5 realmente no impacta a nadie actualmene muevelo al ultimo
```

**Qué salió:** Pasó el hallazgo al #19 y renumeró todas las referencias cruzadas.

---

## Prompt 10

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
vale falta clasificar riesgo y hecho los hallazgo
```

**Qué salió:** Interpretó «Hecho» como un efecto que ya ocurre hoy y «Riesgo» como uno potencial. Etiquetó los 19 hallazgos y propuso 3 casos dudosos.

---

## Prompt 11

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
15 y 18 lo considero como riesgo si
```

**Qué salió:** Reclasificó los dos como Riesgo.

---

## Prompt 12

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
15 lo notan los usuarios que no tienen la opción
```

**Qué salió:** Cambió la casilla 3 y avisó de que eso contradice la etiqueta Riesgo.

---

## Prompt 13

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
realmente 18 deberia estar delante de 9 ya que deslboquea este y otros (mayor radio)
```

**Qué salió:** Subió la arquitectura al #9 y renumeró. Avisó de que también desbloquea al #2, que va delante.

---

## Prompt 14

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
el 2 tambien se resuelve con una contención
```

**Qué salió:** Anotó la contención en el #2 y en el #9, y el orden se quedó como estaba.

---

## Prompt 15

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
Añade una nueva sección Parte B: las tres líneas

1.

Cuántos hallazgos devolvió el agente, y cuántos comprobaste tú abriendo el archivo.

19  vs 9 (revisados por mi)

2.

El hallazgo del que no supiste decidir la prioridad, y en una frase, por qué. Lo que interesa es qué dos cosas se contradecían: lo grave contra lo que bloquea, lo urgente contra lo caro, lo que te pedían contra lo que veías.

Realmente a partir del hallazgo 10 diez no tengo claro el orden , pare que no hay precedencias entre ellos e incluso no se si ponerlso, y luego el hallazgo 1 que es un mandato de las instrucciones pero realmente no es grave.

3.
Algo que el agente afirmó y no pudiste verificar. Ojo al matiz: no es lo que dijo mal. Es lo que no pudiste comprobar ni a favor ni en contra, y te tocó dejar en el aire.

Quizas el hallazgo último es dificil comprobarlo tampoco lo tiene claro el agente
```

**Qué salió:** Añadió la Parte B y avisó de que la línea 2 tenía dos dudas en lugar de una frase.

---

## Prompt 16

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
vale mantemos la línea 2 solo con la referencia al hallazgo 1
```

**Qué salió:** Dejó la línea 2 solo con el hallazgo #1.

---

## Prompt 17

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
en la ultima línea indicar que es dificil probar una test con dos usuarios simultaneos
```

**Qué salió:** Ajustó la línea 3.

---

## Prompt 18

**Modelo:** Opus 5.5 1M Medium
**Herramienta:** Claude Code

```
vale complemta el archivo de @prompts.md  recuerda que he usado opus 5.5 1.0M con esfuerzo medium
```

**Qué salió:** Rellenó este archivo.
