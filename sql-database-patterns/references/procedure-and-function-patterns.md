# Procedures de proceso (`Sp_Acc_`/`Sp_Ver_`) y Functions (`fn_svf_`/`fn_tvf_`)

No todo procedure es CRUD. Para procesos de negocio (ejecutar nómina, generar
feriados, login) o verificaciones complejas, usa esta nomenclatura en vez de
`Sel/Get/Ins/Upd/Del`.

Todo bloque (validación o lógica) lleva su comentario corto en español,
incluidas las functions — ver `validation-patterns.md`.

## Regla de nombre

```
Sp_Acc_{Verbo}_{Entidad}
Sp_Ver_{Verbo}_{Entidad}
```

- **`Acc`** = acción principal que el frontend invoca (con o sin DML). Ej.:
  `Sp_Acc_Generar_Feriado` (muta), `Sp_Acc_Login_Usuario` (no muta, pero ES la acción).
- **`Ver`** = chequeo de apoyo, solo lectura, invocado antes o dentro de otro
  flujo. Ej.: `Sp_Ver_PreCalculo`, `Sp_Ver_Authorize_Usuario`.
- Diferencia semántica, no técnica — no depende de si hay DML.
- `{Verbo}_{Entidad}` son normalmente dos palabras. Excepción: concepto
  compuesto sin "Entidad" limpia va como una sola palabra (`Sp_Ver_PreCalculo`).

## Template `Sp_Acc_*` con mutación

`@usuario_reg`/`@usuario_act` si audita filas. `@msg` final suele reportar un
resultado cuantificado, no un mensaje fijo. Cabecera de documentación siempre
encima (igual que en `sp-templates.md`):

