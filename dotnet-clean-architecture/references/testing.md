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
