# Domain Entity + DTOs

Sustituye `{Entity}`/`{Core}` por los valores concretos — `{Core}` es la
carpeta de negocio (ver skill `business-domain-grouping`).

## Entity (Domain/Entities/{Core}/)

```csharp
namespace {Prefijo}.Domain.Entities;

public class {Entity} : AuditoriaBase
{
    public int {entity}_id { get; set; }
    public string? {campo_negocio_1} { get; set; }
    public string? {campo_negocio_2} { get; set; }
}
```

## DTOs (Application/DTOs/{Core}/{Entity}/)

```csharp
// {Entity}InsDto.cs — sin ID, sin auditoría (se asigna en el Service desde el JWT)
public class {Entity}InsDto : AuditoriaInsDto
{
    public string {campo_negocio_1} { get; set; } = string.Empty;
    public string? {campo_negocio_2} { get; set; }
}

// {Entity}UpdDto.cs — ID encriptado (string) + campos de negocio. Hereda AuditoriaUpdDto (solo `estado`)
public class {Entity}UpdDto : AuditoriaUpdDto
{
    public string {entity}_id { get; set; } = string.Empty;
    public string {campo_negocio_1} { get; set; } = string.Empty;
    public string? {campo_negocio_2} { get; set; }
}

// {Entity}DelDto.cs — solo el ID encriptado
public class {Entity}DelDto : AuditoriaDelDto
{
    public string {entity}_id { get; set; } = string.Empty;
}

// {Entity}SelDto.cs — standalone (no hereda Auditoria*), campos de filtro opcionales
public class {Entity}SelDto
{
    public string? {campo_filtro}   { get; set; }
}

// {Entity}ResponseDto.cs — ID encriptado + auditoría completa
public class {Entity}ResponseDto
{
    public string {entity}_id { get; set; } = string.Empty;
    public string? {campo_negocio_1} { get; set; }
    public string? {campo_negocio_2} { get; set; }
    public bool estado { get; set; }
    public string? usuario_reg { get; set; }
    public DateTime? fecha_reg { get; set; }
    public string? usuario_act { get; set; }
    public DateTime? fecha_act { get; set; }
}
```

Bases: `AuditoriaInsDto`/`AuditoriaDelDto` vacías (marcadores), `AuditoriaUpdDto`
solo tiene `estado`.
