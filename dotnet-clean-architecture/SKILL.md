---
name: dotnet-clean-architecture
description: Genera backend .NET Clean Architecture con estas convenciones — 10 proyectos, Dapper + stored procedures (sin EF), DTOs por operación CRUD, ApiResponse unificado, IDs encriptados, IValidatorService centralizado, auditoría desde JWT, y pruebas unitarias (xUnit v3 + NSubstitute) y de integración contra una BD de pruebas. Úsala para crear un API .NET, un CRUD de entidad nuevo o sus pruebas siguiendo estas convenciones.
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

**Antes de un listado (obligatorio):** por cada listar que pagina en el
servidor, pregunta **qué columnas se pueden ordenar** ("Ninguna" es una
respuesta válida). Una sola vez por listado: si ya la respondió la
especificación u otra skill en la misma tarea (`angular-feature-architecture`
la hace al construir el listado), no se repite. Un listar que trae todo no
recibe orden: lo ordena el front en memoria.

En el mismo listar paginado pregunta también **qué columnas llevan filtro**
("Ninguna" es válida) y, por cada una, su tipo de dato y de filtro
(catálogo con casillas o lista, booleano, condición o rango de número o
texto, fecha o rango de fechas), salvo que ya lo respondió la
especificación (línea **Filtros** del CUS) u otra skill en la misma tarea.
Define la lista blanca del `{Entidad}Mapeo` y los parámetros del SP. Un
listar que trae todo no recibe filtros: los aplica el front en memoria.

## Workflow

0. **Siempre, antes de crear o tocar un Service:** verifica que el proyecto tenga la prueba de arquitectura de
   Services (`tests/{Prefijo}.UnitTests/Arquitectura/ServiciosSoloNegocioTests.cs`, plantilla en
   `references/testing.md`). Si no existe, créala primero. Es lo que garantiza la regla "el Service solo tiene lógica
   de negocio y solo los métodos de su interfaz" en cualquier proyecto, la escriba quien la escriba.
   Antes de escribir el Service, lee el "Ejemplo de referencia: antes y después" de `references/service-pattern.md`
   y reparte el código en sus cuatro archivos (`Mensajes{Core}`, `{Entidad}Mapeo`, `{Entidad}Helper`, Service) con
   esa misma forma. Al terminar, `dotnet test` con la prueba de arquitectura en verde.
   Igual para repositorios, controllers y servicios externos: la prueba `CapasSoloSuResponsabilidadTests`
   (`references/testing.md`) y los helpers únicos `SpEjecutor` (`references/repository-pattern.md`), `RespuestaHttp`
   (`references/controller-pattern.md`) y `JwtEmisor`/`JwtClaims` (`references/security-patterns.md`), copiados tal
   cual, deben existir antes de escribir el primer repositorio, controller o adaptador.
   Y la prueba `ValidatorPorCarpetaDeNegocioTests` antes del primer validador.
1. ¿API nueva desde cero? → `references/project-structure.md` (10 proyectos, `Program.cs`, paquetes).
2. ¿CRUD de una entidad nueva? → sigue el checklist de abajo, un archivo por
   capa: `references/domain-and-dtos.md`, `references/repository-pattern.md`,
   `references/service-pattern.md`, `references/mapper-pattern.md`,
   `references/controller-pattern.md`. Usa `business-domain-grouping` para
   decidir la carpeta `{Core}`.
3. ¿Validaciones o manejo de errores? → `references/validation-and-errors.md`.
4. ¿IDs encriptados, passwords, o auth/JWT? → `references/security-patterns.md`.
5. ¿Vas a instalar un paquete NuGet (nuevo, o un bump de versión mayor de
   uno existente)? → skill `package-security-vetting` **antes** de
   `dotnet add package` — obligatorio, salvo que ya esté en la tabla de
   paquetes de `references/project-structure.md`.
6. Siempre → nomenclatura y namespace único de `references/conventions.md`.
7. Siempre, al terminar una funcionalidad → sus pruebas unitarias y de
   integración: `references/testing.md`. La funcionalidad no está terminada
   hasta que todas las pruebas de la solución están en verde.

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

- **El Service solo expone los métodos que usa el controller** (los de su interfaz: en un CRUD, exactamente
  `GetAllAsync`, `GetByIdAsync`, `InsertAsync`, `UpdateAsync` y `DeleteAsync`). Ningún otro método, ni privado: un paso
  de apoyo va en el helper estático `Application/Helpers/{Core}/{Entidad}Helper.cs`, que recibe por parámetro lo que
  necesita (`IUnitOfWork`, `IIdEncryptionService`, `IAppLogger<T>`).
