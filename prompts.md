# Prompts

Todos los prompts lanzados en la sesión, en orden, tal cual.

## Prompt 1

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code (extensión de VS Code)

```
resumeme los ultimos 5 commit
```

**Qué salió:** el repo solo tenía 3 commits; los resumió. No tiene relación con la auditoría.

## Prompt 2

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code (extensión de VS Code)

```
Ahí va todo lo que produzcas. El directorio docs/auditoria/ ya existe: ahí va el archivo de la tarea, y ahí se queda, porque una auditoría es un artefacto versionado y no una conversación.

📋 La tarea
Son unos 45 minutos y hay que ponerles un reloj. Léete entero el encuadre antes de empezar, porque el entregable no es el que parece.

El entregable no es la lista ordenada. Son las tres líneas de la parte B, y esas se escriben igual de bien con la lista a medias.

De hecho, si terminas con una lista ordenada y estás seguro del orden, vuelve a mirarla: casi nadie lo está en la primera pasada, y el motivo no es la herramienta.

El reloj tampoco es una crueldad de diseño. En una auditoría real tampoco te dan tiempo infinito: te dan la tarde que hay antes de la reunión. Lo que sale en 45 minutos es exactamente la parte que depende de tener criterio, y no la que depende de tener una herramienta mejor.

⚠ Ve guardando cada prompt tal cual lo lanzas, desde el primero. Se entregan junto con la tabla, y no valen reconstruidos: el prompt que arreglas mentalmente diez minutos después no es el que lanzaste, y es justo la diferencia que interesa mirar.

🅰 Parte A: la auditoría
Con un agente, produce la auditoría priorizada de la feature de comentarios y déjala escrita en docs/auditoria/comentarios.md, no en el chat.

Una feature es una funcionalidad del blog de punta a punta: sus rutas en la API, su tabla en la base de datos y lo que la web enseña de ella. Este blog tiene cinco (posts, comentarios, autores, categorías y etiquetas), y la tuya es comentarios: crear, editar, borrar y listar comentarios, en esas tres capas. Es la misma que audita el mentor en el directo.

Solo esa feature. No el sistema entero, aunque quepa y aunque el agente se ofrezca.

Y el orden, con el mismo criterio que usa el mentor: primero la precedencia (lo que abarata o desbloquea a otro hallazgo va antes), y dentro de cada nivel, el daño dividido por el radio. Si un hallazgo tiene el daño más alto y la precedencia más baja a la vez, se parte en dos filas, una contención que corta el daño hoy y el arreglo de fondo que espera su turno. Y las propuestas que nadie ha medido todavía van fuera de la tabla, con lo que habría que medir escrito al lado.

El formato no es negociable: una fila por hallazgo, con estas seis casillas.

Qué es. Una frase. Si necesitas tres, es que hay dos hallazgos.

Dónde está, con la línea que lo demuestra. Sin esta casilla, la fila no se escribe.

Qué se rompe, y quién lo nota. El efecto concreto y a quién le llega. "Nadie todavía" es una respuesta válida y buena.

Cuántos sitios hay que tocar. Contados, no estimados.

¿Se deshace con un revert? Sí o no, y si es que no, por qué no.

¿Bloquea o desbloquea a otro hallazgo? Nombrando al otro. "A ninguno" es la respuesta más frecuente.

⚠ Cuando suene el reloj, para. Aunque esté a medias. Aunque falten filas, aunque haya casillas en blanco, aunque justo estuvieras a punto de comprobar una cosa.

Una fila con tres casillas rellenas y tres en blanco es información: dice exactamente hasta dónde llegaste. Una fila completada de memoria diez minutos después es ruido con formato, y encima es indistinguible de la buena.

🅱 Parte B: las tres líneas
Debajo de la tabla, en el mismo archivo. Esta parte no se puede fallar, y es la que hay que traer sí o sí.

Cuántos hallazgos devolvió el agente, y cuántos comprobaste tú abriendo el archivo. Los dos números, tal cual salieron. No los redondees ni los expliques.

El hallazgo del que no supiste decidir la prioridad, y en una frase, por qué. Lo que interesa es qué dos cosas se contradecían: lo grave contra lo que bloquea, lo urgente contra lo caro, lo que te pedían contra lo que veías.

Algo que el agente afirmó y no pudiste verificar. Ojo al matiz: no es lo que dijo mal. Es lo que no pudiste comprobar ni a favor ni en contra, y te tocó dejar en el aire.

⚠ Ninguna de las tres tiene respuesta correcta. La segunda y la tercera son mejores cuanto más incómodas: un "no supe decidir entre estos dos" honesto vale más que una lista ordenada con seguridad fingida.

Cómo saber que la has hecho bien
Las tres líneas están escritas y son concretas. Si la tercera dice "nada, todo era verificable", vuelve a mirar: es la respuesta que da alguien que comprobó leyendo el hallazgo en vez de abriendo el archivo.

El número de comprobados es menor que el de devueltos. Casi siempre lo es, y eso no es un fallo tuyo: es el dato.

Hay al menos una casilla en blanco. Significa que respetaste el reloj y que no rellenaste de memoria.

La tabla cabe en una pantalla. Si no cabe, te saliste de comentarios y estás auditando el sistema entero otra vez.
```

**Qué salió:** a la primera. Una auditoría de 12 filas con casillas en blanco en las últimas, y las tres líneas de la parte B.

## Prompt 3

**Modelo:** Sonnet 5.5
**Herramienta:** Claude Code (extensión de VS Code)

```
📤 Cómo se entrega
Un pull request desde tu fork, con dos cosas dentro y ni una más:

Tu archivo de auditoría en docs/auditoria/, con la tabla y las tres líneas.

prompts.md, en la raíz del proyecto. Ya está ahí con la plantilla puesta.

git add docs/auditoria prompts.md
git commit -m "auditoria: comentarios + prompts"
git push -u origin auditoria-<tus-iniciales>
Con la rama empujada, GitHub te ofrece arriba el botón para abrir el pull request. Va contra el repositorio del curso, no contra tu fork.

🧠 prompts.md no es papeleo, y es la mitad de lo que se revisa. Lo que se mira no es solo lo que te salió, es cómo lo pediste: un resultado flojo con un prompt bueno y un resultado flojo con un prompt vago necesitan respuestas distintas, y sin ese archivo no se distinguen. Pega los prompts tal cual los lanzaste, con su modelo y su herramienta, e incluye también los que no funcionaron, que suelen ser los más útiles de leer.
```

**Qué salió:** detectó que había dejado los prompts en `docs/auditoria/prompts.md` y con el prompt 2 resumido; los movió aquí, completos.
