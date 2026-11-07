# Testing

Toda funcionalidad se entrega con sus pruebas en verde. Las pruebas no se
escriben para "subir cobertura": se derivan de la especificación (flujos,
validaciones, restricciones y casos de prueba) y cada una provoca un
comportamiento concreto y verifica el resultado exacto que ve el cliente.

Verificado en un backend real (.NET 10) el 2026-11-06.

## Dos proyectos de prueba

| Proyecto | Qué prueba | Contra qué |
|---|---|---|
| `tests/{Prefijo}.UnitTests` | Services de Application (cada rama: validación, ID inválido, rechazo del SP, excepción) y validadores de FluentValidation | Dependencias simuladas con NSubstitute; sin BD ni HTTP |
| `tests/{Prefijo}.IntegrationTests` | Cada endpoint de punta a punta: controller → validador → service → repository → **SP real** | La API en memoria (`WebApplicationFactory<Program>`) y una **BD de pruebas** armada con los scripts del repo |

**Por qué los dos:** las reglas de negocio finales viven en los stored
procedures (ver `sql-database-patterns`). Una prueba unitaria con el
repositorio simulado solo demuestra que el Service *propaga* el mensaje del
SP, no que el SP *detecte* el caso. Por eso **toda validación del SP se
provoca en una prueba de integración**: se manda a propósito el valor que
la viola (un duplicado, un registro en uso, un ID que no es del tipo
esperado) y se espera el texto exacto del `@msg`. Las unitarias cubren lo
que la integración no alcanza con facilidad (la rama del `catch`, un ID
manipulado, la asignación de auditoría).

En disco viven fuera de `src/`, en una Solution Folder `tests` del `.sln`:

```
{api}/
  global.json                      → runner de pruebas (ver abajo)
  src/{Prefijo}.*                  → los 10 proyectos
  tests/
    {Prefijo}.UnitTests/
      Application/Services/{Core}/{Entidad}ServiceTests.cs
      Validator/{Core}/{Entidad}/{Entidad}InsValidatorTests.cs
    {Prefijo}.IntegrationTests/
      Infraestructura/              → AssemblyInfo, ApiPruebas, BdPruebas
      Api/{Core}/{Entidad}/{Entidad}sApiTests.cs
```

```bash
dotnet sln {Prefijo}.sln add --solution-folder tests tests/{Prefijo}.UnitTests/{Prefijo}.UnitTests.csproj
```

## Paquetes y runner

| Paquete | Proyecto | Nota |
|---|---|---|
| `xunit.v3.mtp-v2` | ambos | xUnit v3 sobre Microsoft Testing Platform |
| `NSubstitute` | UnitTests | dobles de prueba |
| `Microsoft.AspNetCore.Mvc.Testing` | IntegrationTests | `WebApplicationFactory`; misma versión mayor que .NET |

Siempre la última versión estable, previo gate de `package-security-vetting`.
`FluentValidation.TestHelper` ya viene en el paquete de FluentValidation.

Microsoft Testing Platform es el runner por defecto de `dotnet test` desde
.NET 10; se fija en `global.json` junto al `.sln`:

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

Los `.csproj` de prueba llevan `<OutputType>Exe</OutputType>`,
`<IsPackable>false</IsPackable>` y `<Using Include="Xunit" />`. UnitTests
referencia `Application` y `Validator`; IntegrationTests referencia `WebApi`.

En `Program.cs` se agrega al final, para `WebApplicationFactory<Program>`:

```csharp
// Visible para las pruebas de integracion (WebApplicationFactory<Program>)
public partial class Program;
```

## BD de pruebas

- Se llama `{bd}_test` y **se borra y se vuelve a crear en cada corrida**
  ejecutando los scripts del repo en orden de despliegue: `schemas.sql`
  (desde `master`, porque crea la BD), `tables.sql`, `functions/`,
  `types/`, `procedures/` y los seeds. Así las pruebas corren contra los SPs
  tal como están en el código, y de paso se comprueba que el repo
  reconstruye la BD desde cero.
- Los scripts se ejecutan con `Microsoft.Data.SqlClient` partidos por `GO`,
  reemplazando el nombre de la BD de desarrollo (`USE {bd};`) por el de
  pruebas. Los seeds con dependencias entre sí se ordenan explícitamente.