- **El Service solo tiene lógica de negocio.** Validaciones, condicionales, flujos, transacciones e invocaciones
  (repositorio, validador, Mapper, helpers, logger); nunca constantes, mensajes literales, tipos internos,
  transformaciones de datos ni mecanismos. Mensajes en `Application/Common/Mensajes/Mensajes{Core}.cs`; entidad → DTO
  de respuesta, árboles, páginas, filtros y JSON de SPs en el helper estático
  `Application/Helpers/{Core}/{Entidad}Mapeo.cs`; DTO de entrada → entidad en AutoMapper. Detalle y tabla de "qué va
  dónde" y ejemplo completo antes/después en `references/service-pattern.md`. **Se garantiza con la prueba de arquitectura obligatoria** (Workflow,
  paso 0): si falla, se mueve el código a su lugar; nunca se relaja la prueba.
- **El repositorio (y el `UnitOfWork`) solo tiene los métodos de su interfaz**, y cada uno solo declara el SP y sus
  parámetros de negocio: el mecanismo (`@error`/`@msg`, ejecución, `SpResult`, ID generado, valores por defecto) va
  en `Persistence/Helpers/SpEjecutor.cs`; parámetros compartidos en `{Entidad}Parametros`. Únicos textos: el nombre
  del SP y de sus `@parámetros`. Ejemplo en `references/repository-pattern.md`.
- **El controller solo tiene endpoints del negocio**: cada uno es `this.Responder(await _service.XAsync(...))`; el
  model binding arma los DTOs (`[FromRoute]` en el Delete), los claims se leen con helpers de `WebApi/Helpers/`.
  Únicos textos: los de sus atributos. Ejemplo en `references/controller-pattern.md`.
- **El servicio externo (Security, Logging, Infrastructure, `WebApi/Services`) solo tiene los métodos de su
  interfaz**: pasos de apoyo en `Helpers/`, nombres de claims en `Application/Common/ClaimsSesion.cs`, textos en
  `Application/Common/Mensajes/MensajesTecnicos.cs`; sin textos literales. Ejemplo en `references/security-patterns.md`.
- Estas tres reglas, como la del Service, **se garantizan con una prueba de arquitectura obligatoria**
  (`CapasSoloSuResponsabilidadTests`, Workflow paso 0); si falla, se mueve el código; nunca se relaja la prueba.
- **Validator por carpeta de negocio:** `Validators/` solo tiene validadores; las reglas reutilizables van en
  `Rules/{Core}/{Entidad o proceso}/`, en la carpeta del negocio cuya lógica expresan (no la del consumidor ni la de la
  entidad que guarda el campo: `PasswordRules` es de `Auth`), o en `Rules/Comun/` si no tienen dueño de negocio.
  Lo garantiza la prueba `ValidatorPorCarpetaDeNegocioTests`.
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
- **Pruebas obligatorias** (`references/testing.md`): toda funcionalidad se
  entrega con pruebas unitarias (Services y validadores, dependencias
  simuladas) y de integración (API en memoria contra una BD de pruebas
  recreada con los scripts del repo). Cada validación del SP se provoca con
  un request real y se verifica su mensaje exacto. Nada se da por terminado
  ni se sube con una prueba en rojo; una prueba en rojo se corrige arreglando
  el código, no la prueba.

- **Orden por columna en los listar paginados (obligatorio si el listado
  tiene columnas ordenables).** El request lleva
  `List<OrdenCampoDto>? orden` (`[{ campo, sentido }]`, una lista aunque la
  UI ordene por una columna); el `{Entidad}Mapeo` lo resuelve con
  `OrdenListado.Resolver(dto.orden, CamposOrden)` contra **su lista blanca**
  (campo del front → valor que entiende el SP) y pasa al filtro
  `orden_campo` (`null` = sin orden: orden de registro por PK) y
  `orden_sentido` (`"DESC"` o `"ASC"`). Campo desconocido → sin orden;
  sentido inválido → `ASC`; nunca un nombre de columna libre hacia el SP.
  `OrdenCampoDto` (`DTOs/Base/`) y `OrdenListado` (`Helpers/`) se crean una
  vez por proyecto, copiados tal cual. Patrón completo:
  `references/domain-and-dtos.md`, "Orden por columna de los listados".

- **Filtros por columna en los listar paginados (obligatorio si el listado
  tiene columnas filtrables).** El request lleva
  `List<FiltroCampoDto>? filtros` (`[{ campo, operador, valores }]`); el
  `{Entidad}Mapeo` lo resuelve con
  `FiltroListado.Resolver(dto.filtros, CamposFiltro, enc)` contra **su lista
  blanca** (`campo` del front → `CampoFiltro` con su tipo de dato y sus
  operadores permitidos) y pasa al filtro de dominio **valores tipados por
  columna** (IDs desencriptados como arreglo JSON, `bool?`, texto con su
  operador, número o fecha con desde/hasta). Un filtro inválido (campo u
  operador fuera de la lista, valor que no es de su tipo, ID que no
  desencripta) **se ignora, nunca da error**. Nunca un nombre de columna ni
  un operador libre hacia el SP. `FiltroCampoDto` (`DTOs/Base/`) y
  `FiltroListado` (`Helpers/`) se crean una vez por proyecto. Patrón
  completo: `references/domain-and-dtos.md`, "Filtros por columna de los
  listados".

