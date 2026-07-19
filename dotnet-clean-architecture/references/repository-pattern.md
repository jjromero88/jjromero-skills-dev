# Repository Pattern

Sustituye `{Entity}`/`{Core}`/`{esquema}` por los valores concretos. Los SPs
invocados (`{esquema}.Sp_Ins_{Entity}`, etc.) siguen la skill
`sql-database-patterns`. Comentario de una línea por método.

## Interfaz (Application/Interfaces/Repository/{Core}/)

Namespace único — siempre `{Prefijo}.Application.Interfaces`, sin importar
la subcarpeta física:

```csharp
namespace {Prefijo}.Application.Interfaces;

public interface I{Entity}Repository
{
    Task<SpResult<IEnumerable<{Entity}>>> GetAllAsync();
    Task<SpResult<{Entity}>> GetByIdAsync(int {entity}Id);
    Task<SpResult> InsertAsync({Entity} {entity});
    Task<SpResult> UpdateAsync({Entity} {entity});
    Task<SpResult> DeleteAsync({Entity} {entity});
}
```

## Repositorio (Persistence/Repositories/{Core}/)

Constructor recibe conexión y transacción compartidas del `UnitOfWork`.
`DynamicParameters` con `DbType`/`size` explícitos. Sin try/catch — deja
subir la excepción (se captura en el Service).

```csharp
namespace {Prefijo}.Persistence.Repositories;

public class {Entity}Repository : I{Entity}Repository
{
    private readonly IDbConnection _connection;
    private readonly Func<IDbTransaction?> _getTransaction;

    public {Entity}Repository(IDbConnection connection, Func<IDbTransaction?> getTransaction)
    {
        _connection = connection;
        _getTransaction = getTransaction;
    }

    // Lista todos los {entity} activos
    public async Task<SpResult<IEnumerable<{Entity}>>> GetAllAsync()
    {
        var parameters = new DynamicParameters();
        parameters.Add("@error", dbType: DbType.Boolean, direction: ParameterDirection.Output);
        parameters.Add("@msg", dbType: DbType.String, size: 500, direction: ParameterDirection.Output);

        var result = await _connection.QueryAsync<{Entity}>(
            "{esquema}.Sp_Sel_{Entity}", parameters,
            transaction: _getTransaction(), commandType: CommandType.StoredProcedure);

        return new SpResult<IEnumerable<{Entity}>>
        {
            Error = parameters.Get<bool>("@error"),
            Msg = parameters.Get<string>("@msg"),
            Data = result
        };
    }

    // Obtiene un {entity} por su ID
    public async Task<SpResult<{Entity}>> GetByIdAsync(int {entity}Id)
    {
        var parameters = new DynamicParameters();
        parameters.Add("@{entity}_id", {entity}Id, DbType.Int32);
        parameters.Add("@error", dbType: DbType.Boolean, direction: ParameterDirection.Output);
        parameters.Add("@msg", dbType: DbType.String, size: 500, direction: ParameterDirection.Output);

        var result = await _connection.QueryFirstOrDefaultAsync<{Entity}>(
            "{esquema}.Sp_Get_{Entity}", parameters,
            transaction: _getTransaction(), commandType: CommandType.StoredProcedure);

        return new SpResult<{Entity}>
        {
            Error = parameters.Get<bool>("@error"),
            Msg = parameters.Get<string>("@msg"),
            Data = result
        };
    }

    // Inserta un nuevo {entity}
    public async Task<SpResult> InsertAsync({Entity} {entity})
    {
        var parameters = new DynamicParameters();
        parameters.Add("@{campo_negocio_1}", {entity}.{campo_negocio_1}, DbType.String, size: 100);
        parameters.Add("@{campo_negocio_2}", {entity}.{campo_negocio_2}, DbType.String, size: 200);
        parameters.Add("@usuario_reg", {entity}.usuario_reg, DbType.String, size: 50);
        parameters.Add("@error", dbType: DbType.Boolean, direction: ParameterDirection.Output);
        parameters.Add("@msg", dbType: DbType.String, size: 500, direction: ParameterDirection.Output);

        await _connection.ExecuteAsync(
            "{esquema}.Sp_Ins_{Entity}", parameters,
            transaction: _getTransaction(), commandType: CommandType.StoredProcedure);

        return new SpResult { Error = parameters.Get<bool>("@error"), Msg = parameters.Get<string>("@msg") };
    }

    // Actualiza un {entity} existente
    public async Task<SpResult> UpdateAsync({Entity} {entity})
    {
        var parameters = new DynamicParameters();
        parameters.Add("@{entity}_id", {entity}.{entity}_id, DbType.Int32);
        parameters.Add("@{campo_negocio_1}", {entity}.{campo_negocio_1}, DbType.String, size: 100);
        parameters.Add("@{campo_negocio_2}", {entity}.{campo_negocio_2}, DbType.String, size: 200);
        parameters.Add("@usuario_act", {entity}.usuario_act, DbType.String, size: 50);
        parameters.Add("@error", dbType: DbType.Boolean, direction: ParameterDirection.Output);
        parameters.Add("@msg", dbType: DbType.String, size: 500, direction: ParameterDirection.Output);

        await _connection.ExecuteAsync(
            "{esquema}.Sp_Upd_{Entity}", parameters,
            transaction: _getTransaction(), commandType: CommandType.StoredProcedure);

        return new SpResult { Error = parameters.Get<bool>("@error"), Msg = parameters.Get<string>("@msg") };
    }

    // Elimina logicamente un {entity}
    public async Task<SpResult> DeleteAsync({Entity} {entity})
    {
        var parameters = new DynamicParameters();
        parameters.Add("@{entity}_id", {entity}.{entity}_id, DbType.Int32);
        parameters.Add("@usuario_act", {entity}.usuario_act, DbType.String, size: 50);
        parameters.Add("@error", dbType: DbType.Boolean, direction: ParameterDirection.Output);
        parameters.Add("@msg", dbType: DbType.String, size: 500, direction: ParameterDirection.Output);

        await _connection.ExecuteAsync(
            "{esquema}.Sp_Del_{Entity}", parameters,
            transaction: _getTransaction(), commandType: CommandType.StoredProcedure);

        return new SpResult { Error = parameters.Get<bool>("@error"), Msg = parameters.Get<string>("@msg") };
    }
}
```

`estado`/`{entity}_id` NO son parámetros del `UpdateAsync` — se preservan en
el SP (ver skill de BD, `Sp_Upd` nunca los toca).

## `IUnitOfWork` — agregar la propiedad lazy

```csharp
// En IUnitOfWork.cs
I{Entity}Repository {Entity}s { get; }

// En UnitOfWork.cs
private I{Entity}Repository? _{entity}s;
public I{Entity}Repository {Entity}s =>
    _{entity}s ??= new {Entity}Repository(_connection, () => _transaction);
```
