---
name: dotnet-clean-architecture
description: Genera backend .NET Clean Architecture con estas convenciones — 10 proyectos, Dapper + stored procedures (sin EF), DTOs por operación CRUD, ApiResponse unificado, IDs encriptados, IValidatorService centralizado, auditoría desde JWT. Úsala para crear un API .NET o un CRUD de entidad nuevo siguiendo estas convenciones.
---

# .NET Clean Architecture

> ⚠️ Skill personal, no genérica de Clean Architecture. No aplicar a proyectos
> que no sigan este estilo.

Basada en un backend .NET real, revisado a fondo el 2026-07-18 leyendo
directamente el código fuente. Ante conflicto entre esta skill y tu código
real, gana el código real.

## Motor

- .NET, Clean Architecture, **sin Entity Framework** — solo Dapper +
  Stored Procedures (ver skill `sql-database-patterns` para el lado BD).
- 10 proyectos en una sola solución, agrupados en 5 capas (Solution Folders
  virtuales, no carpetas físicas — ver `references/project-structure.md`).
- Entidades agrupadas por carpeta de negocio — ver skill
  `business-domain-grouping` para el mapeo esquema↔carpeta.

## Decisiones persistentes entre sesiones

Antes de "Antes de crear", verifica si existe `.claude/skill-decisions.md`
en el proyecto:
- Sección **`## Transversal`** → si ya tiene "Idioma de nomenclatura", no
  la vuelvas a preguntar (es compartida con `sql-database-patterns` y
  `angular-feature-architecture` — la fija la primera skill que se use en
  el proyecto).
- Sección **`## dotnet-clean-architecture`** → si existe, léela y aplica
  esas decisiones (versión de .NET + prefijo) directamente.
- Si ninguna existe todavía, es la primera vez en este proyecto: haz las
  preguntas obligatorias de abajo y crea/completa ambas secciones.

Es un archivo del **proyecto que usa la skill** (no de la skill en sí),
para que una sesión nueva (otra terminal, otro IDE) continúe sobre lo ya
decidido en vez de volver a preguntar todo como si fuera la primera vez.
Formato:

```markdown
## Transversal

- Idioma de nomenclatura: {español|inglés}

## dotnet-clean-architecture

- Versión de .NET: {versión}
- Prefijo: {Prefijo}

### Excepciones por entidad
- {Entidad}: {excepción puntual}
```

Append-only: una excepción nueva se agrega a la lista, nunca se reescribe
una decisión ya tomada salvo que el usuario pida explícitamente cambiarla.

## Antes de crear (obligatorio)

Antes de generar una solución nueva, pregunta siempre:
1. **Versión de .NET** — sugiere por defecto la última estable, pero
   pregunta si el usuario quiere esa u otra.
2. **Prefijo de nomenclatura** de los proyectos (ej. `TuApp`).
3. **¿Idioma de nomenclatura?** (si no está ya fijado en `## Transversal`)
   Español (default) o inglés — aplica a propiedades de Entity/DTOs,
   comentarios, y también a `sql-database-patterns`/
   `angular-feature-architecture` si se usan en el mismo proyecto. Detalle
   de equivalencias (`AuditoriaBase`) en `references/project-structure.md`.

No asumas ninguna — son obligatorias antes de crear cualquier archivo.

## Workflow

1. ¿API nueva desde cero? → `references/project-structure.md` (10 proyectos, `Program.cs`, paquetes).
2. ¿CRUD de una entidad nueva? → sigue el checklist de abajo, un archivo por
   capa: `references/domain-and-dtos.md`, `references/repository-pattern.md`,
   `references/service-pattern.md`, `references/mapper-pattern.md`,
   `references/controller-pattern.md`. Usa `business-domain-grouping` para
   decidir la carpeta `{Core}`.
3. ¿Validaciones o manejo de errores? → `references/validation-and-errors.md`.
4. ¿IDs encriptados, passwords, o auth/JWT? → `references/security-patterns.md`.
5. Siempre → nomenclatura y namespace único de `references/conventions.md`.

## Los 10 proyectos (5 capas)

| Capa | Proyectos |
|---|---|
| 10.Transversal | `Common`, `Mapper`, `Logging`, `Security` |
| 20.Domain | `Domain` |
| 30.Infraestructura | `Infrastructure`, `Persistence` |
| 40.Application | `Application`, `Validator` |
| 50.Presentation | `WebApi` |

**Estas 5 capas son Solution Folders virtuales de Visual Studio** (declaradas
en el `.sln`), no carpetas físicas — en disco los 10 proyectos viven planos
bajo `src/{Prefijo}.{Proyecto}/`. Receta exacta de cómo declarar los
Solution Folders en `references/project-structure.md`.

Flujo de dependencias: `Domain ← Application ← {Persistence, Infrastructure,
Mapper, Validator, Logging} → WebApi`. Detalle completo (contenido de cada
proyecto, DI, `Program.cs`, paquetes) en `references/project-structure.md`.

## Reglas duras

- **Sin Entity Framework.** Todo acceso a datos vía Dapper + Stored
  Procedures (`Sp_{Accion}_{Entidad}` con `@error`/`@msg` OUTPUT — ver skill
  `sql-database-patterns`).
- **Namespace único `{Prefijo}.Application.Interfaces`** para TODAS las
  interfaces (`Repository/`, `Security/`, `Common/`, `Service/`), sin
  importar la subcarpeta física.
- **`IUnitOfWork` con propiedad lazy por repositorio**:
  `public IXRepository X => _x ??= new XRepository(_connection, () =>
  _transaction);`. Repos comparten conexión/transacción del UoW.
  Transacciones sync y async (`BeginTransaction`/`BeginTransactionAsync`, etc).
