# Security Patterns

## IDs encriptados

`IIdEncryptionService` (Application) → `AesIdEncryptionService` (Security,
AES-256-CBC, Base64 URL-safe):

```csharp
public interface IIdEncryptionService
{
    string Encrypt(int id);
    int Decrypt(string encryptedId);
    bool TryDecrypt(string encryptedId, out int id);
}

public static class IdEncryptionExtensions
{
    // FK nullable — usar SIEMPRE esto, nunca un metodo privado EncryptNullableId en el Service
    public static string? EncryptNullable(this IIdEncryptionService service, int? id)
        => id.HasValue ? service.Encrypt(id.Value) : null;

    // Decripta un array de IDs; null si alguno falla
    public static int[]? TryDecryptArray(this IIdEncryptionService service, string[] encryptedIds)
    {
        if (encryptedIds.Length == 0) return [];
        var ids = new int[encryptedIds.Length];
        for (int i = 0; i < encryptedIds.Length; i++)
        {
            if (!service.TryDecrypt(encryptedIds[i], out var id)) return null;
            ids[i] = id;
        }
        return ids;
    }
}
```

- Domain, Repository y BD siguen usando `int` — la conversión ocurre
  exclusivamente en el Service.
- DTOs de entrada (`UpdDto`/`DelDto`) reciben `string {entidad}_id`
  encriptado; DTOs de salida (`ResponseDto`) lo exponen igual.
- Config en `appsettings.json` bajo `Encryption` (`Key`/`IV` en Base64).

## Hashing de passwords

`IPasswordHasher` (Application) → `BCryptPasswordHasher` (Security, BCrypt.Net-Next):

```csharp
public interface IPasswordHasher
{
    string HashPassword(string password);
    bool VerifyPassword(string password, string hash);
}
```

- Hashing ocurre en el Service antes de llamar al repositorio (Insert/Update).
- Hash se almacena como `VARCHAR(200)` (BCrypt genera ~60 chars).
- Verificación (`BCrypt.Verify`) ocurre en C#, nunca en SQL — BCrypt genera
  un hash distinto en cada invocación, no se puede comparar por igualdad en BD.

## JWT

`IJwtTokenService` (Application) → `JwtTokenService` (Security, HMAC-SHA256,
`System.IdentityModel.Tokens.Jwt`):

- Claims: `sub` (id encriptado), `unique_name` (username), `email`, `jti` (GUID único).
- Config en `appsettings.json` sección `Jwt`: `SecretKey` (≥32 chars),
  `Issuer`, `Audience`, `ExpirationMinutes`.
- Registrado como Singleton en `Security/DependencyInjection.cs`.
- `Program.cs`: `AddAuthentication().AddJwtBearer()` con validación completa
  y `ClockSkew = TimeSpan.Zero`. `UseAuthentication()` antes de `UseAuthorization()`.
- Swagger: `SecurityDefinition("Bearer")` + `SecurityRequirement` para el
  botón Authorize.
- Controllers CRUD llevan `[Authorize]` a nivel de clase; endpoints públicos
  (login) llevan `[AllowAnonymous]`.

## Auditoría desde JWT (`ICurrentUserService`)

```csharp
public interface ICurrentUserService
{
    string Username { get; }
}
```

- Implementación en `WebApi/Services/CurrentUserService.cs`, lee
  `HttpContext.User.Identity.Name` (mapea al claim `unique_name`).
- Registrado Scoped, requiere `AddHttpContextAccessor()`.
- **Extensible**: si el negocio necesita otro dato de scoping del usuario
  autenticado además de `Username` (ej. la sucursal/empresa a la que
  pertenece), se agrega como otra propiedad de solo lectura leyendo un claim
  adicional del JWT:

```csharp
public interface ICurrentUserService
{
    string Username { get; }
    string {Scope}Id { get; }   // ej. SucursalId, EmpresaId — el que aplique al negocio
}

public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    public CurrentUserService(IHttpContextAccessor httpContextAccessor) => _httpContextAccessor = httpContextAccessor;

    public string Username =>
        _httpContextAccessor.HttpContext?.User?.Identity?.Name
        ?? throw new UnauthorizedAccessException("No se pudo obtener el usuario autenticado.");

    public string {Scope}Id =>
        _httpContextAccessor.HttpContext?.User?.FindFirst("{scope}_id")?.Value
        ?? throw new UnauthorizedAccessException("No se pudo obtener el {scope} del usuario autenticado.");
}
```

El claim adicional debe agregarse también en `JwtTokenService` al generar el
token de login.
- `usuario_reg`/`usuario_act` **nunca** vienen del cliente — el Service los
  asigna tras el mapeo: `entity.usuario_reg = _currentUser.Username;`
  (Insert), `entity.usuario_act = _currentUser.Username;` (Update/Delete).
- `AuditoriaInsDto`/`AuditoriaDelDto` quedan vacíos; `AuditoriaUpdDto` solo
  conserva `estado`. Los validadores no incluyen reglas para
  `usuario_reg`/`usuario_act`.
- `AuthService` NO usa `ICurrentUserService` — login es `[AllowAnonymous]`,
  no hay usuario autenticado todavía.

## Login con política de bloqueo (si el proyecto requiere auth completa)

Flujo real de referencia:
1. Validar `LoginDto` (username, password).
2. Obtener hash almacenado por username — si no existe, retornar
   "Credenciales inválidas" **sin registrar intento** (evita filtrar qué
   usernames existen).
3. Verificar bloqueo (intentos fallidos recientes vs. `MaxAttempts`/`LockoutMinutes`).
4. Verificar password con `BCrypt.Verify()` en C#.
5. Si falla: registrar intento fallido. Si excede `MaxAttempts`: bloquear
   por `LockoutMinutes`, con mensaje mostrando la hora exacta de desbloqueo.
6. Si tiene éxito: resetear intentos, generar JWT, retornar
   `LoginResponseDto` (id encriptado + token).

**Regla de bloqueo**: solo aplica a usuarios existentes — si el username no
existe, error genérico sin verificar bloqueo ni registrar intento. Config en
`appsettings.json` sección `LoginPolicy` (`MaxAttempts`, `LockoutMinutes`).