- La cadena de conexión sale de los **user-secrets de la API**
  (`AddUserSecrets<Program>()`, o la variable de entorno
  `ConnectionStrings__DefaultConnection` en CI) cambiando solo el
  `InitialCatalog`. Nunca se escribe una contraseña en el proyecto de prueba.
- **Nunca** se apunta a la BD de desarrollo: los datos de prueba solo
  existen en `{bd}_test`.
- Las pruebas comparten la BD y corren **en serie**
  (`[assembly: Parallelization(Mode = ParallelMode.None)]`). Cada prueba
  arma sus propios datos con valores que no choquen (ej. el siguiente número
  libre leído de la BD), en vez de depender del orden de ejecución.

```csharp
// Una sola API y una sola BD de pruebas para toda la corrida
[assembly: AssemblyFixture(typeof(ApiPruebas))]
[assembly: Parallelization(Mode = ParallelMode.None)]
```

```csharp
public class ApiPruebas : WebApplicationFactory<Program>, IAsyncLifetime
{
    public async ValueTask InitializeAsync() => await BdPruebas.RecrearAsync();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Development");
        builder.UseSetting("ConnectionStrings:DefaultConnection", BdPruebas.CadenaConexion);
    }

    // Cliente con JWT firmado por el propio IJwtTokenService de la API (usuario del seed, sin login)
    public HttpClient ClienteAutenticado() { ... }

    public new async ValueTask DisposeAsync() => await base.DisposeAsync();
}
```

`BdPruebas` expone `RecrearAsync()`, `CadenaConexion` y
`EscalarAsync<T>(sql)` para preparar datos o verificar lo que quedó
guardado.

## Qué se prueba por entidad (CRUD)

**Integración, por endpoint** (`{Entidad}sApiTests`):

1. Camino feliz de cada operación: estado 200, el `Message` exacto del SP y
   lo que quedó en la BD (incluida la auditoría `usuario_reg`/`usuario_act`).
2. **Cada `@msg` de rechazo del SP**, provocado con datos reales: duplicado,
   registro en uso al eliminar, FK a un registro inactivo o de otro tipo,
   regla de negocio propia. Estado 400 y el texto exacto.
3. Cada regla de FluentValidation, al menos una vez por endpoint (estado 400
   y su mensaje), para demostrar que el validador está registrado y corre
   antes de la BD.
4. Request sin token → 401.

**Unitarias:**

- `{Entidad}ServiceTests`: validación fallida (no llama al repositorio), ID
  encriptado inválido (`Mensajes.IdentificadorInvalido`), rechazo del SP
  propagado tal cual, excepción → mensaje genérico de `Mensajes` y
  `LogError`, auditoría tomada de `ICurrentUserService`.
- `{Entidad}{Op}ValidatorTests` con `TestValidate`: cada regla con un valor
  válido en el límite y uno inválido, verificando el mensaje.

## Prueba de arquitectura de Services (obligatoria en todo proyecto)

Garantiza las reglas duras "el Service solo expone los métodos de su interfaz" y "el Service solo tiene lógica de
negocio" (`references/service-pattern.md`): falla si un Service tiene un método que no es de su interfaz, declara constantes o campos estáticos, tipos internos, textos literales fuera del log, arma DTOs o JSON, o
tiene métodos privados de transformación. Va en `tests/{Prefijo}.UnitTests/Arquitectura/ServiciosSoloNegocioTests.cs`,
**copiada tal cual** de este bloque (solo se reemplazan `{Prefijo}` y `{AlgunService}`, cualquier Service existente;
no se reescribe ni se resume: es la versión que ya corre en verde en un proyecto real)
y corre con `dotnet test` (y en el hook `pre-push` si el proyecto lo tiene). Se crea al empezar el proyecto, o antes
de tocar el primer Service si aún no existe. Si falla, se mueve el código a su lugar; nunca se relaja la prueba.