```sql
/*
==========================================================================
  Procedimiento  : {esquema}.Sp_Acc_{Verbo}_{Entidad}
  Descripcion    : [Descripción detallada en español]
  Autor          : {autor}
  Fecha creacion : YYYY-MM-DD
  ---------------------------------------------------------
  INPUTS:
    @param       DATATYPE     - Descripción

  OUTPUTS:
    @error       BIT          - 0: exito, 1: error
    @msg         VARCHAR(500) - Mensaje de resultado
==========================================================================
*/

CREATE OR ALTER PROCEDURE {esquema}.Sp_Acc_{Verbo}_{Entidad}
    @{parametro_negocio}   INT,
    @usuario_reg           VARCHAR(50),
    @error                 BIT            OUTPUT,
    @msg                   VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que {parametro_negocio} no sea nulo
        -- (agrega el resto de validaciones de negocio, ordenadas por dependencia)
        IF @{parametro_negocio} IS NULL
        BEGIN
            SET @error = 1;
            SET @msg   = 'El {parametro_negocio} es obligatorio.';
            RETURN;
        END

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Generamos/procesamos {Entidad} de forma idempotente
            DECLARE @generados INT = 0;
            -- ... lógica ...

            SET @msg = 'Se generaron ' + CAST(@generados AS VARCHAR) + ' registro(s).';

        END

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

## Template `Sp_Acc_*` sin mutación (ej. autenticación)

Sin `@usuario_reg`/`@usuario_act` porque no audita nada. Retorna `SELECT` solo si hay éxito:

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Acc_{Verbo}_{Entidad}
    @{credencial_1}   VARCHAR(50),
    @{credencial_2}   VARCHAR(200),
    @error            BIT            OUTPUT,
    @msg              VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el {credencial_1} no sea nulo o vacio
        IF @{credencial_1} IS NULL OR LTRIM(RTRIM(@{credencial_1})) = ''
        BEGIN
            SET @error = 1;
            SET @msg   = 'El campo {credencial_1} es obligatorio.';
            RETURN;
        END

        -- Validamos que el {entidad} exista
        IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {credencial_1} = @{credencial_1})
        BEGIN
            SET @error = 1;
            SET @msg   = 'El {entidad} ingresado no existe.';
            RETURN;
        END

        -- Validamos las credenciales (depende de la existencia validada arriba)
        IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {credencial_1} = @{credencial_1} AND {credencial_2} = @{credencial_2} AND estado = 1)
        BEGIN
            SET @error = 1;
            SET @msg   = 'Las credenciales son incorrectas.';
            RETURN;
        END

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Retornamos los datos del {entidad} tras validación exitosa
            SELECT {entity}_id, {campos_publicos}
            FROM {esquema}.{TABLA}
            WHERE {credencial_1} = @{credencial_1} AND estado = 1;

            SET @msg = '{Verbo} exitoso.';

        END

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

## Template `Sp_Ver_*` con múltiples resultsets

2-3 resultsets relacionados, consumidos con `QueryMultipleAsync` de Dapper:

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Ver_{Verbo}_{Entidad}
    @{param_1}   INT,
    @{param_2}   INT,
    @error       BIT            OUTPUT,
    @msg         VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el {entidad_1} exista y este activo
        IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA_1} WHERE {entity_1}_id = @{param_1} AND estado = 1)
        BEGIN
            SET @error = 1;
            SET @msg   = 'El {entidad_1} no existe o se encuentra inactivo.';
            RETURN;
        END

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Resultset 1: datos de {entidad_1}
            SELECT ... FROM {esquema}.{TABLA_1} WHERE {entity_1}_id = @{param_1};

            -- Resultset 2: datos relacionados de {entidad_2}
            SELECT ... FROM {esquema}.{TABLA_2} WHERE {entity_2}_id = @{param_2};

            -- Resultset 3: datos adicionales (si aplica)
            SELECT ... FROM {esquema}.{TABLA_3} ...;

            SET @msg = 'Verificación exitosa.';

        END

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

## Patrón de resultset de verificación genérico

Para chequeos que devuelven una lista de hallazgos en vez de operar datos.
`@tiene_errores` (hallazgo de negocio bloqueante) es distinto de `@error`
(el SP falló técnicamente):

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Ver_{Concepto}
    @{param}            INT,
    @tiene_errores      BIT            OUTPUT,
    @error              BIT            OUTPUT,
    @msg                VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @tiene_errores = 0;
    SET @error         = 0;
    SET @msg           = '';

    BEGIN TRY

        -- Tabla temporal para acumular los hallazgos
        CREATE TABLE #verificacion (
            tipo        VARCHAR(20)   NOT NULL,   -- GLOBAL | {ENTIDAD}
            nivel       VARCHAR(20)   NOT NULL,   -- ERROR | ADVERTENCIA
            categoria   VARCHAR(50)   NOT NULL,
            codigo      VARCHAR(50)   NOT NULL,   -- código máquina del check
            mensaje     VARCHAR(500)  NOT NULL,
            {entidad}_id INT          NULL,       -- NULL si es check global
            referencia  VARCHAR(200)  NULL
        );

        -- {CODIGO_CHECK}: {regla global que se verifica}
        -- (repetir por cada regla global)
        IF NOT EXISTS (...)
        INSERT INTO #verificacion VALUES ('GLOBAL', 'ERROR', '{categoria}', '{CODIGO_CHECK}', '{mensaje}', NULL, '{referencia}');

        -- {CODIGO_CHECK}: {regla por entidad que se verifica}
        INSERT INTO #verificacion
        SELECT 'GLOBAL', 'ERROR', '{categoria}', '{CODIGO_CHECK}', '{mensaje}', e.{entidad}_id, e.{campo_referencia}
        FROM {esquema}.{TABLA} e
        WHERE {condicion_de_falla};

        -- Determinamos si hay errores bloqueantes
        IF EXISTS (SELECT 1 FROM #verificacion WHERE nivel = 'ERROR')
            SET @tiene_errores = 1;

        -- Retornamos todos los hallazgos
        SELECT * FROM #verificacion ORDER BY tipo, categoria, {entidad}_id;

        SET @msg = CASE
            WHEN @tiene_errores = 1 THEN 'Verificación completada con errores.'
            ELSE 'Verificación completada correctamente.'
        END;

        DROP TABLE #verificacion;

    END TRY
    BEGIN CATCH

        IF OBJECT_ID('tempdb..#verificacion') IS NOT NULL
            DROP TABLE #verificacion;

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

## Patrón de deprecación

No se borra el archivo — cabecera marcada + cuerpo reducido a un error explicativo:

```sql
/*
==========================================================================
  Procedimiento  : {esquema}.Sp_{Acc|Ver}_{Verbo}_{Entidad}
  DEPRECADO      : YYYY-MM-DD
  Razon          : [Por qué ya no se usa y qué lo reemplaza]
  Autor          : {autor}
  Fecha creacion : YYYY-MM-DD
==========================================================================
*/

