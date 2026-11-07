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

## Reglas de arquitectura del repositorio (reglas duras)

1. **El repositorio solo tiene los métodos de su interfaz** `I{Entity}Repository`: los que llaman a un SP. Ningún
   otro método, ni privado: parámetros que comparten varios métodos van en un helper estático
   `Persistence/Helpers/{Core}/{Entity}Parametros.cs`. Lo mismo aplica al `UnitOfWork` (sus guardas de transacción van
   en `Persistence/Helpers/TransaccionHelper.cs`).
2. **Cada método solo declara el SP y sus parámetros de negocio**: `SpEjecutor.Parametros()`, un `parameters.Add`
   por campo (con `DbType`/`size`) y una llamada a `SpEjecutor`. El mecanismo (salidas `@error`/`@msg`, `CommandType`,
   transacción, armado del `SpResult`, leer el ID generado, valores por defecto con `??`, el `0 → null` del ID) vive
   una sola vez en `Persistence/Helpers/SpEjecutor.cs`.
3. **Únicos textos permitidos:** el nombre del SP (`"{esquema}.Sp_Accion_{Entity}"`) y el de sus parámetros
   (`"@campo"`). Son el contrato con el SP y tienen un solo llamador: no van a una clase de constantes.
4. Sin constantes, campos estáticos ni tipos internos.

Lo garantiza la prueba de arquitectura `CapasSoloSuResponsabilidadTests` (`references/testing.md`). Si falla, se
mueve el código a su lugar; nunca se relaja la prueba.

| En el repositorio NO va | Va en |
|---|---|
| `parameters.Add("@error"…)`, `parameters.Add("@msg"…)`, `new DynamicParameters()` | `SpEjecutor.Parametros()` |
| `_connection.QueryAsync/QueryFirstOrDefaultAsync/ExecuteAsync(..., commandType: …)` + `new SpResult {…}` | `_connection.ListarAsync<T>` / `ObtenerAsync<T>` / `EjecutarAsync` / `EjecutarConIdAsync` |
| `entidad.x_id > 0 ? entidad.x_id : null` con `InputOutput` | `parameters.AgregarIdEntradaSalida("@x_id", entidad.x_id)` |
| `entidad.habilitado ?? true` | `parameters.AgregarBit("@habilitado", entidad.habilitado, siFalta: true)` |
| Método privado con parámetros compartidos (alta y edición) | `{Entity}Parametros.Datos(entidad)` en `Persistence/Helpers/{Core}/` |

## Helper de ejecución (Persistence/Helpers/SpEjecutor.cs) — se crea una vez por proyecto, copiado tal cual

```csharp
using System.Data;
using {Prefijo}.Application.Common;
using Dapper;

namespace {Prefijo}.Persistence.Helpers;

// Mecanismo comun de todo repositorio: parametros de salida @error/@msg, ejecucion del SP en la transaccion del
// UnitOfWork y armado del SpResult. El repositorio solo declara el SP y sus parametros de negocio
public static class SpEjecutor
{
    private const string ParametroError = "@error";
    private const string ParametroMsg = "@msg";
    private const int LargoMsg = 500;

    // Parametros nuevos con las salidas @error y @msg que devuelve todo SP (skill sql-database-patterns)
    public static DynamicParameters Parametros()
    {
        var parameters = new DynamicParameters();
        parameters.Add(ParametroError, dbType: DbType.Boolean, direction: ParameterDirection.Output);
        parameters.Add(ParametroMsg, dbType: DbType.String, size: LargoMsg, direction: ParameterDirection.Output);
        return parameters;
    }

    // ID de entrada y salida del alta: 0 usa el identity; con valor inserta ese ID
    public static void AgregarIdEntradaSalida(this DynamicParameters parameters, string nombre, int id) =>
        parameters.Add(nombre, id > 0 ? id : null, DbType.Int32, ParameterDirection.InputOutput);

    // Bit con valor por defecto cuando la entidad no lo trae
    public static void AgregarBit(this DynamicParameters parameters, string nombre, bool? valor, bool siFalta) =>
        parameters.Add(nombre, valor ?? siFalta, DbType.Boolean);

    // SP que devuelve una lista
    public static async Task<SpResult<IEnumerable<T>>> ListarAsync<T>(
        this IDbConnection connection, string sp, DynamicParameters parameters, IDbTransaction? transaction)
    {
        var data = await connection.QueryAsync<T>(sp, parameters, transaction, commandType: CommandType.StoredProcedure);
        return new SpResult<IEnumerable<T>> { Error = Error(parameters), Msg = Msg(parameters), Data = data };
    }

    // SP que devuelve un registro (o ninguno)
    public static async Task<SpResult<T>> ObtenerAsync<T>(
        this IDbConnection connection, string sp, DynamicParameters parameters, IDbTransaction? transaction)
    {
        var data = await connection.QueryFirstOrDefaultAsync<T>(sp, parameters, transaction, commandType: CommandType.StoredProcedure);
        return new SpResult<T> { Error = Error(parameters), Msg = Msg(parameters), Data = data };
    }

    // SP que solo ejecuta (modificar, eliminar, acciones)
    public static async Task<SpResult> EjecutarAsync(
        this IDbConnection connection, string sp, DynamicParameters parameters, IDbTransaction? transaction)
    {
        await connection.ExecuteAsync(sp, parameters, transaction, commandType: CommandType.StoredProcedure);
        return new SpResult { Error = Error(parameters), Msg = Msg(parameters) };
    }

    // SP de alta que devuelve el ID generado en su parametro de entrada y salida
    public static async Task<SpResult<int>> EjecutarConIdAsync(
        this IDbConnection connection, string sp, DynamicParameters parameters, IDbTransaction? transaction, string parametroId)
    {
        await connection.ExecuteAsync(sp, parameters, transaction, commandType: CommandType.StoredProcedure);
        return new SpResult<int> { Error = Error(parameters), Msg = Msg(parameters), Data = parameters.Get<int?>(parametroId) ?? 0 };
    }

    private static bool Error(DynamicParameters parameters) => parameters.Get<bool>(ParametroError);

    private static string Msg(DynamicParameters parameters) => parameters.Get<string>(ParametroMsg);
}
```

