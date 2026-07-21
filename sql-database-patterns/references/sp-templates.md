# Stored Procedure Templates

Sustituye `{Entity}`/`{entity}`/`{TABLA}`/`{esquema}` por valores concretos.
Salida siempre por `OUTPUT`, nunca `SELECT` final.

## Cabecera obligatoria (todo procedure)

```sql
/*
==========================================================================
  Procedimiento  : {esquema}.Sp_{Accion}_{Entity}
  Descripcion    : [Descripción detallada en español]
  Autor          : {autor}
  Fecha creacion : YYYY-MM-DD
  ---------------------------------------------------------
  INPUTS:
    @param       DATATYPE     - Descripción

  OUTPUTS:
    @{entity}_id INT          - ID del registro (solo en Sp_Ins)
    @error       BIT          - 0: exito, 1: error
    @msg         VARCHAR(500) - Mensaje de resultado
==========================================================================
*/
```

---

## Sp_Ins

Campos de negocio + `@usuario_reg`. Siempre retorna `@{entity}_id INT OUTPUT`.

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Ins_{Entity}
    @{campo_negocio_1}   VARCHAR(100),
    @{campo_negocio_2}   VARCHAR(200)   = NULL,
    @usuario_reg         VARCHAR(50),
    @{entity}_id         INT            OUTPUT,
    @error               BIT            OUTPUT,
    @msg                 VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @{entity}_id = 0;
    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el {campo_negocio_1} no sea nulo o vacio
        IF @{campo_negocio_1} IS NULL OR LTRIM(RTRIM(@{campo_negocio_1})) = ''
        BEGIN
            SET @error = 1;
            SET @msg   = 'El campo {campo_negocio_1} es obligatorio.';
            RETURN;
        END

        -- Validamos que el {entidad_padre} exista y este activo (si aplica — ver validation-patterns.md #2)

        -- Verificamos que el {campo_unico} no este asociado a otro {entity} activo (si aplica — ver validation-patterns.md #5)

        -- Agrega aqui las validaciones de negocio propias de {Entity}, ordenadas por dependencia

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Insertamos el {entity}
            INSERT INTO {esquema}.{TABLA} (
                {campo_negocio_1},
                {campo_negocio_2},
                usuario_reg,
                fecha_reg
            )
            VALUES (
                @{campo_negocio_1},
                @{campo_negocio_2},
                @usuario_reg,
                GETDATE()
            );

            SET @{entity}_id = SCOPE_IDENTITY();

            -- Si {TABLA} tiene columna `codigo`: autogenerar desde el PK, zero-pad a 5 digitos
            -- (default obligatorio — ver ddl-conventions.md; omitir solo si el usuario pidio otro formato)
            UPDATE {esquema}.{TABLA}
            SET codigo = RIGHT('00000' + CAST(@{entity}_id AS VARCHAR(5)), 5)
            WHERE {entity}_id = @{entity}_id;

            SET @msg = '{Entity} registrado correctamente.';

        END

    END TRY
    BEGIN CATCH

        SET @{entity}_id = 0;
        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

---

## Sp_Upd

`@{entity}_id` + campos de negocio + `@usuario_act`. Nunca toca
`usuario_reg`/`fecha_reg`. `estado` no es parámetro salvo que se pida
explícito. Tampoco `codigo` (si la tabla lo tiene) — es autogenerado solo
en `Sp_Ins` (ver `ddl-conventions.md`), nunca un parámetro de `Sp_Upd` ni
un campo editable por defecto.

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Upd_{Entity}
    @{entity}_id         INT,
    @{campo_negocio_1}   VARCHAR(100),
    @{campo_negocio_2}   VARCHAR(200)   = NULL,
    @usuario_act         VARCHAR(50),
    @error               BIT            OUTPUT,
    @msg                 VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el {entity} exista
        IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {entity}_id = @{entity}_id)
        BEGIN
            SET @error = 1;
            SET @msg   = 'El {entity} con el ID especificado no existe.';
            RETURN;
        END

        -- Validamos que el {campo_negocio_1} no sea nulo o vacio
        IF @{campo_negocio_1} IS NULL OR LTRIM(RTRIM(@{campo_negocio_1})) = ''
        BEGIN
            SET @error = 1;
            SET @msg   = 'El campo {campo_negocio_1} es obligatorio.';
            RETURN;
        END

        -- Verificamos que el {campo_unico} no este asociado a otro {entity} activo
        -- (excluyendo el propio registro, si aplica — ver validation-patterns.md #5)

        -- Agrega aqui otras validaciones de negocio de {Entity}, ordenadas por dependencia

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Actualizamos el {entity}
            UPDATE {esquema}.{TABLA}
            SET {campo_negocio_1} = @{campo_negocio_1},
                {campo_negocio_2} = @{campo_negocio_2},
                usuario_act       = @usuario_act,
                fecha_act         = GETDATE()
            WHERE {entity}_id = @{entity}_id;

            SET @msg = '{Entity} actualizado correctamente.';

        END

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

---

## Sp_Del

`@{entity}_id` + `@usuario_act`. Siempre eliminación lógica. Un bloque de
dependencia por cada tabla hija.

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Del_{Entity}
    @{entity}_id    INT,
    @usuario_act    VARCHAR(50),
    @error          BIT            OUTPUT,
    @msg            VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el {entity} exista
        IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {entity}_id = @{entity}_id)
        BEGIN
            SET @error = 1;
            SET @msg   = 'El {entity} con el ID especificado no existe.';
            RETURN;
        END

        -- Validamos que el {entity} no este ya inactivo
        IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {entity}_id = @{entity}_id AND estado = 1)
        BEGIN
            SET @error = 1;
            SET @msg   = 'El {entity} ya se encuentra inactivo.';
            RETURN;
        END

        -- Validamos que el {entity} no tenga {entidades_hijas} activas asociadas
        -- (repetir un bloque por cada tabla hija)
        IF EXISTS (SELECT 1 FROM {esquema}.{TABLA_HIJA} WHERE {entity}_id = @{entity}_id AND estado = 1)
        BEGIN
            SET @error = 1;
            SET @msg   = 'No se puede eliminar el {entity} porque está asociado a {entidades_hijas} activas.';
            RETURN;
        END

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Eliminamos logicamente el {entity}
            UPDATE {esquema}.{TABLA}
            SET estado      = 0,
                usuario_act = @usuario_act,
                fecha_act   = GETDATE()
            WHERE {entity}_id = @{entity}_id;

            SET @msg = '{Entity} eliminado correctamente.';

        END

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

