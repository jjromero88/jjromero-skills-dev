# Plantilla: `.claude/rules/codigo-escalable.md`

Se copia al proyecto reemplazando los `{marcadores}`. Las líneas de capas
que el proyecto no tiene se eliminan. Todo lo que va dentro del bloque es
el contenido literal del archivo de regla.

````markdown
---
paths:
  - "{carpeta de fuentes}/**"
---

# Código escalable: lo compartido se centraliza, no se duplica (OBLIGATORIO)

Aplica siempre que se escriba, complete o modifique código, en cualquier módulo y capa del stack
({capas del proyecto, ej. frontend Angular, backend .NET, base de datos SQL Server y worker}), aunque el
pedido no lo mencione.

- **Antes de escribir, preguntarse quién más lo va a necesitar.** Mensajes, constantes, tipos, mapeos,
  validaciones, armado de requests, llamadas HTTP, consultas y bloques SQL: si otra pantalla, servicio o
  procedimiento ya lo tiene, o es previsible que lo necesite (editar después de crear, otro módulo que
  consume el mismo servicio, otro SP que repite el mismo cálculo), se escribe una sola vez en el lugar
  compartido y los consumidores lo referencian. No se espera a la segunda copia para centralizar.
- **Basta que algo sea o pueda ser usado por más de un componente, módulo o procedimiento** para que vaya
  al nivel compartido, en cualquier capa:
  - Frontend: función, componente, pipe, directiva, servicio, modelo, estilo o mecanismo. Ejemplo: un
    mapeo de códigos a etiquetas resuelto con un `if` en una pantalla es una utilidad compartida.
  - Backend: helper, extensión, mensaje, regla de validación, resolver, DTO base o servicio. Ejemplo: el
    desencriptado y armado del JSON de una lista de IDs repetido en dos Services es un helper compartido.
  - Base de datos: un bloque que hace lo mismo en varios procedimientos es una function
    (`fn_svf_`/`fn_tvf_`) o un table type (`TT_`). Ejemplo: el cálculo de los ancestros de un nodo
    jerárquico repetido en dos SPs es una `fn_tvf_`.
- **Lo que pertenece a un servicio o mecanismo transversal vive en su bloque, no en cada consumidor.** Si el
  texto, el tipo o la regla describe al servicio (el servicio no responde, el recurso no existe, el ID no
  es válido) y no a la pantalla o al caso de uso, va en el servicio o en su capa compartida. El consumidor
  solo muestra o usa lo que el bloque le devuelve.
- **Antes de crear algo, se busca** en las carpetas compartidas de la lista de abajo, ejecutando el
  inventario (sección siguiente). No se da por hecho que algo no existe sin haberlo buscado.
- **Dónde va lo compartido** (rutas reales del repo; las marcadas *(se crea al primer uso)* aún no existen
  y su ubicación ya está decidida):
  - **Frontend** (`{ruta del frontend}`)
    - {tipo de pieza}: `{ruta real}`
  - **Backend** (`{ruta del backend}`)
    - {tipo de pieza}: `{ruta real}`
  - **Base de datos** (`{ruta de la BD}`)
    - {tipo de pieza}: `{ruta real}`
- **Sin sobre-abstraer.** Se centraliza lo que se comparte o claramente se va a compartir. Lo propio de una
  pantalla, servicio o caso de uso se queda ahí. No se crea una abstracción para un solo consumidor sin
  reuso previsible.
- **La duplicación que ya existe fuera del pedido se detecta y se informa con una propuesta, pero no se
  refactoriza sin pedido**: lo adyacente se ofrece, no se hace.
- **Centralizar respeta los patrones existentes**: se hace con la estructura y las convenciones del repo y
  de las skills de cada capa, y si obligara a romper una, se advierte antes.
- Al reportar el trabajo, decir qué se centralizó y para quién queda disponible.

## Inventario antes de crear (ejecutar, no suponer)

Desde la raíz del repo. La salida es la lista real de lo compartido: si lo que se va a escribir ya está
ahí, se reutiliza; si no está, recién entonces se crea en la ruta que corresponde de la lista de arriba.

```bash
{comandos find de references/inventario-por-stack.md, con las rutas reales y probados}
```

Para saber si un texto, nombre o bloque concreto ya existe (y cuántas veces se repite), buscarlo literal
en la capa que corresponda:

```bash
{comandos grep -F de references/inventario-por-stack.md, uno por capa}
```

Más de un resultado fuera del lugar compartido es duplicación: se centraliza si está dentro del pedido, o
se informa con una propuesta si está fuera.
````
