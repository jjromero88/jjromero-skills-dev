# Service Pattern

Sustituye `{Entity}`/`{Core}` por los valores concretos.

## Reglas de arquitectura del Service (reglas duras)

**1. El Service solo expone los métodos que usa el controller.** Son los métodos de su interfaz
`I{Entity}Service`: en un CRUD, exactamente `GetAllAsync`, `GetByIdAsync`, `InsertAsync`, `UpdateAsync` y
`DeleteAsync`; en un proceso, sus operaciones de negocio (ej. `LoginAsync`, `AuthorizeAsync`). **Ningún otro método**,
ni privado ni protegido: un paso de apoyo (consultar algo para armar la respuesta, decidir qué rechazo devolver,
notificar fire-and-forget) no es un método de negocio, es un mecanismo que suma a uno, y va en un helper estático
(`Application/Helpers/{Core}/{Entity}Helper.cs`) que recibe lo que necesita como parámetros (`IUnitOfWork`,
`IIdEncryptionService`, `IAppLogger<T>`…). Así, al abrir un Service se ve de un vistazo qué ofrece al controller.

**2. El Service solo tiene lógica de negocio.**

Al abrir un Service solo se ve lógica de negocio: validaciones, condicionales, flujos, transacciones e invocaciones
(repositorio vía `IUnitOfWork`, validador, Mapper, helpers, logger). **Nunca** declara constantes ni mensajes, ni
transforma datos ni implementa un mecanismo. Cada cosa va a su lugar:

| En el Service NO va | Va en | Ejemplo |
|---|---|---|
| Constantes y mensajes (`const`, `static readonly`, textos literales en `Fail`/`Ok`) | `Application/Common/Mensajes/Mensajes{Core}.cs` (uno por esquema; `Mensajes.cs` para los transversales). Un mensaje con datos es un método estático que arma el texto | `MensajesSeg.CredencialesInvalidas`, `MensajesCnf.NodoVerificado(bd, seed)` |
| Constantes de negocio que no son mensajes (grupos, órdenes válidos) | Clase de constantes en `Application/Common/` o en el helper de la entidad | `GruposCatalogo.Marca` |
| Transformar datos: entidad → DTO de respuesta, armar un árbol, una página, un filtro, leer/armar el JSON de un SP, normalizar un texto | Helper estático `Application/Helpers/{Core}/{Entity}Mapeo.cs`; si cifra IDs recibe `IIdEncryptionService enc` como último parámetro | `{Entity}Mapeo.MapToResponseDto(entidad, _encryptionService)` |
| DTO de entrada → entidad | Perfil de AutoMapper (`Mapper/Profiles/`, ver `mapper-pattern.md`) | `_mapper.Map<{Entity}>(dto)` |
| Formas propias de un JSON (records) | Dentro del helper `{Entity}Mapeo`, nunca anidadas en el Service | `public sealed record AccesoEntrada(...)` |
| Normalizar la entrada de un DTO (recortar espacios) | El propio DTO (setter) o el helper | `LoginDto.username` recortado en su setter |

Se permiten en el Service: textos de log (`_logger.LogError(ex, "Error al…")`). Métodos privados, ninguno (regla 1):
lo que antes era un método privado va al helper `{Entity}Mapeo` (transformación) o `{Entity}Helper` (paso de apoyo).

**Las dos reglas se garantizan con una prueba de arquitectura obligatoria en todo proyecto** (`references/testing.md`,
"Prueba de arquitectura de Services"): falla si un Service tiene un método que no es de su interfaz, declara
constantes o tipos internos, escribe textos fuera del log, arma DTOs o JSON, o tiene métodos de transformación. Si no existe en el proyecto, se crea antes de
escribir o tocar un Service. Si falla, se mueve el código a su lugar; nunca se relaja la prueba.

## Ejemplo de referencia: antes y después (copiar esta forma, no interpretarla)

Un Service de proceso (login) escrito **mal**: cada marca `❌` es algo que la prueba de arquitectura rechaza.