---

## Sp_Sel

Sin filtros obligatorios (`= NULL` + `@param IS NULL OR columna = @param`).
Filtra `estado = 1`, ordena por PK.

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Sel_{Entity}
    @{filtro_opcional}   INT            = NULL,
    @error               BIT            OUTPUT,
    @msg                 VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Listamos los {entity} activos, aplicando el filtro opcional si vino
        SELECT  e.{entity}_id,
                e.{campo_negocio_1},
                e.{campo_negocio_2},
                e.estado,
                e.usuario_reg,
                e.fecha_reg,
                e.usuario_act,
                e.fecha_act
        FROM    {esquema}.{TABLA} e
        WHERE   e.estado = 1
            AND (@{filtro_opcional} IS NULL OR e.{filtro_opcional} = @{filtro_opcional})
        ORDER BY e.{entity}_id ASC;

        SET @msg = 'Consulta ejecutada correctamente.';

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

---

## Sp_Get

Un registro por ID. NO filtra `estado` (permite auditar inactivos).

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Get_{Entity}
    @{entity}_id  INT,
    @error        BIT            OUTPUT,
    @msg          VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el {entity} exista
        IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {entity}_id = @{entity}_id)
        BEGIN
            SET @error = 1;
            SET @msg   = 'El {entity} con el ID especificado no existe.';
            RETURN;
        END

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Obtenemos el {entity} por su ID (sin filtrar estado)
            SELECT  {entity}_id,
                    {campo_negocio_1},
                    {campo_negocio_2},
                    estado,
                    usuario_reg,
                    fecha_reg,
                    usuario_act,
                    fecha_act
            FROM    {esquema}.{TABLA}
            WHERE   {entity}_id = @{entity}_id;

            SET @msg = 'Consulta ejecutada correctamente.';

        END

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

---

## Variante: SP de negocio multi-tabla con transacción

Solo cuando un SP debe encapsular una operación atómica que toca varias
tablas (ej. `Sp_Ins_Perfil`: perfil + sociedades + opciones/permisos). CRUD
simple de una tabla NO usa transacción explícita — la atomicidad
multi-repositorio la maneja `IUnitOfWork` desde C#.

Reglas:
- Arrays/objetos anidados como `NVARCHAR(MAX)` + `ISJSON()` (validation-patterns.md #7).
- `OPENJSON`/`JSON_VALUE`/`CROSS APPLY` para volcar el JSON a tablas temporales.
- DML envuelto en `BEGIN TRANSACTION`/`COMMIT TRANSACTION`.
- `CATCH` hace `IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION` antes de setear `@error`/`@msg`.
- Retorna `@{entity}_id OUTPUT` igual que cualquier `Sp_Ins`.

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Ins_{Entity}
    @{campo_negocio}     VARCHAR(200),
    @{hijos}_ids         NVARCHAR(MAX),   -- JSON array, ej. '[1, 3, 5]'
    @usuario_reg         VARCHAR(50),
    @{entity}_id         INT            OUTPUT,
    @error               BIT            OUTPUT,
    @msg                 VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @{entity}_id = 0;
    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el {campo_negocio} no sea nulo o vacio
        IF @{campo_negocio} IS NULL OR LTRIM(RTRIM(@{campo_negocio})) = ''
        BEGIN
            SET @error = 1;
            SET @msg   = 'El campo {campo_negocio} es obligatorio.';
            RETURN;
        END

        -- Validamos que {hijos}_ids sea un JSON valido
        IF @{hijos}_ids IS NULL OR ISJSON(@{hijos}_ids) = 0
        BEGIN
            SET @error = 1;
            SET @msg   = 'El parámetro {hijos}_ids no es un JSON válido.';
            RETURN;
        END

        -- Volcamos el JSON de {hijos}_ids a una tabla temporal
        DECLARE @{Hijos} TABLE ({hijo}_id INT);
        INSERT INTO @{Hijos} ({hijo}_id)
        SELECT CAST(value AS INT) FROM OPENJSON(@{hijos}_ids);

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            BEGIN TRANSACTION;

                -- Insertamos el {entity}
                INSERT INTO {esquema}.{TABLA} ({campo_negocio}, usuario_reg, fecha_reg)
                VALUES (@{campo_negocio}, @usuario_reg, GETDATE());

                SET @{entity}_id = SCOPE_IDENTITY();

                -- Insertamos los {hijos} relacionados al {entity} recien creado
                INSERT INTO {esquema}.{TABLA_HIJA} ({entity}_id, {hijo}_id, usuario_reg, fecha_reg)
                SELECT @{entity}_id, h.{hijo}_id, @usuario_reg, GETDATE()
                FROM @{Hijos} h;

            COMMIT TRANSACTION;

            SET @msg = '{Entity} registrado correctamente.';

        END

    END TRY
    BEGIN CATCH

        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        SET @{entity}_id = 0;
        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```