```csharp
using System.Reflection;
using System.Text.RegularExpressions;
using {Prefijo}.Application.Services;

namespace {Prefijo}.UnitTests.Arquitectura;

// Reglas de arquitectura (skill dotnet-clean-architecture, y la regla del proyecto que la replique, si existe): un
// Service solo expone los metodos que usa el controller (los de su interfaz, ej. GetAll, GetById, Insert, Update y
// Delete en un CRUD) y solo contiene logica de negocio (condicionales, flujos, invocaciones a repositorios, validadores,
// mapeos y helpers). Constantes y mensajes van en Common/Mensajes; transformaciones de datos, mecanismos y pasos de
// apoyo, en Helpers o en el Mapper. Si una de estas pruebas falla, se mueve el codigo a su lugar; nunca se relaja.
public partial class ServiciosSoloNegocioTests
{
    private static readonly Type[] Servicios = typeof({AlgunService}).Assembly.GetTypes()
        .Where(t => t.IsClass && t.Namespace == typeof({AlgunService}).Namespace && t.Name.EndsWith("Service"))
        .ToArray();

    // Codigo fuente de los Services (Application/Services), ubicado subiendo desde la carpeta de salida de las pruebas
    private static readonly string[] Fuentes = BuscarFuentes();

    private static string[] BuscarFuentes()
    {
        for (var dir = new DirectoryInfo(AppContext.BaseDirectory); dir is not null; dir = dir.Parent)
        {
            var carpeta = Path.Combine(dir.FullName, "src", "{Prefijo}.Application", "Services");
            if (Directory.Exists(carpeta)) return Directory.GetFiles(carpeta, "*Service.cs", SearchOption.AllDirectories);
        }
        throw new DirectoryNotFoundException("No se encontro src/{Prefijo}.Application/Services subiendo desde " + AppContext.BaseDirectory);
    }

    private static string Nombre(string archivo) => Path.GetFileNameWithoutExtension(archivo);

    // Lineas de codigo (sin comentarios) que cumplen el patron, con su archivo y numero
    private static IEnumerable<string> Lineas(Regex patron, Func<string, bool>? exceptuar = null) =>
        Fuentes.SelectMany(f => File.ReadAllLines(f)
            .Select((linea, i) => (linea, i))
            .Where(x => !x.linea.TrimStart().StartsWith("//") && patron.IsMatch(x.linea) && !(exceptuar?.Invoke(x.linea) ?? false))
            .Select(x => $"{Nombre(f)}:{x.i + 1}: {x.linea.Trim()}"));

    [Fact]
    public void Se_encuentran_los_Services_a_revisar()
    {
        Assert.NotEmpty(Servicios);
        Assert.Equal(Servicios.Length, Fuentes.Length);
    }

    [Fact]
    public void Cada_Service_solo_tiene_los_metodos_de_su_interfaz()
    {
        var sobrantes = Servicios.SelectMany(t =>
        {
            var deLaInterfaz = t.GetInterfaces().SelectMany(i => t.GetInterfaceMap(i).TargetMethods).ToHashSet();
            return t.GetMethods(BindingFlags.Instance | BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.DeclaredOnly)
                .Where(m => !m.IsSpecialName && !m.Name.StartsWith('<')
                    && !m.IsDefined(typeof(System.Runtime.CompilerServices.CompilerGeneratedAttribute)))
                .Where(m => !deLaInterfaz.Contains(m))
                .Select(m => $"{t.Name}.{m.Name}");
        });

        Assert.Empty(sobrantes);
    }

    [Fact]
    public void Ningun_Service_declara_constantes_ni_campos_estaticos()
    {
        var campos = Servicios.SelectMany(t => t
            .GetFields(BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.DeclaredOnly)
            .Where(f => !f.IsDefined(typeof(System.Runtime.CompilerServices.CompilerGeneratedAttribute)))
            .Select(f => $"{t.Name}.{f.Name}"));

        Assert.Empty(campos);
    }

    [Fact]
    public void Ningun_Service_declara_tipos_internos()
    {
        var tipos = Servicios.SelectMany(t => t.GetNestedTypes(BindingFlags.Public | BindingFlags.NonPublic)
            .Where(n => !n.IsDefined(typeof(System.Runtime.CompilerServices.CompilerGeneratedAttribute)))
            .Select(n => $"{t.Name}.{n.Name}"));

        Assert.Empty(tipos);
    }

    [Fact]
    public void Ningun_Service_escribe_textos_literales_salvo_en_el_log()
    {
        Assert.Empty(Lineas(Literal(), linea => linea.Contains("_logger.")));
    }

    [Fact]
    public void Ningun_Service_arma_DTOs_ni_serializa_JSON()
    {
        Assert.Empty(Lineas(ArmaDtoOJson()));
    }

    [Fact]
    public void Ningun_Service_tiene_metodos_privados_de_transformacion()
    {
        Assert.Empty(Lineas(MetodoDeTransformacion()));
    }

    [GeneratedRegex("\"(?:[^\"\\\\]|\\\\.)*\"|'(?:[^'\\\\]|\\\\.)'")]
    private static partial Regex Literal();

    [GeneratedRegex(@"\bnew\s+[A-Z]\w*Dto\b|\bnew\s+[A-Z]\w*Dto<|JsonSerializer|JsonDocument|\bnew\s*\(\s*\)\s*\{")]
    private static partial Regex ArmaDtoOJson();

    [GeneratedRegex(@"\bprivate\s+(?:static\s+)?[\w<>\[\],?\s]+\s+(Map|To|Armar|Construir|Convertir|Leer|Serializar|Formatear|Normalizar|TryDecrypt|Asignar)\w*\s*\(")]
    private static partial Regex MetodoDeTransformacion();
}
```