```csharp
public class AuthService : IAuthService
{
    private const string CredencialesInvalidas = "Usuario o contraseña incorrectos.";   // ❌ constante en el Service

    public async Task<ApiResponse<LoginResponseDto>> LoginAsync(LoginDto dto)
    {
        var usuario = (await _unitOfWork.Usuarios.LoginAsync(dto.username.Trim())).Data; // ❌ normaliza la entrada
        if (usuario == null)
            return ApiResponse<LoginResponseDto>.Fail(CredencialesInvalidas);
        if (usuario.habilitado == false)
            return ApiResponse<LoginResponseDto>.Fail("Usuario deshabilitado.");            // ❌ texto literal
        var perfiles = await PerfilesDelUsuarioAsync(usuario.usuario_id);                   // ❌ método que no es de la interfaz
        return ApiResponse<LoginResponseDto>.Ok(new LoginResponseDto                        // ❌ arma el DTO
        {
            username = usuario.username,
            perfiles = perfiles
        });
    }

    private async Task<List<PerfilLoginDto>> PerfilesDelUsuarioAsync(int usuarioId)        // ❌ método privado
    {
        var result = await _unitOfWork.Perfiles.GetByUsuarioAsync(usuarioId);
        return result.Data!.Select(p => new PerfilLoginDto                                  // ❌ transforma datos
        {
            perfil_id = _encryptionService.Encrypt(p.perfil_id),
            descripcion = p.descripcion
        }).ToList();
    }
}
```

El mismo código **bien**, repartido en sus cuatro lugares. Cada archivo tiene una sola responsabilidad:

```csharp
// 1) Application/Common/Mensajes/MensajesSeg.cs — textos (uno por esquema; Mensajes.cs para los transversales)
namespace {Prefijo}.Application.Common;

public static class MensajesSeg
{
    public const string CredencialesInvalidas = "Usuario o contraseña incorrectos.";
    public const string UsuarioDeshabilitado = "Usuario deshabilitado.";
    public const string CredencialesValidas = "Credenciales válidas.";
    public const string ErrorIniciarSesion = "No se pudo iniciar sesión. Inténtalo de nuevo.";

    // Un mensaje con datos es un método que arma el texto
    public static string PerfilNoDisponible(string perfil) => $"El perfil {perfil} ya no está disponible.";
}
```

```csharp
// 2) Application/Helpers/seg/AuthMapeo.cs — transformaciones (entidad → DTO), sin acceso a datos
namespace {Prefijo}.Application.Helpers;

public static class AuthMapeo
{
    public static List<PerfilLoginDto> PerfilesLogin(IEnumerable<Perfil> perfiles, IIdEncryptionService enc) =>
        perfiles.Select(p => new PerfilLoginDto
        {
            perfil_id = enc.Encrypt(p.perfil_id),
            descripcion = p.descripcion
        }).ToList();

    public static LoginResponseDto LoginResponse(Usuario usuario, List<PerfilLoginDto> perfiles) => new()
    {
        username = usuario.username,
        perfiles = perfiles
    };
}
```

```csharp
// 3) Application/Helpers/seg/AuthHelper.cs — pasos de apoyo (consultan y arman), reciben todo por parámetro
namespace {Prefijo}.Application.Helpers;

public static class AuthHelper
{
    public static async Task<List<PerfilLoginDto>> PerfilesDisponiblesAsync(IUnitOfWork unitOfWork, int usuarioId, IIdEncryptionService enc)
    {
        var result = await unitOfWork.Perfiles.GetByUsuarioAsync(usuarioId);
        result.LanzarSiEsErrorTecnico();
        return AuthMapeo.PerfilesLogin(result.Data ?? [], enc);
    }
}
```

