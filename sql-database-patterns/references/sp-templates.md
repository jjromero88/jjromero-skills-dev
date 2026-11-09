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

## Sp_Sel paginado con orden por columna

Obligatorio cuando el listado pagina en el servidor y tiene columnas
ordenables (la pregunta del `SKILL.md`, "Antes de crear", punto 4; "Ninguna"
es válida y entonces el SP no lleva los parámetros de orden). Lo resuelve
primero el backend contra su lista blanca (`OrdenListado` en
`dotnet-clean-architecture`); el SP **vuelve a validar** (defensa en
profundidad) y nunca arma SQL dinámico:

- `@orden_campo` = valor de la lista blanca (`NULL` = sin orden) y
  `@orden_sentido` = `'ASC'` / `'DESC'`.
- Un campo que el SP no conoce queda en `NULL`; un sentido distinto de
  `DESC` (sin distinguir mayúsculas ni espacios) es `ASC`.
- Un `CASE` por columna **y por sentido** (un `CASE` no puede cambiar de
  `ASC` a `DESC`), y la PK al final: sin orden ningún `CASE` aplica y queda
  el orden de registro, determinista para `OFFSET`/`FETCH`.
- El total de coincidencias viaja en cada fila (`COUNT(*) OVER ()`).

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Sel_{Entity}
    @busqueda         NVARCHAR(200) = NULL,
    @orden_campo      VARCHAR(30)   = NULL,     -- '{campo_1}', '{campo_2}' o NULL (sin orden: orden de registro)
    @orden_sentido    VARCHAR(4)    = 'ASC',    -- 'ASC' o 'DESC'
    @pagina           INT           = 1,
    @filas_por_pagina INT           = 10,
    @error            BIT           OUTPUT,
    @msg              VARCHAR(500)  OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @error = 0;
    SET @msg   = '';

    -- Normalizamos la paginacion (pagina desde 1, de 1 a 100 filas)
    SET @pagina = IIF(ISNULL(@pagina, 1) < 1, 1, @pagina);
    SET @filas_por_pagina = IIF(ISNULL(@filas_por_pagina, 10) NOT BETWEEN 1 AND 100, 10, @filas_por_pagina);

    -- Normalizamos el orden: un campo desconocido queda sin orden (orden de registro) y un sentido distinto de
    -- DESC es ascendente
    SET @orden_sentido = IIF(@orden_campo IN ('{campo_1}', '{campo_2}')
                             AND UPPER(LTRIM(RTRIM(@orden_sentido))) = 'DESC', 'DESC', 'ASC');
    SET @orden_campo = IIF(@orden_campo IN ('{campo_1}', '{campo_2}'), @orden_campo, NULL);

    BEGIN TRY

        -- Filtramos, ordenamos y paginamos los {entity} activos
        SELECT  e.{entity}_id,
                e.{campo_1},
                e.{campo_2},
                COUNT(*) OVER () AS total_registros
        FROM    {esquema}.{TABLA} e
        WHERE   e.estado = 1
          AND   (@busqueda IS NULL OR e.{campo_2} LIKE N'%' + @busqueda + N'%')
        -- Sin orden ningun CASE aplica y queda solo la PK: orden de registro, determinista para paginar
        ORDER BY
                CASE WHEN @orden_campo = '{campo_1}' AND @orden_sentido = 'ASC'  THEN e.{campo_1} END ASC,
                CASE WHEN @orden_campo = '{campo_1}' AND @orden_sentido = 'DESC' THEN e.{campo_1} END DESC,
                CASE WHEN @orden_campo = '{campo_2}' AND @orden_sentido = 'ASC'  THEN e.{campo_2} END ASC,
                CASE WHEN @orden_campo = '{campo_2}' AND @orden_sentido = 'DESC' THEN e.{campo_2} END DESC,
                e.{entity}_id
        OFFSET (@pagina - 1) * @filas_por_pagina ROWS
        FETCH NEXT @filas_por_pagina ROWS ONLY;

        SET @msg = 'Consulta ejecutada correctamente.';

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

Columnas de tipos distintos (texto y fecha, por ejemplo) van cada una en su
propio par de `CASE`, nunca mezcladas en el mismo `CASE`. Pruebas de
integración obligatorias (`dotnet-clean-architecture`, `references/testing.md`):
cada columna ASC y DESC, sin orden = orden de PK (con datos registrados en un
orden distinto al de las columnas, para que la prueba distinga), y un campo o
sentido no permitido (incluido un intento de inyección) cae al orden de
registro sin error.

---

## Sp_Sel paginado con filtros por columna

Obligatorio cuando el listado pagina en el servidor y tiene columnas
filtrables (la pregunta del `SKILL.md`, "Antes de crear", punto 5;
"Ninguna" es válida y entonces el SP no lleva estos parámetros). Lo resuelve
primero el backend contra su lista blanca (`FiltroListado` en
`dotnet-clean-architecture`): el SP recibe **valores ya tipados, un
parámetro (o un grupo) por columna filtrable**, y **vuelve a validar**
(defensa en profundidad). Nunca SQL dinámico, nunca un nombre de columna ni
un operador que no esté en el `CASE` del propio SP.