## Prueba de arquitectura de repositorios, controllers y servicios externos (obligatoria en todo proyecto)

Garantiza las reglas duras de `references/repository-pattern.md`, `references/controller-pattern.md` y
`references/security-patterns.md` (servicios externos): repositorio, `UnitOfWork` y servicio externo solo con los
métodos de su interfaz; controller solo con endpoints `[Http*]`; ninguno con constantes, campos estáticos ni tipos
internos; textos solo los permitidos (nombre del SP y `@parámetros` en el repositorio, atributos en el controller,
ninguno en un servicio externo); y sin el mecanismo que va en los helpers (`SpEjecutor`, `RespuestaHttp`,
`ClaimsUsuario`, `JwtEmisor`). Va en `tests/{Prefijo}.UnitTests/Arquitectura/CapasSoloSuResponsabilidadTests.cs`,
**copiada tal cual** de este bloque: solo se reemplazan `{Prefijo}`, `{AlgunRepository}` y `{AlgunController}`
(cualquiera existente). El proyecto de pruebas unitarias referencia `Persistence`, `Logging` y `WebApi` para poder
revisarlos por reflexión. Se crea junto con la de Services.

**Cómo comprobar que la prueba funciona (una vez, al crearla):** mete a propósito una violación en cada capa (un
método privado en un repositorio, un `?? "x"` en un endpoint, un texto en `JwtTokenService`, un `new SpResult()` en
un repositorio), corre `dotnet test` y verifica que falla nombrando el archivo y la línea; luego deshaz la violación.