```csharp
// 4) Application/Services/seg/AuthService.cs — solo los métodos de IAuthService, solo decisiones de negocio
public class AuthService : IAuthService
{
    public async Task<ApiResponse<LoginResponseDto>> LoginAsync(LoginDto dto)   // dto.username ya llega recortado (setter del DTO)
    {
        try
        {
            var result = await _unitOfWork.Usuarios.LoginAsync(dto.username);
            result.LanzarSiEsErrorTecnico();

            var usuario = result.Data;
            if (usuario == null)
                return ApiResponse<LoginResponseDto>.Fail(MensajesSeg.CredencialesInvalidas);
            if (usuario.habilitado == false)
                return ApiResponse<LoginResponseDto>.Fail(MensajesSeg.UsuarioDeshabilitado);

            var perfiles = await AuthHelper.PerfilesDisponiblesAsync(_unitOfWork, usuario.usuario_id, _encryptionService);
            return ApiResponse<LoginResponseDto>.Ok(AuthMapeo.LoginResponse(usuario, perfiles), MensajesSeg.CredencialesValidas);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al iniciar sesión.");   // único texto permitido: el del log
            return ApiResponse<LoginResponseDto>.Fail(MensajesSeg.ErrorIniciarSesion);
        }
    }
}
```

Cómo decidir dónde va cada pieza (en este orden, la primera que aplique):

1. ¿Es un texto o un valor fijo? → `Mensajes{Core}` (o la clase de constantes de `Common/`).
2. ¿Convierte datos que ya tiene, sin consultar nada (DTO, árbol, página, filtro, JSON)? → `{Entity}Mapeo`.
3. ¿Consulta o arma algo para ayudar a un método de negocio, pero el controller no lo llama? → `{Entity}Helper`.
4. ¿Lo llama el controller? → es un método del Service y está en su interfaz.

Si un Service del proyecto ya tiene código en la forma de "antes", se refactoriza a la de "después" antes de
agregarle algo: la prueba de arquitectura no deja pasar ninguno de los dos estados mezclados.

## Service (Application/Services/{Core}/)

Orden fijo: validar → mapear → asignar auditoría desde JWT → (des)encriptar
ID → repo → `ApiResponse`. Try/catch envuelve cada método (única capa que
captura excepciones).

