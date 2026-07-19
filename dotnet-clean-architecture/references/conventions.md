# Conventions

## Nomenclatura

| Elemento | Convención | Ejemplo |
|---|---|---|
| Namespace | `{Project}.{Capa}.{Subcarpeta}` | `{Prefijo}.Application.Services` |
| Namespace de Interfaces | **único**, sin importar subcarpeta física | `{Prefijo}.Application.Interfaces` |
| Clases públicas | PascalCase + sufijo de tipo | `{Entity}Service`, `{Entity}sController` |
| Interfaces | PascalCase + prefijo `I` | `I{Entity}Service`, `IUnitOfWork` |
| Métodos | PascalCase + sufijo `Async` | `GetAllAsync`, `InsertAsync` |
| Propiedades DTO/Entity | **snake_case** (coincide con columnas SQL) | `{entity}_id`, `usuario_reg` |
| Campos privados | `_camelCase` | `_unitOfWork`, `_mapper` |
| Archivo | `{NombreClase}.cs` | `{Entity}InsDto.cs` |
| Controller | `{Entity}sController` (plural pegado) | `PermisosController` |
| Idioma | mixto — dominio en español, infra en inglés | — |

## Agrupación por carpeta de negocio (`{Core}`)

`Persistence/Repositories/`, `Application/Services/`,
`Application/Interfaces/{Repository,Service}/`, `Application/DTOs/`, y
`Validator/Validators/` se agrupan TODOS por `{Core}` (ver skill
`business-domain-grouping`) — nunca afecta el namespace, solo la carpeta
física. Única excepción: `Mapper/Profiles/` es plano por entidad.

## `Application/Common` — propósito real

No es solo "clases transversales" en abstracto. Contiene:
- `SpResult.cs`/`SpResult<T>` — transversal real, usado por todos los repos.
- Clases agregadoras de resultsets múltiples por entidad (ej.
  `{Entidad}Results.cs` con sus filas anidadas, para SPs que retornan varios
  SELECT — ver skill `sql-database-patterns`, patrón de resultset múltiple).

Agregar aquí solo lo que de verdad se reutiliza en 2+ clases de Application
— no es un cajón general.

## `Application/Helpers` — propósito real

Lógica reutilizada por más de una clase de Application, o mecanismos que se
extraen de un método de caso de uso para no engrosarlo (ej. validar
metadata de un archivo adjunto antes de procesar un DTO). Los métodos de
Application deben contener únicamente lógica del caso de uso/negocio — todo
lo demás se disgrega a un Helper.

## Namespace único de Interfaces

Todas las subcarpetas de `Application/Interfaces/` (`Repository/`,
`Security/`, `Common/`, `Service/`) comparten **un solo namespace**:
`{Prefijo}.Application.Interfaces`. No generar `...Interfaces.Repository`,
`...Interfaces.Service`, etc. — es una carpeta física, no un namespace
distinto.

## Una entidad = un Service/Controller/Repository propio

Cada entidad/tabla tiene su propio Service, Controller y Repository
independiente con los métodos que le correspondan. **Nunca mezclar métodos
de distintas entidades en un mismo Service o Controller.** Aplica a:
- Entidades principales.
- Tablas de referencia/catálogo.
- Tablas junction (muchos-a-muchos).

Si un Service necesita orquestar varias entidades en una transacción (ej.
insertar una entidad + sus junctions), accede a los repos vía `IUnitOfWork`
— pero los endpoints GET de las entidades auxiliares van en sus propios
controllers, no en el de la entidad principal.

Métodos adicionales al CRUD son válidos si corresponden a un caso de uso de
esa entidad (ej. `ActivarEmpleado`), pero nunca métodos que pertenecen a
otra entidad.

**Esta es la regla por defecto.** Solo se mezclan métodos de distintas
entidades si el usuario lo pide explícitamente en el prompt.

## Patrón GetAll con filtro genérico (SelDto)

Todo listado usa `GetAllAsync` con un objeto filtro (`{Entity}SelDto`), no
parámetros sueltos — permite agregar filtros futuros sin romper firmas:

1. **SelDto** — campos filtrables con IDs encriptados (strings). Filtros de
   seguridad (ej. `sociedad_id`) vienen del JWT, no del cliente.
2. **Controller** — `GetAll([FromQuery] {Entity}SelDto filter)`.
3. **Service** — crea `new {Entity}()`, desencripta cada filtro
   condicionalmente (`if !string.IsNullOrEmpty`), asigna filtros de
   seguridad del JWT, pasa la entidad al repo.
4. **Repository** — extrae `filter.campo` para cada parámetro del SP; campos
   `null` se pasan como `NULL`.
5. **SP** — parámetros con `= NULL`, `WHERE (@campo IS NULL OR columna = @campo)`.

Agregar un filtro nuevo = 3 toques (SelDto + Service + SP), firmas intactas
en el resto de capas.

## Checklist: agregar un CRUD de entidad nueva

`{Core}` = carpeta de negocio (ver skill `business-domain-grouping`).

1. `Domain/Entities/{Core}/{Entity}.cs` heredando `AuditoriaBase`.
2. DTOs en `Application/DTOs/{Core}/{Entity}/` (Ins/Upd/Del/Sel/Response).
3. `I{Entity}Repository` en `Application/Interfaces/Repository/{Core}/`.
4. `I{Entity}Service` en `Application/Interfaces/Service/{Core}/` (mismo namespace único).
5. `{Entity}Repository` en `Persistence/Repositories/{Core}/`.
6. Agregar propiedad `I{Entity}Repository` a `IUnitOfWork` + lazy-init en `UnitOfWork`.
7. Validadores en `Validator/Validators/{Core}/{Entity}/`.
8. `{Entity}Profile` en `Mapper/Profiles/` (plano, sin `{Core}`) — **y registrarlo** en `Mapper/DependencyInjection.cs`.
9. `{Entity}Service` en `Application/Services/{Core}/`.
10. Registrar `I{Entity}Service` en `Application/DependencyInjection.cs`.
11. `{Entity}sController` en `WebApi/Controllers/` con `[Authorize]`.

Ver código completo de cada paso en `domain-and-dtos.md`,
`repository-pattern.md`, `service-pattern.md`, `mapper-pattern.md` y
`controller-pattern.md` (uno por capa — ver también el checklist enlazado
en `SKILL.md`).
