# Trusted Origins — vía rápida

Un paquete cae en la vía rápida si cumple **cualquiera** de los criterios
de su ecosistema. Fuera de eso, aplica la ruta obligatoria de
`vetting-sources.md` sin excepción.

## NuGet

| Criterio | Ejemplo |
|---|---|
| Namespace `Microsoft.*` o `System.*` | `Microsoft.Data.SqlClient`, `System.Text.Json` |
| Ya listado como default en `dotnet-clean-architecture/references/project-structure.md` | `Dapper`, `AutoMapper`, `FluentValidation.DependencyInjectionExtensions`, `BCrypt.Net-Next`, `System.IdentityModel.Tokens.Jwt`, `Swashbuckle.AspNetCore` |
| Publicado por una cuenta con el badge "Verified Account" de NuGet.org, perteneciente a una organización reconocida (ej. una fundación .NET Foundation) | — evalúa caso por caso, el badge por sí solo no exime del chequeo si el publicador no es reconocible |

## npm

| Criterio | Ejemplo |
|---|---|
| Scope de una organización oficial del ecosistema | `@angular/*`, `@angular/cdk`, `@nestjs/*` |
| Ya listado como default en `angular-feature-architecture`/`angular-design-system` | `primeng`, `boxicons`, `chart.js`, `ng2-charts` |
| Paquete con badge de **provenance** (procedencia verificada vía npm/GitHub Actions) publicado por una organización reconocida | — el badge reduce el riesgo pero no reemplaza el criterio de reconocibilidad del publicador |

## Chequeo de typosquat (aplica incluso en vía rápida)

Antes de instalar cualquier paquete de la vía rápida, confirma que el
nombre coincide **exactamente**, carácter por carácter, con el paquete
real conocido — no una variante con guion/número/mayúscula cambiada
(`reqeust` en vez de `request`, `angualr-forms` en vez de
`@angular/forms`, `Dapper.Cotrib` en vez de `Dapper.Contrib`). Esta es la
defensa mínima contra el vector de ataque más común: registrar un paquete
malicioso con un nombre casi idéntico a uno confiable. Si hay la más
mínima duda sobre el nombre, trata el paquete como si no estuviera en la
vía rápida y aplica la ruta obligatoria completa.

## Actualizar esta whitelist

Cuando el usuario adopte un nuevo paquete de forma recurrente en un
proyecto (ya vetado una vez sin hallazgos, o con un riesgo aceptado
explícitamente), no hace falta agregarlo aquí — eso ya lo cubre la lista
append-only `### Paquetes ya vetados` de `.claude/skill-decisions.md` (ver
`SKILL.md`), que es por proyecto. Esta tabla es solo el whitelist genérico
de orígenes, reutilizable entre proyectos.
