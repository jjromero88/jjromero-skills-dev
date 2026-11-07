# Controller Pattern

Sustituye `{Entity}` por el valor concreto.

## Reglas de arquitectura del controller (reglas duras)

1. **El controller solo tiene endpoints del negocio**: cada método público lleva su `[Http*]` y delega en su Service
   en una sola expresión. Ningún otro método, ni privado.
2. **Sin mecanismos**: no decide el código HTTP (`this.Responder(...)` de `WebApi/Helpers/RespuestaHttp.cs`), no arma
   DTOs (los arma el model binding: `[FromRoute]`, `[FromQuery]`, `[FromBody]`), no lee claims ni `HttpContext`
   (extensiones de `ClaimsPrincipal` en `WebApi/Helpers/`, o `ICurrentUserService`), sin `var`, `??` ni ternarios.
3. **Únicos textos permitidos: los de sus atributos** (rutas, esquemas de autenticación). Sin constantes, campos
   estáticos ni tipos internos.

Lo garantiza la prueba de arquitectura `CapasSoloSuResponsabilidadTests` (`references/testing.md`).

| En el controller NO va | Va en |
|---|---|
| `return !result.Success ? BadRequest(result) : Ok(result);` | `this.Responder(await _service.XAsync(...))` |
| `var dto = new {Entity}DelDto { {entity}_id = id };` | `[HttpDelete("{{entity}_id}")] Delete([FromRoute] {Entity}DelDto dto)` (la URL no cambia) |
| `User.FindFirst(JwtRegisteredClaimNames.Sub)?.Value ?? string.Empty` | `User.UsuarioId()` (`WebApi/Helpers/ClaimsUsuario.cs`) |

## Helpers de WebApi (se crean una vez por proyecto, copiados tal cual)

```csharp
// WebApi/Helpers/RespuestaHttp.cs
using {Prefijo}.Common.Response;
using Microsoft.AspNetCore.Mvc;

namespace {Prefijo}.WebApi.Helpers;

// Mecanismo comun de todo endpoint: traduce la respuesta del Service a su codigo HTTP
public static class RespuestaHttp
{
    // 200 con la respuesta si tuvo exito; 400 con la misma respuesta (y su mensaje) si no
    public static IActionResult Responder<T>(this ControllerBase controller, ApiResponse<T> result) =>
        result.Success ? controller.Ok(result) : controller.BadRequest(result);
}
```

```csharp
// WebApi/Helpers/ClaimsUsuario.cs (solo si un endpoint necesita un claim que ICurrentUserService no da)
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;

namespace {Prefijo}.WebApi.Helpers;

public static class ClaimsUsuario
{
    // ID encriptado del usuario (claim sub); vacio si el token no lo trae
    public static string UsuarioId(this ClaimsPrincipal user) =>
        user.FindFirst(JwtRegisteredClaimNames.Sub)?.Value ?? string.Empty;
}
```

## Controller (WebApi/Controllers/)

Nombre con "s" pegada al final (`{Entity}sController`). Patrón idéntico en
los 5 endpoints — regla dura, no sugerencia:

```csharp
using {Prefijo}.Application.DTOs;
using {Prefijo}.Application.Interfaces;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using {Prefijo}.WebApi.Helpers;

namespace {Prefijo}.WebApi.Controllers;

[Authorize]
[ApiController]
[Route("api/[controller]")]
public class {Entity}sController : ControllerBase
{
    private readonly I{Entity}Service _{entity}Service;

    public {Entity}sController(I{Entity}Service {entity}Service)
    {
        _{entity}Service = {entity}Service;
    }

    [HttpGet]
    public async Task<IActionResult> GetAll() =>
        this.Responder(await _{entity}Service.GetAllAsync());

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(string id) =>
        this.Responder(await _{entity}Service.GetByIdAsync(id));

    [HttpPost]
    public async Task<IActionResult> Insert([FromBody] {Entity}InsDto dto) =>
        this.Responder(await _{entity}Service.InsertAsync(dto));

    [HttpPut]
    public async Task<IActionResult> Update([FromBody] {Entity}UpdDto dto) =>
        this.Responder(await _{entity}Service.UpdateAsync(dto));

    // El model binding arma el DTO desde la ruta: el nombre del segmento es el de la propiedad del DTO
    [HttpDelete("{{entity}_id}")]
    public async Task<IActionResult> Delete([FromRoute] {Entity}DelDto dto) =>
        this.Responder(await _{entity}Service.DeleteAsync(dto));
}
```
