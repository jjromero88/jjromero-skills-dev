# Project Structure

## Los 10 proyectos — estructura física real (plana en disco)

```
src/
  {Prefijo}.Common/            # ApiResponse<T>
  {Prefijo}.Mapper/             # AutoMapper Profiles + Resolvers
  {Prefijo}.Logging/            # IAppLogger<T> / AppLogger<T>
  {Prefijo}.Security/           # Encryption, Hashing, Jwt
  {Prefijo}.Domain/             # Entities (cero dependencias)
  {Prefijo}.Infrastructure/     # Comunicación externa (cache, colas, otras APIs)
  {Prefijo}.Persistence/        # Context, Repositories, UnitOfWork
  {Prefijo}.Application/        # DTOs, Interfaces, Services, Common
  {Prefijo}.Validator/          # FluentValidation
  {Prefijo}.WebApi/             # Controllers, Program.cs
```

En disco todos son hermanos bajo `src/` — no hay carpetas físicas
`10.Transversal/`, `20.Domain/`, etc. Esa agrupación en 5 capas es **solo
visual**, vía Solution Folders de Visual Studio (ver sección siguiente).

Flujo de dependencias: `Domain ← Application ← {Persistence, Infrastructure,
Mapper, Validator, Logging} → WebApi`.

## Solution Folders (agrupación visual en 5 capas)

Las carpetas `10.Transversal`/`20.Domain`/`30.Infraestructura`/
`40.Application`/`50.Presentation` que se ven en Visual Studio son
**Solution Folders virtuales**, declaradas en el `.sln` — no existen como
carpetas en el disco. Se crean así:

```
# Un bloque Project/EndProject por Solution Folder, con el GUID de tipo
# Solution Folder ({2150E333-8FDC-42A3-9474-1A3956D46DE8}):
Project("{2150E333-8FDC-42A3-9474-1A3956D46DE8}") = "10.Transversal", "10.Transversal", "{GUID-A}"
EndProject
Project("{2150E333-8FDC-42A3-9474-1A3956D46DE8}") = "20.Domain", "20.Domain", "{GUID-B}"
EndProject
# ... una por cada capa

# Cada proyecto real ya tiene su propio bloque Project con el GUID de tipo
# C# ({FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}), generado por `dotnet new`/
# `dotnet sln add` — no se toca.

# Al final del .sln, dentro de Global, el bloque que ancla cada proyecto
# a su Solution Folder (hijo = padre):
GlobalSection(NestedProjects) = preSolution
    {GUID-del-proyecto-Domain} = {GUID-B}          # Domain → 20.Domain
    {GUID-del-proyecto-Application} = {GUID-D}     # Application → 40.Application
    {GUID-del-proyecto-Validator} = {GUID-D}       # Validator → 40.Application
    {GUID-del-proyecto-Persistence} = {GUID-C}     # Persistence → 30.Infraestructura
    {GUID-del-proyecto-Infrastructure} = {GUID-C}  # Infrastructure → 30.Infraestructura
    {GUID-del-proyecto-Mapper} = {GUID-A}          # Mapper → 10.Transversal
    {GUID-del-proyecto-Logging} = {GUID-A}         # Logging → 10.Transversal
    {GUID-del-proyecto-Common} = {GUID-A}          # Common → 10.Transversal
    {GUID-del-proyecto-Security} = {GUID-A}        # Security → 10.Transversal
    {GUID-del-proyecto-WebApi} = {GUID-E}          # WebApi → 50.Presentation
EndGlobalSection
```

Si `dotnet sln add` no soporta Solution Folders directamente en tu versión
de CLI, agrega los proyectos primero (planos) y luego edita el `.sln` a mano
con la estructura de arriba — Visual Studio la respeta al abrir la solución.

## Contenido por proyecto

| Proyecto | Contiene |
|---|---|
| `Common` | `Response/ApiResponse.cs` |
| `Mapper` | `Profiles/{Entidad}Profile.cs` (plano), `Resolvers/{X}Resolver.cs`, `DependencyInjection.cs` (`AddMapper()`) |
| `Logging` | `AppLogger.cs`, `DependencyInjection.cs` (`AddAppLogging()`) |
| `Security` | `Encryption/` (AES), `Hashing/` (BCrypt), `Jwt/`, `DependencyInjection.cs` (`AddSecurity(config)`) |
| `Domain` | `Entities/{Core}/{Entidad}.cs` heredando `AuditoriaBase` |
| `Infrastructure` | Comunicación externa: cache (Redis), colas (RabbitMQ), otras APIs/microservicios/WCF. `DependencyInjection.cs` (`AddInfrastructure()`) — vacío hasta que el proyecto integre algo externo |
| `Persistence` | `Context/DbConnectionFactory.cs`, `Repositories/{Core}/`, `UnitOfWork.cs`, `DependencyInjection.cs` (`AddPersistence(connString)`) |
| `Application` | `Common/` (`SpResult.cs` + clases `{Entidad}Results.cs` de resultsets múltiples), `DTOs/{Core}/`, `Helpers/`, `Interfaces/` (namespace único), `Services/{Core}/`, `Settings/`, `DependencyInjection.cs` (`AddApplication()`) |
| `Validator` | `Validators/{Core}/{Entidad}/`, `ValidatorService.cs`, `DependencyInjection.cs` (`AddValidator()`) |
| `WebApi` | `Controllers/`, `Services/CurrentUserService.cs`, `Program.cs`, `appsettings.json` |