```csharp
using System.Reflection;
using System.Runtime.CompilerServices;
using System.Text.RegularExpressions;
using {Prefijo}.Logging;
using {Prefijo}.Persistence;
using {Prefijo}.Persistence.Repositories;
using {Prefijo}.Security.Jwt;
using {Prefijo}.WebApi.Controllers;
using {Prefijo}.WebApi.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Routing;

namespace {Prefijo}.UnitTests.Arquitectura;

// Reglas de arquitectura (skill dotnet-clean-architecture, y la regla del proyecto que la replique, si existe):
// - Repositorio y UnitOfWork: solo los metodos de su interfaz; cada metodo declara el SP y sus parametros de negocio y
//   delega el mecanismo (@error/@msg, ejecucion, SpResult, valores por defecto) en Persistence/Helpers.
// - Controller: solo endpoints (metodos con [Http*]) que delegan en su Service; la respuesta HTTP y la lectura de
//   claims van en WebApi/Helpers, y los DTOs los arma el model binding.
// - Servicio externo (Security, Logging, WebApi/Services): solo los metodos de su interfaz de Application; claims,
//   firmas, textos y pasos de apoyo van en Helpers o en Application/Common.
// Ninguno declara constantes, campos estaticos ni tipos internos. Si una prueba falla, se mueve el codigo a su lugar;
// nunca se relaja la prueba.
public partial class CapasSoloSuResponsabilidadTests
{
    private const BindingFlags Declarados =
        BindingFlags.Instance | BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.DeclaredOnly;

    // Repositorios: las clases del namespace de los repositorios, mas el UnitOfWork
    private static readonly Type[] Repositorios = typeof({AlgunRepository}).Assembly.GetTypes()
        .Where(t => t.IsClass && t.Namespace == typeof({AlgunRepository}).Namespace && t.Name.EndsWith("Repository"))
        .ToArray();

    private static readonly Type[] Controllers = typeof({AlgunController}).Assembly.GetTypes()
        .Where(t => t.IsClass && typeof(ControllerBase).IsAssignableFrom(t))
        .ToArray();

    // Servicios externos: clases de Security, Logging y WebApi/Services que implementan una interfaz de Application
    private static readonly Type[] ServiciosExternos = new[]
        {
            typeof(JwtTokenService).Assembly, typeof(AppLogger<>).Assembly, typeof(CurrentUserService).Assembly
        }
        .SelectMany(a => a.GetTypes())
        .Where(t => t.IsClass && !t.IsAbstract && !t.IsDefined(typeof(CompilerGeneratedAttribute))
            && t.Namespace != typeof({AlgunRepository}).Namespace
            && t.GetInterfaces().Any(i => i.Namespace == typeof({Prefijo}.Application.Interfaces.IUnitOfWork).Namespace))
        .Where(t => t.Assembly != typeof(CurrentUserService).Assembly || t.Namespace == typeof(CurrentUserService).Namespace)
        .Distinct()
        .ToArray();

    private static readonly string Src = BuscarSrc();

    // Carpeta src/ de la solucion, ubicada subiendo desde la carpeta de salida de las pruebas
    private static string BuscarSrc()
    {
        for (var dir = new DirectoryInfo(AppContext.BaseDirectory); dir is not null; dir = dir.Parent)
        {
            var src = Path.Combine(dir.FullName, "src");
            if (Directory.Exists(Path.Combine(src, "{Prefijo}.Persistence"))) return src;
        }
        throw new DirectoryNotFoundException("No se encontro src/ subiendo desde " + AppContext.BaseDirectory);
    }

    // Archivo fuente de un tipo: se busca por su nombre (sin la aridad de los genericos)
    private static string Fuente(Type t)
    {
        var nombre = t.Name.Split('`')[0] + ".cs";
        var archivos = Directory.GetFiles(Src, nombre, SearchOption.AllDirectories)
            .Where(f => !f.Contains($"{Path.DirectorySeparatorChar}bin{Path.DirectorySeparatorChar}")
                && !f.Contains($"{Path.DirectorySeparatorChar}obj{Path.DirectorySeparatorChar}"))
            .ToArray();
        Assert.True(archivos.Length == 1, $"Se esperaba un solo archivo {nombre} en src/ y hay {archivos.Length}");
        return archivos[0];
    }

    // Lineas de codigo (sin comentarios) de los tipos que cumplen el patron, con su archivo y numero
    private static IEnumerable<string> Lineas(IEnumerable<Type> tipos, Regex patron, Func<string, bool>? exceptuar = null) =>
        tipos.Select(Fuente).Distinct().SelectMany(f => File.ReadAllLines(f)
            .Select((linea, i) => (linea, i))
            .Where(x => !x.linea.TrimStart().StartsWith("//") && patron.IsMatch(x.linea) && !(exceptuar?.Invoke(x.linea) ?? false))
            .Select(x => $"{Path.GetFileNameWithoutExtension(f)}:{x.i + 1}: {x.linea.Trim()}"));

    // Metodos declarados en el tipo (sin propiedades, eventos ni lo que genera el compilador)
    private static IEnumerable<MethodInfo> Metodos(Type t) => t.GetMethods(Declarados)
        .Where(m => !m.IsSpecialName && !m.Name.StartsWith('<') && !m.IsDefined(typeof(CompilerGeneratedAttribute)));

    // Metodos que no implementan ninguna interfaz del tipo
    private static IEnumerable<string> FueraDeSuInterfaz(Type tipo)
    {
        var t = tipo.IsGenericTypeDefinition ? tipo.MakeGenericType(tipo.GetGenericArguments().Select(_ => typeof(object)).ToArray()) : tipo;
        var deLaInterfaz = t.GetInterfaces().SelectMany(i => t.GetInterfaceMap(i).TargetMethods).ToHashSet();
        return Metodos(t).Where(m => !deLaInterfaz.Contains(m)).Select(m => $"{t.Name}.{m.Name}");
    }

    private static IEnumerable<string> CamposEstaticos(IEnumerable<Type> tipos) => tipos.SelectMany(t => t
        .GetFields(BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.DeclaredOnly)
        .Where(f => !f.IsDefined(typeof(CompilerGeneratedAttribute)))
        .Select(f => $"{t.Name}.{f.Name}"));

    private static IEnumerable<string> TiposInternos(IEnumerable<Type> tipos) => tipos.SelectMany(t => t
        .GetNestedTypes(BindingFlags.Public | BindingFlags.NonPublic)
        .Where(n => !n.IsDefined(typeof(CompilerGeneratedAttribute)))
        .Select(n => $"{t.Name}.{n.Name}"));

    [Fact]
    public void Se_encuentran_las_clases_a_revisar()
    {
        Assert.NotEmpty(Repositorios);
        Assert.NotEmpty(Controllers);
        Assert.Contains(typeof(JwtTokenService), ServiciosExternos);
        Assert.Contains(typeof(CurrentUserService), ServiciosExternos);
        Assert.Contains(typeof(AppLogger<>), ServiciosExternos);
    }

    // ---------- Repositorios y UnitOfWork ----------

    [Fact]
    public void Cada_repositorio_y_el_UnitOfWork_solo_tienen_los_metodos_de_su_interfaz() =>
        Assert.Empty(Repositorios.Append(typeof(UnitOfWork)).SelectMany(FueraDeSuInterfaz));

    [Fact]
    public void Ningun_repositorio_ni_el_UnitOfWork_declara_constantes_campos_estaticos_ni_tipos_internos()
    {
        var tipos = Repositorios.Append(typeof(UnitOfWork)).ToArray();
        Assert.Empty(CamposEstaticos(tipos).Concat(TiposInternos(tipos)));
    }

    // Unicos textos de un repositorio: el nombre del SP ("esquema.Sp_Accion_Entidad") y sus parametros ("@campo")
    [Fact]
    public void Los_repositorios_solo_escriben_el_nombre_del_SP_y_de_sus_parametros() =>
        Assert.Empty(Repositorios.Append(typeof(UnitOfWork)).Select(Fuente).Distinct().SelectMany(f => File.ReadAllLines(f)
            .Select((linea, i) => (linea, i))
            .Where(x => !x.linea.TrimStart().StartsWith("//"))
            .SelectMany(x => Literal().Matches(x.linea).Select(m => (m.Value, x.i)))
            .Where(x => !LiteralDeSp().IsMatch(x.Value))
            .Select(x => $"{Path.GetFileNameWithoutExtension(f)}:{x.i + 1}: {x.Value}")));

    // El mecanismo (salidas @error/@msg, ejecucion, SpResult, valores por defecto, ID de entrada y salida) va en
    // Persistence/Helpers/SpEjecutor
    [Fact]
    public void Ningun_repositorio_implementa_el_mecanismo_de_ejecucion_del_SP() =>
        Assert.Empty(Lineas(Repositorios, MecanismoDeSp()));

    // ---------- Controllers ----------

    [Fact]
    public void Cada_controller_solo_tiene_endpoints() =>
        Assert.Empty(Controllers.SelectMany(t => Metodos(t)
            .Where(m => !m.IsPublic || !m.GetCustomAttributes<HttpMethodAttribute>(inherit: true).Any())
            .Select(m => $"{t.Name}.{m.Name}")));

    [Fact]
    public void Ningun_controller_declara_constantes_campos_estaticos_ni_tipos_internos() =>
        Assert.Empty(CamposEstaticos(Controllers).Concat(TiposInternos(Controllers)));

    // Unicos textos de un controller: los de sus atributos (rutas, esquemas)
    [Fact]
    public void Los_controllers_solo_escriben_textos_en_sus_atributos() =>
        Assert.Empty(Lineas(Controllers, Literal(), linea => linea.TrimStart().StartsWith('[')));

    // Un endpoint delega en su Service: no arma DTOs, no decide la respuesta HTTP ni lee claims (WebApi/Helpers)
    [Fact]
    public void Ningun_controller_implementa_mecanismos() =>
        Assert.Empty(Lineas(Controllers, MecanismoDeController()));

    // ---------- Servicios externos ----------

    [Fact]
    public void Cada_servicio_externo_solo_tiene_los_metodos_de_su_interfaz() =>
        Assert.Empty(ServiciosExternos.SelectMany(FueraDeSuInterfaz));

    [Fact]
    public void Ningun_servicio_externo_declara_constantes_campos_estaticos_ni_tipos_internos() =>
        Assert.Empty(CamposEstaticos(ServiciosExternos).Concat(TiposInternos(ServiciosExternos)));

    // Claims, mensajes y nombres van en Application/Common (ClaimsSesion, MensajesTecnicos) o en Helpers
    [Fact]
    public void Ningun_servicio_externo_escribe_textos_literales() =>
        Assert.Empty(Lineas(ServiciosExternos, Literal()));

    [GeneratedRegex("\"(?:[^\"\\\\]|\\\\.)*\"|'(?:[^'\\\\]|\\\\.)'")]
    private static partial Regex Literal();

    [GeneratedRegex(@"^""(?:[a-z]+\.Sp_\w+|@\w+)""$")]
    private static partial Regex LiteralDeSp();

    [GeneratedRegex(@"new\s+SpResult|new\s+DynamicParameters|ParameterDirection|CommandType|\.Get<|\?\?|\s\?\s|_connection\.(?:Query|Execute)")]
    private static partial Regex MecanismoDeSp();

    [GeneratedRegex(@"\bnew\s+\w|\?\?|\s\?\s|\bBadRequest\(|\bOk\(|\.Success\b|\bUser\.Find|\bHttpContext\b|\bvar\s")]
    private static partial Regex MecanismoDeController();
}
```

## Prueba de ubicación del proyecto Validator (obligatoria en todo proyecto)

Garantiza la regla dura de `references/validation-and-errors.md`, "Reglas reutilizables": `Validators/` solo con
validadores en `Validators/{Core}/{Entidad}/`, y cada regla en `Rules/{Core}/{Entidad o proceso}/` (cores = los de
`Validators/`) o en `Rules/Comun/`. Va en `tests/{Prefijo}.UnitTests/Arquitectura/ValidatorPorCarpetaDeNegocioTests.cs`,
**copiada tal cual** (solo se reemplaza `{Prefijo}`). Al crearla, compruébala: pon una clase de reglas en
`Validators/{Core}/{Entidad}/` y otra suelta en `Rules/`; `dotnet test` debe fallar nombrando las dos; luego bórralas.

```csharp
using FluentValidation;

