---
name: scalable-code
description: Regla de código escalable para todo el stack (Angular, .NET, SQL Server/PostgreSQL, workers) — lo que usa o puede usar más de un consumidor se escribe una sola vez en su lugar compartido, sin sobre-abstraer. La primera vez en un proyecto instala la regla `.claude/rules/codigo-escalable.md` con las rutas reales del repo y un inventario determinista (comandos `find`/`grep` probados) más un resumen en `CLAUDE.md`. Úsala siempre que se escriba, complete o modifique código, junto con `angular-feature-architecture`, `dotnet-clean-architecture` y `sql-database-patterns`.
---

# Scalable Code

> ⚠️ Skill personal, transversal a frontend, backend y base de datos.
> Complementa a las skills de cada capa: ellas dicen **cómo** se escribe
> cada pieza; esta dice **dónde** vive lo compartido y **cuándo** se
> centraliza en vez de duplicar.

## Instalación en el proyecto (primera vez)

Copiar la carpeta de la skill a `.claude/skills/` no instala nada por sí
solo: la regla se crea la primera vez que la skill se usa en el proyecto.

1. Si ya existe `.claude/rules/codigo-escalable.md`, la regla está
   instalada: se aplica tal cual y no se rehace (sus rutas son decisiones
   del proyecto). Ir a "Aplicación".
2. Revisar la estructura del repo y ubicar dónde vive hoy lo compartido de
   cada capa (componentes, servicios, utilidades, tipos, constantes,
   mensajes, reglas de validación, estilos, functions, table types). Usar
   como punto de partida la tabla "Ubicaciones por defecto" y confirmarla
   contra el repo real. **No inventar carpetas**: si para algún tipo de
   pieza no hay un lugar claro ni lo define la skill de su capa,
   preguntar al usuario antes de seguir.
3. Crear `.claude/rules/codigo-escalable.md` desde la plantilla
   `references/regla-codigo-escalable.md`:
   - `{carpeta de fuentes}` → la carpeta del código fuente, relativa a la
     carpeta que contiene `.claude/` (ahí se resuelve `paths`).
   - Completar "Dónde va lo compartido" con las rutas reales del paso 2;
     marcar *(se crea al primer uso)* las que aún no existen pero ya están
     decididas.
   - Armar el inventario con `references/inventario-por-stack.md` y
     **ejecutar cada comando** antes de dejarlo en la regla: debe listar
     los archivos compartidos que ya existen. Un patrón que no los lista
     se corrige (ej. `*/Application/Common/*` no coincide con
     `{Prefijo}.Application/Common/`; usar `*.Application/Common/*`).
   - Quitar las capas que el proyecto no tiene.
4. Agregar a `CLAUDE.md` un resumen de 3 líneas con enlace a la regla; si
   `CLAUDE.md` no existe, crearlo solo con ese resumen:

   ```markdown
   ## Reglas del proyecto

   - Código escalable ([`.claude/rules/codigo-escalable.md`](.claude/rules/codigo-escalable.md)): en todo el stack, lo que use o pueda usar más de un consumidor se escribe una sola vez en su lugar compartido.
   - Antes de crear algo se ejecuta el inventario de la regla y se reutiliza lo que ya existe; sin abstracciones para un solo consumidor.
   - La duplicación fuera del pedido se informa con una propuesta, no se refactoriza sin pedido.
   ```

5. Mostrar al usuario cómo quedaron ambos archivos. No tocar nada más.

Cómo carga Claude Code la regla (doc oficial, *Path-specific rules* en
https://code.claude.com/docs/en/memory.md): los `.md` de `.claude/rules/`
se descubren recursivamente; una regla con `paths` se carga cuando Claude
lee o edita un archivo que coincide (no al arrancar ni al buscar con
Grep/Glob); los patrones son relativos a la carpeta que contiene `.claude/`.

## Ubicaciones por defecto

Punto de partida para el paso 2 cuando el proyecto sigue las skills de
cada capa. Siempre se confirman contra el repo real.

| Capa | Pieza | Ubicación |
|---|---|---|
| Angular | Transversal a la app (HTTP, auth, layout, tema, menú) | `core/{services,models,guards,interceptors,layout}/` |
| Angular | Componentes de UI reutilizables (ej. `tabla-listado`, `ubigeo-cascada`) | `shared/ui/` |
| Angular | Servicios / modelos usados por 2+ features | `shared/services/`, `shared/models/` |
| Angular | Funciones puras, directivas, datos estáticos | `shared/helpers/`, `shared/directives/`, `shared/data/` |
| Angular | Estilos, tokens y clases compartidas | `styles.css` |
| .NET | Respuesta estándar de la API | `{Prefijo}.Common/Response/` |
| .NET | `SpResult` + constantes y mensajes compartidos entre servicios | `{Prefijo}.Application/Common/` |
| .NET | Helpers de Application (ej. `SpJson`: JSON hacia y desde los SPs) | `{Prefijo}.Application/Helpers/` |
| .NET | Extensiones de una interfaz, junto a ella (ej. `IdEncryptionExtensions`: `EncryptNullable`, `TryDecryptNullable`, `TryDecryptArray`) | `{Prefijo}.Application/Interfaces/Security/` |
| .NET | DTOs base, interfaces transversales | `{Prefijo}.Application/DTOs/Base/`, `{Prefijo}.Application/Interfaces/Common/` |
| .NET | Reglas de validación reutilizables (extensiones de FluentValidation) | `{Prefijo}.Validator/Rules/` |
| .NET | Resolvers de AutoMapper; cifrado, hashing, JWT | `{Prefijo}.Mapper/Resolvers/`, `{Prefijo}.Security/` |
| SQL | Lógica repetida en varios procedimientos | `functions/fn_svf_{Concepto}.sql`, `functions/fn_tvf_{Concepto}.sql` |
| SQL | Listas que viajan entre procedimientos | `types/TT_{Concepto}.sql` |
| Worker | Reutiliza los proyectos del backend | sin carpeta propia de compartidos |

## Aplicación

Con la regla instalada, su contenido manda. Si todavía no lo está en la
sesión, igual rige lo mismo (resumen de la plantilla):

- Antes de escribir, preguntarse quién más lo va a necesitar; no se espera
  a la segunda copia para centralizar.
- Basta que algo sea o pueda ser usado por más de un consumidor (pantalla,
  servicio, módulo o procedimiento) para que vaya al nivel compartido.
- Lo que describe a un servicio o mecanismo transversal vive en su bloque,
  no en cada consumidor.
- Antes de crear, se ejecuta el inventario; no se afirma que algo no
  existe sin haberlo buscado.
- Sin sobre-abstraer: nada de abstracciones para un solo consumidor sin
  reuso previsible.
- La duplicación fuera del pedido se informa con una propuesta, no se
  refactoriza sin pedido.
- Centralizar respeta la estructura y las convenciones del repo y de las
  skills de cada capa; si obligara a romper una, se advierte antes.
- Al reportar, decir qué se centralizó y para quién queda disponible.

## Referencias

- `references/regla-codigo-escalable.md` — plantilla de la regla (frontmatter `paths`, reglas, ubicaciones, inventario).
- `references/inventario-por-stack.md` — comandos `find`/`grep` por capa, con los patrones que fallan y su corrección.