CREATE OR ALTER PROCEDURE {esquema}.Sp_{Acc|Ver}_{Verbo}_{Entidad}
    @{params_originales},
    @error    BIT            OUTPUT,
    @msg      VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @error = 1;
    SET @msg   = '{Nombre del SP} esta deprecado. [Qué usar en su lugar].';
END
GO
```

---

# Functions (`fn_svf_`/`fn_tvf_`)

Sin precedente real en el proyecto — convención nueva, consistente con las
restricciones de T-SQL.

## Regla de nombre

```
fn_svf_{Concepto}   → scalar value function (un solo valor)
fn_tvf_{Concepto}   → table valued function (conjunto de filas)
```

## Reglas duras (restricciones reales de SQL Server)

- Sin parámetros `OUTPUT` — no existe el contrato `@error`/`@msg`. Escalar
  inválido retorna `NULL`; TVF inválido retorna 0 filas. La validación real
  vive en el SP/Service que consume la función.
- Sin `THROW`/`RAISERROR`.
- Sin `INSERT`/`UPDATE`/`DELETE` contra tablas permanentes (solo contra
  `@Resultado` en un TVF multi-statement).
- No se puede invocar un stored procedure desde una function.
- Preferir **inline** (`RETURN (SELECT ...)`) sobre multi-statement — SQL
  Server la integra al plan de ejecución del llamador. Multi-statement solo
  si la lógica necesita bucles/variables intermedias.

## Template `fn_svf_` (scalar)

```sql
/*
==========================================================================
  Funcion        : {esquema}.fn_svf_{Concepto}
  Descripcion    : [Qué calcula. Indicar cuándo retorna NULL.]
  Autor          : {autor}
  Fecha creacion : YYYY-MM-DD
  ---------------------------------------------------------
  INPUTS:
    @param       DATATYPE     - Descripción
  RETURNS:
    {TIPO_RETORNO} - Descripción. NULL si {condición de input inválido}.
==========================================================================
*/
CREATE OR ALTER FUNCTION {esquema}.fn_svf_{Concepto} (@{param} {DATATYPE})
RETURNS {TIPO_RETORNO}
AS
BEGIN
    DECLARE @resultado {TIPO_RETORNO};

    -- Calculamos/obtenemos {Concepto} (retorna NULL si {condicion} no matchea)
    SELECT @resultado = {expresion_o_columna}
    FROM {esquema}.{TABLA}
    WHERE {condicion};

    RETURN @resultado;
END
GO
```

## Template `fn_tvf_` (inline — preferida)

```sql
/*
==========================================================================
  Funcion        : {esquema}.fn_tvf_{Concepto}
  Descripcion    : [Qué conjunto de filas retorna]
  Autor          : {autor}
  Fecha creacion : YYYY-MM-DD
  ---------------------------------------------------------
  INPUTS:
    @param       DATATYPE     - Descripción
  RETURNS:
    TABLE con columnas: {col1}, {col2}, ...
==========================================================================
*/
CREATE OR ALTER FUNCTION {esquema}.fn_tvf_{Concepto} (@{param} {DATATYPE})
RETURNS TABLE
AS
RETURN
(
    -- Retornamos {Concepto} filtrado por {param}, solo activos
    SELECT {col1}, {col2}, ...
    FROM {esquema}.{TABLA}
    WHERE {condicion} = @{param}
      AND estado = 1
);
GO
```

## Template `fn_tvf_` (multi-statement — solo si inline no alcanza)

```sql
CREATE OR ALTER FUNCTION {esquema}.fn_tvf_{Concepto} (@{param} {DATATYPE})
RETURNS @Resultado TABLE (
    {col1} {TIPO},
    {col2} {TIPO}
)
AS
BEGIN
    -- Calculamos/armamos {Concepto} (logica que un SELECT plano no expresa)
    INSERT INTO @Resultado ({col1}, {col2})
    SELECT {col1}, {col2}
    FROM {esquema}.{TABLA}
    WHERE {condicion} = @{param};

    RETURN;
END
GO
```