```csharp
namespace {Prefijo}.Application.Services;

public class {Entity}Service : I{Entity}Service
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMapper _mapper;
    private readonly IValidatorService _validatorService;
    private readonly IAppLogger<{Entity}Service> _logger;
    private readonly IIdEncryptionService _encryptionService;
    private readonly ICurrentUserService _currentUser;

    public {Entity}Service(
        IUnitOfWork unitOfWork, IMapper mapper, IValidatorService validatorService,
        IAppLogger<{Entity}Service> logger, IIdEncryptionService encryptionService,
        ICurrentUserService currentUser)
    {
        _unitOfWork = unitOfWork;
        _mapper = mapper;
        _validatorService = validatorService;
        _logger = logger;
        _encryptionService = encryptionService;
        _currentUser = currentUser;
    }

    // Lista todos los {entity} activos
    public async Task<ApiResponse<IEnumerable<{Entity}ResponseDto>>> GetAllAsync()
    {
        try
        {
            var result = await _unitOfWork.{Entity}s.GetAllAsync();
            if (result.Error)
                return ApiResponse<IEnumerable<{Entity}ResponseDto>>.Fail(result.Msg);

            return ApiResponse<IEnumerable<{Entity}ResponseDto>>.Ok(result.Data!.Select(x => {Entity}Mapeo.MapToResponseDto(x, _encryptionService)));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al obtener la lista de {entity}.");
            return ApiResponse<IEnumerable<{Entity}ResponseDto>>.Fail(Mensajes.ErrorObtenerLista);
        }
    }

    // Obtiene un {entity} por su ID encriptado
    public async Task<ApiResponse<{Entity}ResponseDto>> GetByIdAsync(string encrypted{Entity}Id)
    {
        try
        {
            if (!_encryptionService.TryDecrypt(encrypted{Entity}Id, out var {entity}Id))
                return ApiResponse<{Entity}ResponseDto>.Fail(Mensajes.IdentificadorInvalido);

            var result = await _unitOfWork.{Entity}s.GetByIdAsync({entity}Id);
            if (result.Error)
                return ApiResponse<{Entity}ResponseDto>.Fail(result.Msg);

            return ApiResponse<{Entity}ResponseDto>.Ok({Entity}Mapeo.MapToResponseDto(result.Data!, _encryptionService));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al obtener el {entity}.");
            return ApiResponse<{Entity}ResponseDto>.Fail(Mensajes.ErrorObtenerRegistro);
        }
    }

    // Valida e inserta un nuevo {entity}
    public async Task<ApiResponse<bool>> InsertAsync({Entity}InsDto dto)
    {
        try
        {
            var validation = await _validatorService.ValidateAsync(dto);
            if (validation.Error)
                return ApiResponse<bool>.Fail(validation.Msg);

            var {entity} = _mapper.Map<{Entity}>(dto);
            {entity}.usuario_reg = _currentUser.Username;
            var result = await _unitOfWork.{Entity}s.InsertAsync({entity});
            if (result.Error)
                return ApiResponse<bool>.Fail(result.Msg);

            return ApiResponse<bool>.Ok(true, result.Msg);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al insertar el {entity}.");
            return ApiResponse<bool>.Fail(Mensajes.ErrorInsertarRegistro);
        }
    }

    // Valida y actualiza un {entity} existente
    public async Task<ApiResponse<bool>> UpdateAsync({Entity}UpdDto dto)
    {
        try
        {
            var validation = await _validatorService.ValidateAsync(dto);
            if (validation.Error)
                return ApiResponse<bool>.Fail(validation.Msg);

            if (!_encryptionService.TryDecrypt(dto.{entity}_id, out var {entity}Id))
                return ApiResponse<bool>.Fail(Mensajes.IdentificadorInvalido);

            var {entity} = _mapper.Map<{Entity}>(dto);
            {entity}.{entity}_id = {entity}Id;
            {entity}.usuario_act = _currentUser.Username;
            var result = await _unitOfWork.{Entity}s.UpdateAsync({entity});
            if (result.Error)
                return ApiResponse<bool>.Fail(result.Msg);

            return ApiResponse<bool>.Ok(true, result.Msg);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al actualizar el {entity}.");
            return ApiResponse<bool>.Fail(Mensajes.ErrorActualizarRegistro);
        }
    }

    // Valida y elimina logicamente un {entity}
    public async Task<ApiResponse<bool>> DeleteAsync({Entity}DelDto dto)
    {
        try
        {
            var validation = await _validatorService.ValidateAsync(dto);
            if (validation.Error)
                return ApiResponse<bool>.Fail(validation.Msg);

            if (!_encryptionService.TryDecrypt(dto.{entity}_id, out var {entity}Id))
                return ApiResponse<bool>.Fail(Mensajes.IdentificadorInvalido);

            var {entity} = _mapper.Map<{Entity}>(dto);
            {entity}.{entity}_id = {entity}Id;
            {entity}.usuario_act = _currentUser.Username;
            var result = await _unitOfWork.{Entity}s.DeleteAsync({entity});
            if (result.Error)
                return ApiResponse<bool>.Fail(result.Msg);

            return ApiResponse<bool>.Ok(true, result.Msg);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al eliminar el {entity}.");
            return ApiResponse<bool>.Fail(Mensajes.ErrorEliminarRegistro);
        }
    }

}
```

Los mensajes genéricos (`Mensajes.IdentificadorInvalido`, `Mensajes.Error*Registro`) viven en
`Application/Common/Mensajes/Mensajes.cs`; los propios de un esquema, en `Mensajes{Core}.cs`.

## Helper de transformaciones (Application/Helpers/{Core}/{Entity}Mapeo.cs)

```csharp
namespace {Prefijo}.Application.Helpers;

// Transformaciones de datos de {Entity} que no son logica de negocio
public static class {Entity}Mapeo
{
    // Mapea la entidad a su DTO de respuesta, encriptando el ID
    public static {Entity}ResponseDto MapToResponseDto({Entity} {entity}, IIdEncryptionService enc) => new()
    {
        {entity}_id = enc.Encrypt({entity}.{entity}_id),
        {campo_negocio_1} = {entity}.{campo_negocio_1},
        {campo_negocio_2} = {entity}.{campo_negocio_2},
        estado = {entity}.estado,
        usuario_reg = {entity}.usuario_reg,
        fecha_reg = {entity}.fecha_reg,
        usuario_act = {entity}.usuario_act,
        fecha_act = {entity}.fecha_act
    };
}
```