namespace {Prefijo}.UnitTests.Arquitectura;

// Regla de arquitectura (skill dotnet-clean-architecture, references/validation-and-errors.md; regla del proyecto
// `_rules/codigo-escalable.md`): en el proyecto Validator, los validadores viven en Validators/{core}/{Entidad o
// proceso}/ y las reglas reutilizables en Rules/{core}/{Entidad o proceso}/, en la carpeta del negocio cuya logica
// expresan (no la de quien las consume ni la de la entidad que guarda el campo: la politica de password es de Auth).
// Una regla que no pertenece a ningun negocio (formato generico, ej. geolocalizacion) va en Rules/Comun/. Si una
// prueba falla, se mueve el archivo a su carpeta; nunca se relaja la prueba.
public class ValidatorPorCarpetaDeNegocioTests
{
    private const string CarpetaComun = "Comun";

    private static readonly string Raiz = BuscarRaiz();

    // Carpeta src/{Prefijo}.Validator, ubicada subiendo desde la carpeta de salida de las pruebas
    private static string BuscarRaiz()
    {
        var proyecto = typeof({Prefijo}.Validator.DependencyInjection).Assembly.GetName().Name!;
        for (var dir = new DirectoryInfo(AppContext.BaseDirectory); dir is not null; dir = dir.Parent)
        {
            var raiz = Path.Combine(dir.FullName, "src", proyecto);
            if (Directory.Exists(raiz)) return raiz;
        }
        throw new DirectoryNotFoundException("No se encontro src/ del proyecto Validator subiendo desde " + AppContext.BaseDirectory);
    }

