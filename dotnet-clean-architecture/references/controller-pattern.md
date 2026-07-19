# Controller Pattern

Sustituye `{Entity}` por el valor concreto.

## Controller (WebApi/Controllers/)

Nombre con "s" pegada al final (`{Entity}sController`). Patrón idéntico en
los 5 endpoints — regla dura, no sugerencia:

```csharp
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
    public async Task<IActionResult> GetAll()
    {
        var result = await _{entity}Service.GetAllAsync();
        return !result.Success ? BadRequest(result) : Ok(result);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(string id)
    {
        var result = await _{entity}Service.GetByIdAsync(id);
        return !result.Success ? BadRequest(result) : Ok(result);
    }

    [HttpPost]
    public async Task<IActionResult> Insert([FromBody] {Entity}InsDto dto)
    {
        var result = await _{entity}Service.InsertAsync(dto);
        return !result.Success ? BadRequest(result) : Ok(result);
    }

    [HttpPut]
    public async Task<IActionResult> Update([FromBody] {Entity}UpdDto dto)
    {
        var result = await _{entity}Service.UpdateAsync(dto);
        return !result.Success ? BadRequest(result) : Ok(result);
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(string id)
    {
        var dto = new {Entity}DelDto { {entity}_id = id };
        var result = await _{entity}Service.DeleteAsync(dto);
        return !result.Success ? BadRequest(result) : Ok(result);
    }
}
```
