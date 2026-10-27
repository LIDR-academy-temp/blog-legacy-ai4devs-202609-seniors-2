## Prompt 1

**Modelo:** Opus 5.5
**Herramienta:** Claude Code

```
Necesito una auditoría de la feature de comentarios del blog. Solo comentarios: crear, editar, borrar y listar. Eso incluye las rutas de la API, la tabla en la base de datos y lo que la web muestra o manda. Nada de posts, autores, categorías ni etiquetas. Si ves algo raro fuera de comentarios, como mucho lo apuntas en una línea al final y sigues. No me propongas auditar el resto.

No toques el código, esto es solo lectura. Lo único que escribes es docs/auditoria/comentarios.md. El resultado va en ese archivo, no en el chat.

Ve escribiendo el archivo a medida que avanzas, no al final. Primero haz un mapa rápido de dónde vive la feature: las rutas con su handler (archivo:línea), dónde se define la tabla y qué componentes de la web tocan comentarios. Después vas metiendo hallazgos uno a uno y guardando.

Si te digo que pares, paras ahí mismo y dejas el archivo como esté. Si una casilla no la has comprobado abriendo el archivo, se queda vacía. No la rellenes con lo que "seguramente" pasa.

Cada hallazgo es una fila de una tabla con estas columnas:

- Qué es: una frase. Si no cabe en una, son dos hallazgos.
- Dónde está: archivo:línea y la línea copiada tal cual. Sin esto no hay fila.
- Qué se rompe y quién lo nota: el efecto real y a quién le llega. "Nadie todavía" vale.
- Sitios a tocar: contados de verdad, con la lista de archivo:línea. Nada de "unos 3 o 4".
- ¿Se deshace con un revert?: sí o no. Si es no, di por qué (migración, datos ya rotos, etc).
- ¿Bloquea o desbloquea otro?: di cuál (#). Lo normal es "a ninguno".

Para el orden: primero lo que desbloquea o abarata otros hallazgos. Dentro de eso, lo que más daño hace con menos sitios que tocar va arriba. Si algo hace muchísimo daño pero depende de que se arreglen otras cosas antes, pártelo en dos filas: una contención rápida que corte el daño ya, arriba, y el arreglo de fondo en su sitio.

Lo que sean ideas sin evidencia (por ejemplo "el listado igual va lento") no va en la tabla. Ponlo en una sección aparte con qué habría que medir para saber si es real.

Al final del archivo pon si quedó completa o en qué te quedaste.
```