| Tipo de filtro de la columna | Parámetros | Sin filtro cuando |
|---|---|---|
| Catálogo (casillas / lista de IDs) | `@{col}_ids NVARCHAR(MAX)`: arreglo JSON (`'[3,7]'`) | `NULL`, JSON inválido o sin IDs válidos |
| Booleano (activo / inactivo) | `@{col} BIT = NULL` | `NULL` (las dos casillas marcadas) |
| Condición de texto | `@{col}_operador VARCHAR(12)` + `@{col}_valor NVARCHAR(200)` | operador desconocido o valor vacío |
| Condición o rango numérico | `@{col}_operador VARCHAR(12)` + `@{col}_valor` + `@{col}_hasta` (del tipo de la columna) | operador desconocido, valor `NULL`, o `entre` sin hasta / al revés |
| Fecha o rango de fechas | `@{col}_operador VARCHAR(12)` + `@{col}_desde DATE` + `@{col}_hasta DATE` | igual que el numérico |

**Lista de IDs**: una function inline compartida que lee el arreglo JSON
(un `IN (SELECT ... OPENJSON(...))` repetido por columna y por SP es
duplicación). Solo un arreglo JSON válido llega a `OPENJSON` (un texto
inválido lo haría fallar) y solo cuentan los números enteros positivos:

```sql
CREATE OR ALTER FUNCTION {esquema_transversal}.fn_tvf_ListaIds (@json NVARCHAR(MAX))
RETURNS TABLE
AS
RETURN
(
    SELECT  DISTINCT TRY_CAST(j.value AS INT) AS id
    FROM    OPENJSON(IIF(ISJSON(@json) = 1 AND LEFT(LTRIM(@json), 1) = N'[', @json, N'[]')) j
    WHERE   j.type = 2                         -- numeros (no textos, objetos ni NULL)
      AND   TRY_CAST(j.value AS INT) > 0       -- enteros positivos (no decimales)
);
GO
```