`{Core}` = carpeta de negocio — ver skill `business-domain-grouping`.
`Infrastructure` es exclusivamente comunicación con sistemas externos;
`Persistence` es exclusivamente acceso a datos propio (Dapper + SPs) — no
mezclar responsabilidades entre ambos.

## `AuditoriaBase` (Domain)

Español (default):

```csharp
public abstract class AuditoriaBase
{
    public bool estado { get; set; }
    public string? usuario_reg { get; set; }
    public DateTime? fecha_reg { get; set; }
    public string? usuario_act { get; set; }
    public DateTime? fecha_act { get; set; }
}
```

Inglés (si el proyecto decidió nomenclatura en inglés — ver `SKILL.md`,
"Antes de crear"; misma equivalencia idiomática que
`sql-database-patterns`, no traducción literal):

```csharp
public abstract class AuditoriaBase
{
    public bool status { get; set; }
    public string? created_by { get; set; }
    public DateTime? created_at { get; set; }
    public string? updated_by { get; set; }
    public DateTime? updated_at { get; set; }
}
```

Las propiedades mantienen el mismo casing (`snake_case`, espejo exacto de
la columna SQL) en ambos idiomas — solo cambia la palabra, igual que en
`sql-database-patterns`.

## `Program.cs` (orden de registro)

```csharp
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection")
    ?? throw new InvalidOperationException("Connection string 'DefaultConnection' not found.");

builder.Services.AddApplication();
builder.Services.AddInfrastructure();
builder.Services.AddPersistence(connectionString);
builder.Services.AddMapper();
builder.Services.AddValidator();
builder.Services.AddAppLogging();
builder.Services.AddSecurity(builder.Configuration);

builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();

// CORS: whitelist desde appsettings.json ("Cors:AllowedOrigins")
var allowedOrigins = builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>() ?? [];
builder.Services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", policy =>
        policy.WithOrigins(allowedOrigins).AllowAnyHeader().AllowAnyMethod());
});

// JWT — ver security-patterns.md para el detalle completo
builder.Services.AddAuthentication(o => { /* JwtBearerDefaults... */ }).AddJwtBearer(/* ... */);
builder.Services.AddAuthorization();

builder.Services.AddControllers();
builder.Services.AddSwaggerGen(options =>
{
    // SecurityDefinition("Bearer") + SecurityRequirement — ver security-patterns.md
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
if (!app.Environment.IsDevelopment())
    app.UseHttpsRedirection();

app.UseCors("CorsPolicy");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

Orden de middleware crítico: `UseCors → UseAuthentication → UseAuthorization
→ MapControllers`.

## `DependencyInjection.cs` por proyecto

`Application` (registra cada Service — uno por entidad — como Scoped; motores
transversales como Singleton si aplica):

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<I{Entidad}Service, {Entidad}Service>();
        // ... un AddScoped por cada entidad
        return services;
    }
}
```

`Persistence` (mínimo — solo la connection factory y el UnitOfWork):

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddPersistence(this IServiceCollection services, string connectionString)
    {
        services.AddSingleton<IDbConnectionFactory>(new DbConnectionFactory(connectionString));
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        return services;
    }
}
```

`Infrastructure` (vacío hasta integrar algo externo; ejemplo cuando sí aplica):

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
    {
        // Registrar external services aqui — ej. cache distribuido:
        // services.AddStackExchangeRedisCache(options =>
        //     options.Configuration = configuration.GetConnectionString("Redis"));
        // services.AddScoped<ICacheService, RedisCacheService>();

        return services;
    }
}
```

## Paquetes NuGet

**Regla dura: siempre la última versión estable compatible con la versión
de .NET elegida** — nunca fijar una versión antigua por defecto. Las
versiones de abajo son solo la referencia observada en un proyecto real
(.NET 9) al momento de escribir esta skill; verifica la versión actual antes
de instalar.

| Paquete | Versión observada (.NET 9, referencia) | Proyecto |
|---|---|---|
| `Dapper` | 2.1.72 | Persistence |
| `Microsoft.Data.SqlClient` | 6.1.4 | Persistence |
| `FluentValidation.DependencyInjectionExtensions` | 12.1.1 | Validator |
| `AutoMapper` | 16.1.1 | Application, Mapper |
| `BCrypt.Net-Next` | 4.1.0 | Security |
| `System.IdentityModel.Tokens.Jwt` | 8.7.0 | Security |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | 9.0.3 | WebApi |
| `Swashbuckle.AspNetCore` | 6.9.0 | WebApi |
| `Microsoft.AspNetCore.OpenApi` | 9.0.13 | WebApi |
| `Microsoft.Extensions.{Configuration.Abstractions, Configuration.Binder, DependencyInjection.Abstractions, Logging, Options, Options.ConfigurationExtensions}` | 10.0.3 | según proyecto |

**Excepción real observada**: `Swashbuckle.AspNetCore` 10.x resultó
incompatible con .NET 9 al momento de este proyecto — si al instalar la
última versión de un paquete falla la compilación o hay warnings de
compatibilidad con la versión de .NET elegida, retrocede a la última versión
estable que sí sea compatible, y verifícalo antes de continuar.

No hay MediatR, no hay Entity Framework, no hay paquetes de testing (aún no
existen proyectos de test).