## Checklist: agregar un CRUD de entidad nueva

`{Core}` = carpeta de negocio (ver skill `business-domain-grouping`).

1. `Domain/Entities/{Core}/{Entidad}.cs` heredando `AuditoriaBase` — `references/domain-and-dtos.md`.
2. `Application/DTOs/{Core}/{Entidad}/` — Ins/Upd/Del/Sel/Response — `references/domain-and-dtos.md`.
   Si el listar pagina y tiene columnas ordenables: `orden` en el `SelDto`,
   `CamposOrden` en el `{Entidad}Mapeo` y `orden_campo`/`orden_sentido` en el
   filtro — `references/domain-and-dtos.md`, "Orden por columna de los listados".
   Si tiene columnas filtrables: `filtros` en el `SelDto`, `CamposFiltro` en
   el `{Entidad}Mapeo` y un campo tipado por columna en el filtro —
   `references/domain-and-dtos.md`, "Filtros por columna de los listados".
3. `Application/Interfaces/Repository/{Core}/I{Entidad}Repository.cs` (namespace único) — `references/repository-pattern.md`.
4. `Application/Interfaces/Service/{Core}/I{Entidad}Service.cs` (namespace único) — `references/service-pattern.md`.
5. `Persistence/Repositories/{Core}/{Entidad}Repository.cs` — `references/repository-pattern.md` (cada método: `SpEjecutor.Parametros()` + parámetros de negocio + una llamada a `SpEjecutor`).
6. Agregar propiedad a `IUnitOfWork` y su lazy-init en `UnitOfWork` — `references/repository-pattern.md`.
7. `Validator/Validators/{Core}/{Entidad}/` — un validador por DTO; reglas que comparten dos o más validadores en `Validator/Rules/{Core}/{Entidad o proceso}/` (carpeta del negocio cuya lógica expresan) — `references/validation-and-errors.md`.
8. `Mapper/Profiles/{Entidad}Profile.cs` (plano, sin `{Core}`) + registrar en `Mapper/DependencyInjection.cs` — `references/mapper-pattern.md`.
9. `Application/Services/{Core}/{Entidad}Service.cs` (solo lógica de negocio) + sus mensajes en
   `Application/Common/Mensajes/Mensajes{Core}.cs` + su helper `Application/Helpers/{Core}/{Entidad}Mapeo.cs` —
   `references/service-pattern.md`.
10. Registrar `I{Entidad}Service` en `Application/DependencyInjection.cs` — `references/service-pattern.md`.
11. `WebApi/Controllers/{Entidad}sController.cs` con `[Authorize]` — `references/controller-pattern.md` (cada endpoint: `this.Responder(await ...)`; Delete con `[FromRoute] {Entidad}DelDto`).
12. Pruebas: `tests/{Prefijo}.UnitTests/Application/Services/{Core}/{Entidad}ServiceTests.cs`,
    `tests/{Prefijo}.UnitTests/Validator/{Core}/{Entidad}/` y
    `tests/{Prefijo}.IntegrationTests/Api/{Core}/{Entidad}/{Entidad}sApiTests.cs`;
    `dotnet test` en verde, incluida la prueba de arquitectura de Services — `references/testing.md`.

## Referencias

- `references/project-structure.md` — 10 proyectos, Solution Folders, DI, `Program.cs`, paquetes NuGet.
- `references/domain-and-dtos.md` — Entity (Domain) + los 5 DTOs + orden por columna de los listados (`OrdenCampoDto`, `OrdenListado`, lista blanca) + filtros por columna (`FiltroCampoDto`, `FiltroListado`, `CampoFiltro`).
- `references/repository-pattern.md` — Interfaz + Repository (Dapper) + registro en `IUnitOfWork`.
- `references/service-pattern.md` — Service completo + transacción multi-paso.
- `references/mapper-pattern.md` — Profile + Resolver + registro en `DependencyInjection`.
- `references/controller-pattern.md` — Controller (patrón de 5 endpoints).
- `references/validation-and-errors.md` — FluentValidation, `IValidatorService`, `SpResult`/`ApiResponse`, try/catch.
- `references/security-patterns.md` — IDs encriptados, hashing de passwords, JWT, auditoría desde JWT.
- `references/conventions.md` — nomenclatura, namespace único, regla de una entidad por Service/Controller/Repository.
- `references/testing.md` — proyectos de prueba unitaria e integración, BD de pruebas, qué se prueba por CRUD, trazabilidad.
- Skill `business-domain-grouping` — mapeo esquema↔carpeta de negocio (`{Core}`).
- Skill `package-security-vetting` — chequeo de seguridad obligatorio antes de instalar cualquier paquete NuGet fuera de la tabla de defaults.