- **`IValidatorService` centralizado** — los Services inyectan un solo
  `IValidatorService`, nunca `IValidator<T>` directamente.
- **Try/catch solo en la capa Service** ("capturar en la frontera") — los
  repositorios dejan subir la excepción sin capturarla.
- **Controller siempre**: `if (!result.Success) return BadRequest(result);
  return Ok(result);` — sin excepción, en los 5 endpoints CRUD.
- **Sin paginación** — solo filtro vía `{Entidad}SelDto`.
- **Auditoría desde JWT** — `usuario_reg`/`usuario_act` (o `created_by`/
  `updated_by` si el proyecto decidió inglés) nunca vienen del DTO; el
  Service los asigna desde `ICurrentUserService.Username`.
- **`estado` (o `status` en inglés) en Update**: no es parámetro ni del SP
  ni del Service por defecto — se preserva. Solo `Delete` lo cambia a `0`/`false`.
- **IDs siempre encriptados al cliente** (`IIdEncryptionService`) — Domain,
  Repository y BD siguen usando `int`. Nunca crear métodos privados
  `EncryptNullableId` en un Service — usar la extensión `EncryptNullable`.
- **Una entidad = un Service/Controller/Repository propio** — nunca mezclar
  métodos de distintas entidades (aplica a entidades principales, tablas de
  referencia, y tablas junction). Excepción: un Service puede orquestar
  varias entidades en una transacción vía `IUnitOfWork`, pero los GET de
  las entidades auxiliares van en sus propios controllers. Solo se mezcla
  si el usuario lo pide explícitamente.
- **AutoMapper con `MemberList.Source`**, perfiles registrados
  **manualmente** en `Mapper/DependencyInjection.cs` — no hay auto-scan.
  `Mapper/Profiles/` es plano por entidad (única excepción a la agrupación
  por core). `Mapper/Resolvers/` para `IMemberValueResolver` reutilizables
  (ej. desencriptar un ID directamente en el mapeo).
- **Logging**: `AddAppLogging()` — nunca `AddLogging()` (conflictúa con el
  built-in de Microsoft).
- **Comentario de una línea por método** (Repository/Application/
  Infrastructure) explicando qué hace, en el idioma decidido (ver
  `.claude/skill-decisions.md`, español por defecto). Excepción: un
  proceso de Application largo, no-CRUD, con verificaciones o pasos
  extra, lleva un comentario más extenso (mínimo 2, máximo 6 líneas)
  explicando el caso práctico y la secuencia — directo, sin relleno, solo
  más largo.
- **Transacción `UnitOfWork` obligatoria** cuando un método de Application
  ejecuta más de un paso todo-o-nada (ej. crear A y B donde ambos deben
  existir o ninguno). Patrón completo en `references/service-pattern.md`.
- **Presentation**: sufijo por defecto `WebApi`, salvo que el usuario pida
  explícitamente otro nombre.
- **Agrupación por carpeta de negocio (`{Core}`)** en Persistence, Application
  (Services/DTOs/Interfaces), y Validator — ver skill `business-domain-grouping`.

## Checklist: agregar un CRUD de entidad nueva

`{Core}` = carpeta de negocio (ver skill `business-domain-grouping`).

1. `Domain/Entities/{Core}/{Entidad}.cs` heredando `AuditoriaBase` — `references/domain-and-dtos.md`.
2. `Application/DTOs/{Core}/{Entidad}/` — Ins/Upd/Del/Sel/Response — `references/domain-and-dtos.md`.
3. `Application/Interfaces/Repository/{Core}/I{Entidad}Repository.cs` (namespace único) — `references/repository-pattern.md`.
4. `Application/Interfaces/Service/{Core}/I{Entidad}Service.cs` (namespace único) — `references/service-pattern.md`.
5. `Persistence/Repositories/{Core}/{Entidad}Repository.cs` — `references/repository-pattern.md`.
6. Agregar propiedad a `IUnitOfWork` y su lazy-init en `UnitOfWork` — `references/repository-pattern.md`.
7. `Validator/Validators/{Core}/{Entidad}/` — un validador por DTO — `references/validation-and-errors.md`.
8. `Mapper/Profiles/{Entidad}Profile.cs` (plano, sin `{Core}`) + registrar en `Mapper/DependencyInjection.cs` — `references/mapper-pattern.md`.
9. `Application/Services/{Core}/{Entidad}Service.cs` — `references/service-pattern.md`.
10. Registrar `I{Entidad}Service` en `Application/DependencyInjection.cs` — `references/service-pattern.md`.
11. `WebApi/Controllers/{Entidad}sController.cs` con `[Authorize]` — `references/controller-pattern.md`.

## Referencias

- `references/project-structure.md` — 10 proyectos, Solution Folders, DI, `Program.cs`, paquetes NuGet.
- `references/domain-and-dtos.md` — Entity (Domain) + los 5 DTOs.
- `references/repository-pattern.md` — Interfaz + Repository (Dapper) + registro en `IUnitOfWork`.
- `references/service-pattern.md` — Service completo + transacción multi-paso.
- `references/mapper-pattern.md` — Profile + Resolver + registro en `DependencyInjection`.
- `references/controller-pattern.md` — Controller (patrón de 5 endpoints).
- `references/validation-and-errors.md` — FluentValidation, `IValidatorService`, `SpResult`/`ApiResponse`, try/catch.
- `references/security-patterns.md` — IDs encriptados, hashing de passwords, JWT, auditoría desde JWT.
- `references/conventions.md` — nomenclatura, namespace único, regla de una entidad por Service/Controller/Repository.
- Skill `business-domain-grouping` — mapeo esquema↔carpeta de negocio (`{Core}`).
