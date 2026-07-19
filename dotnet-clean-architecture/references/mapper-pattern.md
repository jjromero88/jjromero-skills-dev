# Mapper Pattern

Sustituye `{Entity}` por el valor concreto.

## Profile (Mapper/Profiles/)

`MemberList.Source` porque los DTOs son subconjuntos de la entidad. El ID
se ignora en Upd/Del (el Service lo asigna manualmente tras desencriptar).
Plano por entidad — no lleva `{Core}`:

```csharp
namespace {Prefijo}.Mapper.Profiles;

public class {Entity}Profile : Profile
{
    public {Entity}Profile()
    {
        CreateMap<{Entity}InsDto, {Entity}>(MemberList.Source);
        CreateMap<{Entity}UpdDto, {Entity}>(MemberList.Source)
            .ForMember(dest => dest.{entity}_id, opt => opt.Ignore())
            .ForSourceMember(src => src.{entity}_id, opt => opt.DoNotValidate());
        CreateMap<{Entity}DelDto, {Entity}>(MemberList.Source)
            .ForMember(dest => dest.{entity}_id, opt => opt.Ignore())
            .ForSourceMember(src => src.{entity}_id, opt => opt.DoNotValidate());
    }
}
```

Registrar manualmente en `Mapper/DependencyInjection.cs`:
`cfg.AddProfile<{Entity}Profile>();` — no hay auto-scan de perfiles.

## Resolver (Mapper/Resolvers/)

Para desencriptar un ID directamente durante el mapeo (en vez de asignarlo
a mano en el Service) — reutilizable en cualquier Profile:

```csharp
namespace {Prefijo}.Mapper.Resolvers;

public class DecryptIdResolver : IMemberValueResolver<object, object, string?, int?>
{
    private readonly IIdEncryptionService _encryptionService;

    public DecryptIdResolver(IIdEncryptionService encryptionService)
    {
        _encryptionService = encryptionService;
    }

    // Desencripta el ID de origen; null si viene vacio o no es valido
    public int? Resolve(object source, object destination, string? sourceMember, int? destMember, ResolutionContext context)
    {
        if (string.IsNullOrWhiteSpace(sourceMember))
            return null;
        return _encryptionService.TryDecrypt(sourceMember, out var id) ? id : null;
    }
}
```

Registro en `Mapper/DependencyInjection.cs` (antes de `AddAutoMapper`):

```csharp
services.AddTransient<DecryptIdResolver>();
services.AddAutoMapper(cfg =>
{
    cfg.AddProfile<{Entity}Profile>();
    // ... un AddProfile por cada entidad
});
```