## Repositorio (Persistence/Repositories/{Core}/)

Constructor recibe conexión y transacción compartidas del `UnitOfWork`. Sin try/catch — deja subir la excepción (se
captura en el Service). Cada método: parámetros de negocio + una llamada a `SpEjecutor`.

```csharp
using System.Data;
using {Prefijo}.Application.Common;
using {Prefijo}.Application.Interfaces;
using {Prefijo}.Domain.Entities;
using {Prefijo}.Persistence.Helpers;

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
        var parameters = SpEjecutor.Parametros();

        return await _connection.ListarAsync<{Entity}>("{esquema}.Sp_Sel_{Entity}", parameters, _getTransaction());
    }

    // Obtiene un {entity} por su ID
    public async Task<SpResult<{Entity}>> GetByIdAsync(int {entity}Id)
    {
        var parameters = SpEjecutor.Parametros();
        parameters.Add("@{entity}_id", {entity}Id, DbType.Int32);

        return await _connection.ObtenerAsync<{Entity}>("{esquema}.Sp_Get_{Entity}", parameters, _getTransaction());
    }

    // Inserta un nuevo {entity} y retorna su ID
    public async Task<SpResult<int>> InsertAsync({Entity} {entity})
    {
        var parameters = SpEjecutor.Parametros();
        parameters.Add("@{campo_negocio_1}", {entity}.{campo_negocio_1}, DbType.String, size: 100);
        parameters.Add("@{campo_negocio_2}", {entity}.{campo_negocio_2}, DbType.String, size: 200);
        parameters.AgregarBit("@habilitado", {entity}.habilitado, siFalta: true);
        parameters.Add("@usuario_reg", {entity}.usuario_reg, DbType.String, size: 50);
        parameters.AgregarIdEntradaSalida("@{entity}_id", {entity}.{entity}_id);

        return await _connection.EjecutarConIdAsync("{esquema}.Sp_Ins_{Entity}", parameters, _getTransaction(), "@{entity}_id");
    }

    // Actualiza un {entity} existente
    public async Task<SpResult> UpdateAsync({Entity} {entity})
    {
        var parameters = SpEjecutor.Parametros();
        parameters.Add("@{entity}_id", {entity}.{entity}_id, DbType.Int32);
        parameters.Add("@{campo_negocio_1}", {entity}.{campo_negocio_1}, DbType.String, size: 100);
        parameters.Add("@{campo_negocio_2}", {entity}.{campo_negocio_2}, DbType.String, size: 200);
        parameters.Add("@usuario_act", {entity}.usuario_act, DbType.String, size: 50);

        return await _connection.EjecutarAsync("{esquema}.Sp_Upd_{Entity}", parameters, _getTransaction());
    }

    // Elimina logicamente un {entity}
    public async Task<SpResult> DeleteAsync({Entity} {entity})
    {
        var parameters = SpEjecutor.Parametros();
        parameters.Add("@{entity}_id", {entity}.{entity}_id, DbType.Int32);
        parameters.Add("@usuario_act", {entity}.usuario_act, DbType.String, size: 50);

        return await _connection.EjecutarAsync("{esquema}.Sp_Del_{Entity}", parameters, _getTransaction());
    }
}
```

Parámetros compartidos por alta y edición (muchos campos), en `Persistence/Helpers/{Core}/{Entity}Parametros.cs`:

```csharp
namespace {Prefijo}.Persistence.Helpers;

public static class {Entity}Parametros
{
    // Parametros comunes del registro y la modificacion
    public static DynamicParameters Datos({Entity} {entity})
    {
        var parameters = SpEjecutor.Parametros();
        parameters.Add("@{campo_negocio_1}", {entity}.{campo_negocio_1}, DbType.String, size: 100);
        parameters.Add("@{campo_negocio_2}", {entity}.{campo_negocio_2}, DbType.String, size: 200);
        return parameters;
    }
}
// En el repositorio: var parameters = {Entity}Parametros.Datos({entity}); + los propios del alta o la edicion
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