FK nullable: `enc.EncryptNullable(id)` — nunca un método propio `EncryptNullableId`. El helper es estático y recibe
el servicio de cifrado como parámetro: el Service no cambia su constructor y las pruebas unitarias siguen simulando
`IIdEncryptionService`.

## Listas y estructuras anidadas hacia/desde un SP (JSON)

Cuando un SP recibe una lista o una estructura anidada (patrón multi-tabla
de `sql-database-patterns`: parámetro `NVARCHAR(MAX)` leído con
`OPENJSON`) o devuelve una columna armada con `FOR JSON PATH`, el
mecanismo vive **una sola vez** en `Application/Helpers/SpJson.cs`; la
**forma** de cada caso (records) y su lectura/armado van en el helper
`{Entity}Mapeo` de la entidad, nunca en el Service.

```csharp
// Application/Helpers/SpJson.cs
public static class SpJson
{
    public static string Serialize<T>(T value) => JsonSerializer.Serialize(value);

    // Una columna FOR JSON sin filas llega como NULL: se lee como lista vacia
    public static List<T> ReadList<T>(string? json)
        => string.IsNullOrWhiteSpace(json) ? [] : JsonSerializer.Deserialize<List<T>>(json) ?? [];
}
```

```csharp
// En Application/Helpers/seg/PerfilMapeo.cs: entrada (IDs encriptados -> JSON de enteros) y salida (JSON del SP ->
// IDs encriptados). El Service solo invoca: PerfilMapeo.TryDecryptAccesos(dto.accesos, out var json, _encryptionService)
public static bool TryDecryptAccesos(IEnumerable<PerfilAccesoDto> accesos, out string accesosJson, IIdEncryptionService enc)
{
    accesosJson = "[]";
    var lista = new List<AccesoEntrada>();
    foreach (var acceso in accesos)
    {
        var permisoIds = enc.TryDecryptArray(acceso.permiso_ids);
        if (!enc.TryDecrypt(acceso.opcion_id, out var opcionId) || permisoIds is null)
            return false;
        lista.Add(new AccesoEntrada(opcionId, permisoIds));
    }
    accesosJson = SpJson.Serialize(lista);
    return true;
}

public static List<PerfilAccesoDto> MapAccesos(string? accesosJson, IIdEncryptionService enc)
    => SpJson.ReadList<AccesoSalida>(accesosJson)
        .Select(a => new PerfilAccesoDto
        {
            opcion_id = enc.Encrypt(a.opcion_id),
            permiso_ids = (a.permisos ?? []).Select(p => enc.Encrypt(p.permiso_id)).ToList()
        })
        .ToList();

// Formas propias del caso (snake_case = nombres de columna/propiedad del JSON del SP), dentro del helper
public sealed record AccesoEntrada(int opcion_id, int[] permiso_ids);
public sealed record AccesoSalida(int opcion_id, List<PermisoSalida>? permisos);
public sealed record PermisoSalida(int permiso_id);
```

- La entidad lleva el JSON como `string?` (ej. `accesos`) y el
  Repository lo pasa como `DbType.String, size: -1`.
- Los repetidos se resuelven en el SP (`SELECT DISTINCT ... FROM
  OPENJSON`); el Service no necesita deduplicar.
- Nunca `JsonSerializer` directo en un Service ni en un helper de entidad: siempre `SpJson`.

## Registro en `Application/DependencyInjection.cs`

```csharp
services.AddScoped<I{Entity}Service, {Entity}Service>();
```

## Transacción multi-paso (obligatoria cuando hay más de un paso todo-o-nada)

