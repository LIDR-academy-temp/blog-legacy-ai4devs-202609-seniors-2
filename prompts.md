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

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
## Contexto
El blog tiene cinco features: posts, comentarios, autores, categorías y etiquetas. Una feature abarca sus rutas en la API, su tabla en la base de datos y lo que la web muestra de ella. La tuya es **comentarios**: crear, editar, borrar y listar, en esas tres capas.

## Tarea
Audita la feature de comentarios y escribe la auditoría priorizada en `docs/auditoria/comentarios.md`.

## Formato (no negociable)
Una fila por hallazgo, con seis casillas. Cada una se responde con un hecho, no con una nota del 1 al 5.

1. **Qué es**: una frase. Si necesitas tres, son dos hallazgos.
2. **Dónde está**: archivo y línea que lo demuestra. Sin esto, la fila no se escribe.
3. **Qué se rompe y quién lo nota**: el efecto concreto y a quién le llega (un lector, un socio, el equipo, o "nadie todavía").
4. **Cuántos sitios hay que tocar**: contados y listados. Si son tres, se escriben los tres.
5. **¿Se deshace con un revert?**: sí o no; si no, por qué (toca datos guardados o algo que otros ya leen). Si no lo sabes, pon "no": entra con una persona delante.
6. **¿Bloquea o desbloquea a otro hallazgo?**: nombra cuál. "A ninguno" es válido y lo más común.

## Casillas que no puedes demostrar
Rellena solo lo que puedes demostrar con el código. Lo que no, déjalo como `VACÍO`, a la vista. No lo supongas.
Esto aplica sobre todo al daño: saber quién lo nota exige saber quién usa el sistema, y eso no está en el código. Una casilla vacía es mejor que una inventada.

## Orden
1. Primero, precedencia: si arreglar A abarata o desbloquea B, A va antes.
2. Dentro de cada nivel, daño dividido por radio.
3. Si un hallazgo tiene el daño más alto y la precedencia más baja, se parte en dos filas: una contención que corta el daño hoy y el arreglo de fondo, que espera su turno.
4. Las filas que no se pueden ordenar porque tienen el daño o el radio en `VACÍO` van al final de la tabla, sin ordenar. Las revisa una persona.
5. Las propuestas que nadie ha medido van fuera de la tabla, con lo que habría que medir al lado.

## Restricciones
- Solo la feature de comentarios. No el sistema entero, aunque quepa.
- El resultado va en el archivo, no en el chat.

## Definition of done
- `docs/auditoria/comentarios.md` existe y contiene la tabla.
- Cada fila tiene las seis casillas, rellenas con un hecho o marcadas `VACÍO`.
- Ninguna fila sin archivo y línea.
- El radio lista los sitios, no solo los cuenta.
- La tabla sigue el orden indicado, con las filas sin ordenar al final.
- Las propuestas sin medir están fuera de la tabla, con su medida pendiente.
```

**Qué salió:** escribió la auditoría de 18 filas; el afinado de orden, daños y reverts vino en los prompts siguientes.

## Prompt 2

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
donde puedo ver el valor del dano de cada hallazgo ?
```

**Qué salió:** funcionó a la primera: resumió daño, radio y cociente de todas las filas.

## Prompt 3

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
porque F01 N1 es la primera?
```

**Qué salió:** funcionó a la primera: la precedencia, porque las pruebas desbloquean todo lo demás.

## Prompt 4

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
justifica porque F02 N2 va de segunda
```

**Qué salió:** la justificación dejó ver que el orden era débil: el daño dependía de una carrera sin demostrar.

## Prompt 5

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
Que tan probable es que pase esto ?
```

**Qué salió:** no dio una cifra: dijo que no lo sabe y propuso medirlo.

## Prompt 6

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
justifica el revert del punto F03 N2
```

**Qué salió:** defendió un «No» con una razón floja; tuve que insistir en el siguiente.

## Prompt 7

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
pero el cambio es un cambio de codigo no? si es un cambio de codigo, se pued revertir facilmente, no?
```

**Qué salió:** tuve que insistir: admitió que el «No» estaba mal y lo corrigió.

## Prompt 8

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
revisa los otros hallazgos y aplica esta misma logica para determinar si se puede revertir
```

**Qué salió:** funcionó: cambió tres filas más a «Sí» y dejó cuatro en «No» con su razón.

## Prompt 9

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
debatamos esto "Quien conozca el correo de un comentarista puede borrar "yo veo que tambien hay una logica que verifica si es moderador "const esModerador = clave === SECRETO_SESION"
```

**Qué salió:** tuve que señalarlo yo: la contención de F03 dejaba abierta la vía de la clave de moderación y no estaba escrito.

## Prompt 10

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
ok, dejemoslo asi. Esta contencion de que hallazgo principal es?
```

**Qué salió:** funcionó a la primera: es la contención de F17, el arreglo de fondo del mismo hallazgo.

## Prompt 11

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
porque no se pueden definir los sitios a tocar de F17 ? para determinar su orden
```

**Qué salió:** tuve que insistir: había dejado el radio en `VACÍO` sin ofrecer contar los sitios por mecanismo.

## Prompt 12

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
explica mejor la opcion 2
```

**Qué salió:** funcionó a la primera.

## Prompt 13

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
esta opcion 2 es la mejor alternativa, verificando la sesion del usuario
```

**Qué salió:** funcionó: leyó el starter kit, contó 5 sitios para F17 y reordenó la tabla.

## Prompt 14

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
antes teniamos 3 items sin ordernar, y ya solo 1, porque?
```

**Qué salió:** funcionó a la primera.

## Prompt 15

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
Ok. Explica porque no hay daño en F18
```

**Qué salió:** funcionó a la primera: el daño depende de si hay un panel de moderación fuera del repo.

## Prompt 16

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code

```
opino que es mas prioritario F03 que F02
```

**Qué salió:** tuve que imponer el criterio: por cociente salía al revés; lo aplicó como ajuste manual y lo dejó escrito.