    private static string Carpeta(string nombre) => Path.Combine(Raiz, nombre);

    // Ruta relativa de cada .cs bajo la carpeta, partida en segmentos (carpetas + archivo)
    private static IEnumerable<string[]> Archivos(string carpeta) =>
        Directory.GetFiles(Carpeta(carpeta), "*.cs", SearchOption.AllDirectories)
            .Select(f => Path.GetRelativePath(Carpeta(carpeta), f).Split(Path.DirectorySeparatorChar));

    // Carpetas de negocio (cores) que usa el proyecto: las de primer nivel de Validators/
    private static readonly HashSet<string> Cores =
        Directory.GetDirectories(Carpeta("Validators")).Select(Path.GetFileName).OfType<string>().ToHashSet();

    [Fact]
    public void Se_encuentran_las_carpetas_de_negocio()
    {
        Assert.NotEmpty(Cores);
        Assert.DoesNotContain(CarpetaComun, Cores);
    }

    [Fact]
    public void Cada_validador_esta_en_Validators_core_entidad() =>
        Assert.Empty(Archivos("Validators")
            .Where(s => s.Length != 3 || !Cores.Contains(s[0]))
            .Select(s => "Validators/" + string.Join('/', s)));

    // En Validators/ solo hay validadores: una clase de reglas ahi queda fuera de Rules/ y de su carpeta de negocio
    [Fact]
    public void En_Validators_solo_hay_validadores() =>
        Assert.Empty(Directory.GetFiles(Carpeta("Validators"), "*.cs", SearchOption.AllDirectories)
            .Where(f => !File.ReadAllText(f).Contains(": AbstractValidator<"))
            .Select(f => Path.GetRelativePath(Raiz, f)));

