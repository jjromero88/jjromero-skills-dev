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


## Orden por columna de los listados

Mecanismo obligatorio de todo listar **paginado en el servidor** con
columnas ordenables (cuáles: pregunta obligatoria del `SKILL.md`; "Ninguna"
es válida y entonces no se agrega nada). Un listar que trae todo no lleva
orden: lo ordena el front en memoria. Contrato con el front: lista de
criterios aunque la UI ordene por una sola columna; el servidor aplica el
primero válido; lista vacía = sin orden = **orden de registro (PK)**, como
Excel sin ordenar.

```json
"orden": [ { "campo": "descripcion", "sentido": "ASC" } ]
```

En un `GET` viaja enlazado con `[FromQuery]`:
`?orden[0].campo=descripcion&orden[0].sentido=DESC`.

**Piezas compartidas** (se crean una vez por proyecto, copiadas tal cual):

```csharp
// Application/DTOs/Base/OrdenCampoDto.cs — un criterio de orden del listado
public class OrdenCampoDto
{
    public string? campo { get; set; }
    public string? sentido { get; set; }
}

// Application/Helpers/OrdenListado.cs
// Orden resuelto: el valor que entiende el SP (null = sin orden: orden de registro) y el sentido normalizado
public sealed record OrdenResuelto(string? Campo, string Sentido);

public static class OrdenListado
{
    public const string Ascendente = "ASC";
    public const string Descendente = "DESC";

    // Primer criterio cuyo campo esta en la lista blanca (sin distinguir mayusculas ni espacios extremos), con su
    // sentido normalizado. Sin criterios o sin ninguno valido: el defecto del listado (por omision, sin orden)
    public static OrdenResuelto Resolver(
        IEnumerable<OrdenCampoDto>? orden,
        IReadOnlyDictionary<string, string> permitidos,
        string? campoDefecto = null,
        string sentidoDefecto = Ascendente)
    {
        foreach (var criterio in orden ?? [])
        {
            var campo = criterio?.campo?.Trim();
            if (string.IsNullOrEmpty(campo)) continue;
            foreach (var permitido in permitidos)
                if (string.Equals(permitido.Key, campo, StringComparison.OrdinalIgnoreCase))
                    return new OrdenResuelto(permitido.Value, Sentido(criterio!.sentido));
        }
        return new OrdenResuelto(campoDefecto, campoDefecto is null ? Ascendente : Sentido(sentidoDefecto));
    }

    // "DESC" (sin distinguir mayusculas ni espacios extremos) es descendente; cualquier otro valor, ascendente
    public static string Sentido(string? sentido) =>
        string.Equals(sentido?.Trim(), Descendente, StringComparison.OrdinalIgnoreCase) ? Descendente : Ascendente;
}
```

**Por entidad:**

```csharp
// {Entity}SelDto.cs
public List<OrdenCampoDto>? orden { get; set; }        // vacio = sin orden (orden de registro)

// Helpers/{Core}/{Entity}Mapeo.cs — lista blanca: campo del front -> valor de @orden_campo del SP
public static readonly IReadOnlyDictionary<string, string> CamposOrden = new Dictionary<string, string>
{
    ["{campo_1}"] = "{campo_1}",
    ["{campo_2}"] = "{campo_2}"
};

public static {Entity}Filtro Filtro({Entity}SelDto dto)
{
    var orden = OrdenListado.Resolver(dto.orden, CamposOrden);
    return new {Entity}Filtro { /* busqueda, filtros, pagina... */ orden_campo = orden.Campo, orden_sentido = orden.Sentido };
}

// Domain/Entities/{Core}/{Entity}Filtro.cs
public string? orden_campo { get; set; }               // null = sin orden
public string orden_sentido { get; set; } = "ASC";

// Repositorio: dos parametros mas hacia el Sp_Sel
parameters.Add("@orden_campo", filtro.orden_campo, DbType.String, size: 30);
parameters.Add("@orden_sentido", filtro.orden_sentido, DbType.String, size: 4);
```

El SP vuelve a validar el orden y ordena sin SQL dinámico
(`sql-database-patterns`, `references/sp-templates.md`, "Sp_Sel paginado con
orden por columna"). Pruebas obligatorias en `references/testing.md`.
