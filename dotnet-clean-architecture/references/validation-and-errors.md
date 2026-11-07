# Validation and Error Handling

## Dos tipos de resultado

- **`SpResult`/`SpResult<T>`** — interno, lo que retornan repositorios y
  `IValidatorService` (refleja outputs del SP o errores de validación).
- **`ApiResponse<T>`** — externo, lo que el Controller devuelve al cliente.

```csharp
// Application/Common/SpResult.cs
public class SpResult
{
    public bool Error { get; set; }
    public string Msg { get; set; } = string.Empty;
    public List<string>? Errors { get; set; }
}
public class SpResult<T> : SpResult
{
    public T? Data { get; set; }
}
```

```csharp
// Common/Response/ApiResponse.cs
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public string Message { get; set; } = string.Empty;
    public T? Data { get; set; }
    public IEnumerable<string>? Errors { get; set; }

    public static ApiResponse<T> Ok(T data, string message = "Operación exitosa.")
        => new() { Success = true, Data = data, Message = message };

    public static ApiResponse<T> Fail(string message, IEnumerable<string>? errors = null)
        => new() { Success = false, Message = message, Errors = errors };
}
```

## `IValidatorService` — resolución centralizada

Los Services inyectan **un solo** `IValidatorService`, nunca `IValidator<T>`
directamente:

```csharp
public class ValidatorService : IValidatorService
{
    private readonly IServiceProvider _serviceProvider;
    public ValidatorService(IServiceProvider serviceProvider) => _serviceProvider = serviceProvider;

    public async Task<SpResult> ValidateAsync<T>(T instance)
    {
        var validator = _serviceProvider.GetService<IValidator<T>>();
        if (validator == null)
            return new SpResult();

        var result = await validator.ValidateAsync(instance);
        if (!result.IsValid)
        {
            var errors = result.Errors.Select(e => e.ErrorMessage).ToList();
            return new SpResult { Error = true, Msg = errors.First(), Errors = errors };
        }
        return new SpResult();
    }
}
```

Sin validador registrado para `T` → `SpResult` vacío (sin error), no lanza
excepción.

## FluentValidation — un validador por DTO

`Validator/Validators/{Core}/{Entity}/{Entity}InsValidator.cs`:

```csharp
public class {Entity}InsValidator : AbstractValidator<{Entity}InsDto>
{
    public {Entity}InsValidator()
    {
        RuleFor(x => x.{campo_negocio_1})
            .NotEmpty().WithMessage("El campo {campo_negocio_1} es obligatorio.")
            .MaximumLength(100).WithMessage("El campo {campo_negocio_1} no debe exceder 100 caracteres.");

        RuleFor(x => x.{campo_negocio_2})
            .MaximumLength(200).WithMessage("El campo {campo_negocio_2} no debe exceder 200 caracteres.")
            .When(x => x.{campo_negocio_2} != null);
    }
}
```

Mensajes siempre en español, nombre de campo en `snake_case` tal cual la
propiedad. Un validador por operación (`Ins`/`Upd`/`Del`) — no reutilizar uno
solo para varias.

## Reglas reutilizables (`Validator/Rules/`) — dónde va cada una (regla dura)

Una regla que usan dos o más validadores (alta y edición, o dos entidades) se escribe una sola vez como extensión
estática en `Validator/Rules/`, **en la carpeta de negocio cuya lógica expresa**:

```
Validator/
  Validators/{Core}/{Entidad o proceso}/{Entidad}{Ins|Upd|Del}Validator.cs   ← solo validadores
  Rules/{Core}/{Entidad o proceso}/{Concepto}Rules.cs                        ← reglas de ese negocio
  Rules/Comun/{Concepto}Rules.cs                                             ← formato genérico, sin negocio
```

- **Decide la lógica de negocio que la regla expresa**, no quién la consume ni qué entidad guarda el campo. Ejemplo:
  la política de password la aplican el alta de usuario y el restablecimiento (Usuario) y el login usa su máximo,
  pero define qué es una credencial válida, que es negocio de **Auth** → `Rules/seg/Auth/PasswordRules.cs`. El
  nombre sigue a la carpeta: `PasswordRules`, no `UsuarioPasswordRules`.
- `{Core}/{Entidad o proceso}` son las mismas carpetas de `Validators/` (skill `business-domain-grouping`).
- `Rules/Comun/` solo para un formato que no pertenece a ningún negocio (ej. `GeolocalizacionRules`: latitud y
  longitud). Si tiene dueño de negocio, no va en `Comun`.
- `Validators/` solo contiene validadores (`: AbstractValidator<T>`); una clase de reglas ahí está mal ubicada.
- La carpeta no cambia el namespace: todas las reglas usan `{Prefijo}.Validator.Rules` (los validadores las importan
  con `using {Prefijo}.Validator.Rules;`).

La ubicación la garantiza la prueba `ValidatorPorCarpetaDeNegocioTests` (`references/testing.md`): falla si una
regla está fuera de `Rules/{Core}/{X}/` o `Rules/Comun/`, o si hay algo que no es validador en `Validators/`. Qué
negocio es dueño de la regla no lo puede decidir una prueba: se decide con el criterio de arriba al crearla.

## Regla de password segura (si la entidad maneja credenciales)

Mínimo 8, máximo 15, al menos una mayúscula, un número, y un carácter especial. Es negocio de Auth:
`Validator/Rules/{Core}/Auth/PasswordRules.cs`, y la usan los validadores que reciben una password nueva
(`RuleFor(x => x.password).PasswordSegura();`). El login solo valida el máximo
(`.MaximumLength(PasswordRules.MaximoCaracteres)`), para que una password válida de antes siga entrando.

```csharp
using FluentValidation;

namespace {Prefijo}.Validator.Rules;

// Politica de la credencial (negocio de Auth)
public static class PasswordRules
{
    public const int MaximoCaracteres = 15;

    public static IRuleBuilderOptions<T, string> PasswordSegura<T>(this IRuleBuilder<T, string> regla) => regla
        .NotEmpty().WithMessage("El campo password es obligatorio.")
        .MinimumLength(8).WithMessage("El campo password debe tener al menos 8 caracteres.")
        .MaximumLength(MaximoCaracteres).WithMessage($"El campo password no debe exceder {MaximoCaracteres} caracteres.")
        .Matches("[A-Z]").WithMessage("El campo password debe contener al menos una letra mayúscula.")
        .Matches("[0-9]").WithMessage("El campo password debe contener al menos un número.")
        .Matches(@"[!@#$%^&*()_+\-=\[\]{};':""\\|,.<>\/?]").WithMessage("El campo password debe contener al menos un carácter especial.");
}
```

## Registro (`Validator/DependencyInjection.cs`)

```csharp
public static IServiceCollection AddValidator(this IServiceCollection services)
{
    services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);
    services.AddScoped<IValidatorService, ValidatorService>();
    return services;
}
```

Auto-descubre todos los `IValidator<T>` del assembly — no hay que
registrarlos uno por uno.

## Try/catch solo en la capa Service

- **Repository**: sin try/catch, deja subir la excepción.
- **Service**: envuelve cada método, loguea con `IAppLogger<T>.LogError(ex,
  mensaje)`, retorna `ApiResponse<T>.Fail("mensaje amigable en español")`.
- **Controller**: no maneja excepciones — solo interpreta `result.Success`.

```csharp
try
{
    // validar → mapear → repo
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error al {operacion} el {entidad}.");
    return ApiResponse<T>.Fail("Ocurrió un error al {operacion} el registro.");
}
```