Cuando un método de Application ejecuta más de un paso donde o se cumplen
todos o no se cumple ninguno (ej. crear una entidad principal + una entidad
relacionada en la misma operación), envuelve esos pasos en
`_unitOfWork.BeginTransactionAsync()` con un **try/catch interno** propio,
distinto del try/catch externo del método:

- Cada paso que falla (`result.Error`) hace `RollbackAsync()` y retorna
  `Fail` de inmediato — no sigue al siguiente paso.
- El `catch` interno hace `RollbackAsync()` y **re-lanza** (`throw;`) — deja
  que el try/catch externo del método (el que ya loguea y responde en todo
  Service) sea el que finalmente maneja la excepción.
- Tras el `CommitAsync()`, pasos adicionales que no son parte de la
  atomicidad (ej. generar datos derivados, notificar) van **fuera** de la
  transacción, como "fire-and-forget": método estático `TryXAsync(...)` en el helper
  `{Entity}Helper` (nunca un método privado del Service), con su propio try/catch que solo loguea
  `LogWarning` — nunca hacen fallar la operación principal ni se re-lanzan.

Ejemplo genérico (equivalente a un caso real: crear un `Pedido` con una
`PedidoLinea` embebida en la misma llamada):

```csharp
public async Task<ApiResponse<bool>> InsertAsync(PedidoInsDto dto)
{
    try
    {
        var validation = await _validatorService.ValidateAsync(dto);
        if (validation.Error)
            return ApiResponse<bool>.Fail(validation.Msg);

        var pedido = _mapper.Map<Pedido>(dto);
        pedido.usuario_reg = _currentUser.Username;

        // Alta con linea embebida → requiere transaccion (todo o nada)
        if (dto.linea != null)
        {
            var linea = _mapper.Map<PedidoLinea>(dto.linea);
            linea.usuario_reg = _currentUser.Username;

            await _unitOfWork.BeginTransactionAsync();
            try
            {
                var result = await _unitOfWork.Pedidos.InsertAsync(pedido);
                if (result.Error)
                {
                    await _unitOfWork.RollbackAsync();
                    return ApiResponse<bool>.Fail(result.Msg);
                }

                var newPedidoId = result.Data;
                linea.pedido_id = newPedidoId;

                var lineaResult = await _unitOfWork.PedidoLineas.InsertAsync(linea);
                if (lineaResult.Error)
                {
                    await _unitOfWork.RollbackAsync();
                    return ApiResponse<bool>.Fail(lineaResult.Msg);
                }

                await _unitOfWork.CommitAsync();

                // Fuera de la transaccion: efecto derivado, no bloqueante
                await PedidoHelper.TryNotificarPedidoCreadoAsync(_unitOfWork, _logger, newPedidoId);

                return ApiResponse<bool>.Ok(true, result.Msg);
            }
            catch
            {
                await _unitOfWork.RollbackAsync();
                throw; // lo maneja el catch externo del metodo
            }
        }
        else
        {
            // Flujo simple sin transaccion (sin linea embebida)
            var result = await _unitOfWork.Pedidos.InsertAsync(pedido);
            if (result.Error)
                return ApiResponse<bool>.Fail(result.Msg);

            return ApiResponse<bool>.Ok(true, result.Msg);
        }
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error al insertar el pedido.");
        return ApiResponse<bool>.Fail(Mensajes.ErrorInsertarRegistro);
    }
}

// Application/Helpers/ven/PedidoHelper.cs — fire-and-forget: nunca falla la operacion principal, solo advierte
public static class PedidoHelper
{
    public static async Task TryNotificarPedidoCreadoAsync(IUnitOfWork unitOfWork, IAppLogger<PedidoService> logger, int pedidoId)
    {
        try
        {
            var result = await unitOfWork.Notificaciones.NotificarPedidoAsync(pedidoId);
            if (result.Error)
                logger.LogWarning("No se pudo notificar el pedido {Id}: {Msg}", pedidoId, result.Msg);
        }
        catch (Exception ex)
        {
            logger.LogWarning("Error al notificar el pedido {Id}: {Error}", pedidoId, ex.Message);
        }
    }
}
```