```sql
CREATE OR ALTER PROCEDURE {esquema}.Sp_Sel_{Entity}
    @busqueda           NVARCHAR(200) = NULL,
    @{catalogo}_ids     NVARCHAR(MAX) = NULL,     -- '[3,7]'; NULL = todos
    @habilitado         BIT           = NULL,     -- 1 activos, 0 inactivos, NULL = ambos
    @{texto}_operador   VARCHAR(12)   = NULL,     -- contiene, no_contiene, empieza, termina, igual, diferente
    @{texto}_valor      NVARCHAR(200) = NULL,
    @{numero}_operador  VARCHAR(12)   = NULL,     -- igual, diferente, mayor, mayor_igual, menor, menor_igual, entre
    @{numero}_valor     DECIMAL(12,2) = NULL,
    @{numero}_hasta     DECIMAL(12,2) = NULL,     -- solo con 'entre'
    @{fecha}_operador   VARCHAR(12)   = NULL,     -- igual, entre, mayor_igual, menor_igual
    @{fecha}_desde      DATE          = NULL,
    @{fecha}_hasta      DATE          = NULL,     -- solo con 'entre'
    @orden_campo        VARCHAR(30)   = NULL,
    @orden_sentido      VARCHAR(4)    = 'ASC',
    @pagina             INT           = 1,
    @filas_por_pagina   INT           = 10,
    @error              BIT           OUTPUT,
    @msg                VARCHAR(500)  OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @error = 0;
    SET @msg   = '';

    -- (paginacion y orden normalizados como en "Sp_Sel paginado con orden por columna")

    -- Normalizamos los filtros por columna (RG-38): lo invalido es "sin filtro", nunca "ninguna fila"
    DECLARE @hay_{catalogo} BIT = IIF(EXISTS (SELECT 1 FROM {esquema_transversal}.fn_tvf_ListaIds(@{catalogo}_ids)), 1, 0);

    SET @{texto}_valor = NULLIF(LTRIM(RTRIM(@{texto}_valor)), N'');
    SET @{texto}_operador = IIF(@{texto}_valor IS NOT NULL
                                AND @{texto}_operador IN ('contiene', 'no_contiene', 'empieza', 'termina', 'igual', 'diferente'),
                                @{texto}_operador, NULL);
    -- Escapamos los comodines del LIKE ([, % y _ se buscan literales)
    DECLARE @{texto}_patron NVARCHAR(400) =
        REPLACE(REPLACE(REPLACE(@{texto}_valor, N'[', N'[[]'), N'%', N'[%]'), N'_', N'[_]');

    SET @{numero}_operador = IIF(@{numero}_valor IS NOT NULL
                                 AND @{numero}_operador IN ('igual', 'diferente', 'mayor', 'mayor_igual', 'menor', 'menor_igual')
                                 OR (@{numero}_operador = 'entre' AND @{numero}_valor <= @{numero}_hasta),
                                 @{numero}_operador, NULL);

    SET @{fecha}_operador = IIF(@{fecha}_desde IS NOT NULL AND @{fecha}_operador IN ('igual', 'mayor_igual')
                                OR (@{fecha}_operador = 'menor_igual' AND @{fecha}_hasta IS NOT NULL)
                                OR (@{fecha}_operador = 'entre' AND @{fecha}_desde <= @{fecha}_hasta),
                                @{fecha}_operador, NULL);

    BEGIN TRY

        SELECT  e.{entity}_id,
                e.{texto},
                COUNT(*) OVER () AS total_registros
        FROM    {esquema}.{TABLA} e
        WHERE   e.estado = 1
          -- Catalogo: cualquiera de los IDs marcados
          AND   (@hay_{catalogo} = 0 OR e.{catalogo}_id IN (SELECT id FROM {esquema_transversal}.fn_tvf_ListaIds(@{catalogo}_ids)))
          -- Booleano
          AND   (@habilitado IS NULL OR e.habilitado = @habilitado)
          -- Condicion de texto, sin distinguir mayusculas ni tildes; un vacio cumple "no contiene" y "es diferente de"
          AND   (@{texto}_operador IS NULL
                 OR (@{texto}_operador = 'contiene'    AND ISNULL(e.{texto}, N'') COLLATE SQL_Latin1_General_CP1_CI_AI LIKE N'%' + @{texto}_patron + N'%')
                 OR (@{texto}_operador = 'no_contiene' AND ISNULL(e.{texto}, N'') COLLATE SQL_Latin1_General_CP1_CI_AI NOT LIKE N'%' + @{texto}_patron + N'%')
                 OR (@{texto}_operador = 'empieza'     AND ISNULL(e.{texto}, N'') COLLATE SQL_Latin1_General_CP1_CI_AI LIKE @{texto}_patron + N'%')
                 OR (@{texto}_operador = 'termina'     AND ISNULL(e.{texto}, N'') COLLATE SQL_Latin1_General_CP1_CI_AI LIKE N'%' + @{texto}_patron)
                 OR (@{texto}_operador = 'igual'       AND ISNULL(e.{texto}, N'') COLLATE SQL_Latin1_General_CP1_CI_AI = @{texto}_valor)
                 OR (@{texto}_operador = 'diferente'   AND ISNULL(e.{texto}, N'') COLLATE SQL_Latin1_General_CP1_CI_AI <> @{texto}_valor))
          -- Condicion o rango numerico (un NULL en la fila solo cumple "es diferente de")
          AND   (@{numero}_operador IS NULL
                 OR (@{numero}_operador = 'igual'       AND e.{numero} = @{numero}_valor)
                 OR (@{numero}_operador = 'diferente'   AND (e.{numero} <> @{numero}_valor OR e.{numero} IS NULL))
                 OR (@{numero}_operador = 'mayor'       AND e.{numero} > @{numero}_valor)
                 OR (@{numero}_operador = 'mayor_igual' AND e.{numero} >= @{numero}_valor)
                 OR (@{numero}_operador = 'menor'       AND e.{numero} < @{numero}_valor)
                 OR (@{numero}_operador = 'menor_igual' AND e.{numero} <= @{numero}_valor)
                 OR (@{numero}_operador = 'entre'       AND e.{numero} BETWEEN @{numero}_valor AND @{numero}_hasta))
          -- Fecha o rango de fechas por dia (una columna DATETIME se compara sin la hora)
          AND   (@{fecha}_operador IS NULL
                 OR (@{fecha}_operador = 'igual'       AND e.{fecha} >= @{fecha}_desde AND e.{fecha} < DATEADD(DAY, 1, @{fecha}_desde))
                 OR (@{fecha}_operador = 'mayor_igual' AND e.{fecha} >= @{fecha}_desde)
                 OR (@{fecha}_operador = 'menor_igual' AND e.{fecha} < DATEADD(DAY, 1, @{fecha}_hasta))
                 OR (@{fecha}_operador = 'entre'       AND e.{fecha} >= @{fecha}_desde AND e.{fecha} < DATEADD(DAY, 1, @{fecha}_hasta)))
        ORDER BY
                -- (CASE de orden por columna y sentido)
                e.{entity}_id
        OFFSET (@pagina - 1) * @filas_por_pagina ROWS
        FETCH NEXT @filas_por_pagina ROWS ONLY;

        SET @msg = 'Consulta ejecutada correctamente.';

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

- Solo se declaran los grupos de las columnas que el listado filtra (en
  ComercialMs, `inv.Sp_Sel_Producto`: `@marca_ids`, `@linea_ids` y
  `@habilitado`).
- Un rango con un solo extremo llega como `mayor_igual` (solo desde) o
  `menor_igual` (solo hasta); `entre` siempre trae los dos.
- El total (`COUNT(*) OVER ()`) se calcula después de todos los filtros:
  es el de las coincidencias, no el de la tabla.

Pruebas de integración obligatorias (`dotnet-clean-architecture`,
`references/testing.md`, punto 6): cada columna con datos reales (varios
IDs, cada booleano y los dos, cada operador y rango usado), la combinación
`AND` con la búsqueda, el orden y la página, el total, y llamando al SP
directo, listas que no son un arreglo JSON de enteros (texto, objeto,
decimales, negativos, vacío) se toman como sin filtro sin error.

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