    [Fact]
    public void Cada_regla_esta_en_Rules_core_entidad_o_en_Rules_Comun() =>
        Assert.Empty(Archivos("Rules")
            .Where(s => !(s.Length == 3 && Cores.Contains(s[0])) && !(s.Length == 2 && s[0] == CarpetaComun))
            .Select(s => "Rules/" + string.Join('/', s)));

    // Cada validador concreto del ensamblado tiene su archivo (nombre del tipo) dentro de Validators/
    [Fact]
    public void Cada_validador_del_ensamblado_tiene_su_archivo_en_Validators()
    {
        var archivos = Archivos("Validators").Select(s => Path.GetFileNameWithoutExtension(s[^1])).ToHashSet();
        var validadores = typeof({Prefijo}.Validator.DependencyInjection).Assembly.GetTypes()
            .Where(t => t is { IsClass: true, IsAbstract: false } && t.BaseType is { IsGenericType: true }
                && t.BaseType.GetGenericTypeDefinition() == typeof(AbstractValidator<>));

        Assert.Empty(validadores.Where(t => !archivos.Contains(t.Name)).Select(t => t.Name));
    }
}
```

## Convenciones

- Una clase de prueba por clase probada; nombre `{Clase}Tests`.
- Métodos en el idioma de nomenclatura del proyecto, con guiones bajos,
  diciendo el comportamiento: `Rechaza_un_numero_de_sede_ya_usado`,
  `InsertAsync_oculta_el_error_tecnico_y_lo_registra_en_el_log`.
- Arrange / Act / Assert separados por una línea en blanco; un
  comportamiento por prueba.
- Los mensajes esperados se escriben **literales** en la prueba (copiados de
  la especificación o del SP), nunca leídos del código que se prueba: si
  alguien cambia el texto, la prueba debe fallar.
- **Trazabilidad:** si el proyecto especifica casos de prueba con ID (ej.
  `CP-07`) o validaciones numeradas (`V-03`), cada prueba que los cubre lo
  declara: `[Trait("CP", "CP-07")]`, `[Trait("V", "V-03")]`.
- Una prueba en rojo se arregla corrigiendo el **código**, no la prueba. La
  prueba solo se cambia si contradice la especificación (y entonces manda la
  especificación).

## Ejecutar

```bash
dotnet test                       # desde la carpeta del .sln: unitarias + integracion
dotnet test --project tests/{Prefijo}.UnitTests
```

Las de integración necesitan SQL Server accesible: si no responde, la
corrida falla (no se omite).
