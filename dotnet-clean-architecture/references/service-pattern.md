# Service Pattern

Sustituye `{Entity}`/`{Core}` por los valores concretos.

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

            return ApiResponse<IEnumerable<{Entity}ResponseDto>>.Ok(result.Data!.Select(MapToResponseDto));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al obtener la lista de {entity}.");
            return ApiResponse<IEnumerable<{Entity}ResponseDto>>.Fail("Ocurrió un error al obtener la lista.");
        }
    }

    // Obtiene un {entity} por su ID encriptado
    public async Task<ApiResponse<{Entity}ResponseDto>> GetByIdAsync(string encrypted{Entity}Id)
    {
        try
        {
            if (!_encryptionService.TryDecrypt(encrypted{Entity}Id, out var {entity}Id))
                return ApiResponse<{Entity}ResponseDto>.Fail("El identificador proporcionado no es válido.");

            var result = await _unitOfWork.{Entity}s.GetByIdAsync({entity}Id);
            if (result.Error)
                return ApiResponse<{Entity}ResponseDto>.Fail(result.Msg);

            return ApiResponse<{Entity}ResponseDto>.Ok(MapToResponseDto(result.Data!));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al obtener el {entity}.");
            return ApiResponse<{Entity}ResponseDto>.Fail("Ocurrió un error al obtener el registro.");
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
            return ApiResponse<bool>.Fail("Ocurrió un error al insertar el registro.");
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
                return ApiResponse<bool>.Fail("El identificador proporcionado no es válido.");

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
            return ApiResponse<bool>.Fail("Ocurrió un error al actualizar el registro.");
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
                return ApiResponse<bool>.Fail("El identificador proporcionado no es válido.");

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
            return ApiResponse<bool>.Fail("Ocurrió un error al eliminar el registro.");
        }
    }

    // Mapea la entidad a su DTO de respuesta, encriptando el ID
    private {Entity}ResponseDto MapToResponseDto({Entity} {entity})
    {
        return new {Entity}ResponseDto
        {
            {entity}_id = _encryptionService.Encrypt({entity}.{entity}_id),
            {campo_negocio_1} = {entity}.{campo_negocio_1},
            {campo_negocio_2} = {entity}.{campo_negocio_2},
            estado = {entity}.estado,
            usuario_reg = {entity}.usuario_reg,
            fecha_reg = {entity}.fecha_reg,
            usuario_act = {entity}.usuario_act,
            fecha_act = {entity}.fecha_act
        };
    }
}
```

FK nullable en `MapToResponseDto`: usar `_encryptionService.EncryptNullable(id)`
— nunca un método privado `EncryptNullableId`.

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
  transacción, como "fire-and-forget": método `private async Task TryXAsync(...)`
  con su propio try/catch que solo loguea `LogWarning` — nunca hacen fallar
  la operación principal ni se re-lanzan.

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
                await TryNotificarPedidoCreadoAsync(newPedidoId);

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
        return ApiResponse<bool>.Fail("Ocurrió un error al insertar el registro.");
    }
}

// Fire-and-forget: nunca falla la operacion principal, solo advierte
private async Task TryNotificarPedidoCreadoAsync(int pedidoId)
{
    try
    {
        var result = await _unitOfWork.Notificaciones.NotificarPedidoAsync(pedidoId);
        if (result.Error)
            _logger.LogWarning("No se pudo notificar el pedido {Id}: {Msg}", pedidoId, result.Msg);
    }
    catch (Exception ex)
    {
        _logger.LogWarning("Error al notificar el pedido {Id}: {Error}", pedidoId, ex.Message);
    }
}
```
