# Validation Patterns

Todo procedure valida ANTES de cualquier DML, en un solo bloque secuencial.
Cada validación es un `IF` plano (no anidado) que hace `RETURN` de inmediato
si falla — no se acumulan errores.

## Piso, no techo

Las 6 validaciones de abajo son la base estructural. Cada procedure agrega
además sus propias reglas de negocio, específicas de lo que hace — mismo
bloque, misma forma (`IF ... SET @error=1; SET @msg=...; RETURN;`). Ver el
ejemplo completo al final.

## Mecanismo: `RETURN`, no ifs anidados

T-SQL no tiene `BREAK` para condicionales. `RETURN` dentro del `IF` ya logra
"detener todo en la primera que falla" — valida plano y secuencial, no anides.

## Wrap defensivo: `IF (@error = 0)`

Además del `RETURN` por validación, el cuerpo/lógica (todo lo que va después
del bloque de validaciones) se envuelve en `IF (@error = 0) BEGIN ... END`.
Es redundante en teoría (si ya hizo `RETURN` nunca se llega ahí con
`@error = 1`), pero es una capa explícita a prueba de que alguien quite un
`RETURN` sin querer al mantener el SP. Ver plantilla en "Manejo de errores"
más abajo. No aplica a functions (no admiten `OUTPUT`).

## Orden: por dependencia, no solo por categoría

Una validación va después de cualquier otra de la que dependa. Ejemplo: al
actualizar un empleado, el chequeo de DNI duplicado debe excluir el propio
registro (`AND empleado_id <> @empleado_id`) — por eso va después de
confirmar que el empleado existe, no antes.

## Comentarios obligatorios

Todo bloque (validación o lógica) lleva un comentario corto en español
encima, en lenguaje natural — no un número de paso:

```sql
-- Validamos que el DNI exista
-- Verificamos que el DNI no este asociado a otro empleado
```

Aplica también a bloques de DML (`-- Insertamos el registro`) y a functions.

## Comentario extendido (bloques complejos)

Si un bloque implementa un algoritmo no obvio, o antecede una operación que
depende de una regla de negocio compleja, el comentario de una línea no
alcanza — usa un banner multi-línea explicando qué hace, por qué, y casos
especiales:

```sql
-- ==========================================
-- Algoritmo de Computus (Gregorian Anonymous)
-- Calcula la fecha de Pascua para el anho dado
-- ==========================================
```

Es la excepción a la regla de "una línea" — resérvalo para lógica que de
verdad lo amerita (algoritmos, cálculos no triviales), no para validaciones o
DML estándar.

## Orden obligatorio (piso — inserta tus validaciones de negocio donde su dependencia lo indique)

1. Campos requeridos.
2. FK / entidad padre existe y activa (si aplica).
3. La entidad objetivo existe (`Upd`/`Del`/`Get`).
4. La entidad objetivo está activa (`Del`).
5. Unicidad de campo de negocio contra activos (solo si la entidad lo requiere).
6. Dependencias en tablas hijas activas (`Del`).

> `codigo` u otro campo único: sin patrón reservado — decisión de negocio por
> entidad. Si aplica, documéntalo en la cabecera del SP y úbicalo en el punto 5.

## Snippets

### 1. Campo requerido

```sql
-- Validamos que el {campo} no sea nulo o vacio
IF @{campo} IS NULL OR LTRIM(RTRIM(@{campo})) = ''
BEGIN
    SET @error = 1;
    SET @msg   = 'El campo {campo} es obligatorio.';
    RETURN;
END
```

### 2. FK / entidad padre existe y activa

```sql
-- Validamos que el {entidad_padre} exista y este activo
IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA_PADRE} WHERE {padre}_id = @{padre}_id AND estado = 1)
BEGIN
    SET @error = 1;
    SET @msg   = 'La {entidad_padre} especificada no existe o se encuentra inactiva.';
    RETURN;
END
```

FK opcional (`NULL` permitido): envolver en `IF @{padre}_id IS NOT NULL BEGIN ... END`.

### 3. Entidad existe (Upd/Del/Get)

```sql
-- Validamos que el {entidad} exista
IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {entidad}_id = @{entidad}_id)
BEGIN
    SET @error = 1;
    SET @msg   = 'El {entidad} con el ID especificado no existe.';
    RETURN;
END
```

### 4. Entidad activa (Del)

```sql
-- Validamos que el {entidad} no este ya inactivo
IF NOT EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {entidad}_id = @{entidad}_id AND estado = 1)
BEGIN
    SET @error = 1;
    SET @msg   = 'El {entidad} ya se encuentra inactivo.';
    RETURN;
END
```

### 5. Unicidad de campo de negocio (solo si aplica)

`Ins` (sin registro propio que excluir):

```sql
-- Verificamos que el {campo_unico} no este asociado a otro {entidad} activo
IF EXISTS (SELECT 1 FROM {esquema}.{TABLA} WHERE {campo_unico} = @{campo_unico} AND estado = 1)
BEGIN
    SET @error = 1;
    SET @msg   = 'Ya existe un {entidad} activo con el {campo_unico} ingresado.';
    RETURN;
END
```

`Upd` (excluye el propio registro — por eso va después del snippet 3):

```sql
-- Verificamos que el {campo_unico} no este asociado a otro {entidad} activo (excluyendo el propio registro)
IF EXISTS (
    SELECT 1 FROM {esquema}.{TABLA}
    WHERE {campo_unico} = @{campo_unico}
      AND estado = 1
      AND {entidad}_id <> @{entidad}_id
)
BEGIN
    SET @error = 1;
    SET @msg   = 'Ya existe otro {entidad} activo con el {campo_unico} ingresado.';
    RETURN;
END
```

**Excepción justificada** (ej. `numero_ruc` compartido entre sucursales):
omite el bloque y documenta el motivo:

```sql
-- Nota: no se valida unicidad de {campo} porque {motivo de negocio}
```

### 6. Dependencia en tabla hija activa (Del)

```sql
-- Validamos que el {entidad} no tenga {entidades_hijas} activas asociadas
IF EXISTS (SELECT 1 FROM {esquema}.{TABLA_HIJA} WHERE {entidad}_id = @{entidad}_id AND estado = 1)
BEGIN
    SET @error = 1;
    SET @msg   = 'No se puede eliminar el {entidad} porque está asociado a {entidades_hijas} activas.';
    RETURN;
END
```

Un bloque por cada tabla hija. Omitir si no hay tablas hijas.

### 7. JSON (arrays/objetos anidados)

```sql
-- Validamos que {param} sea un JSON valido
IF @{param} IS NULL OR ISJSON(@{param}) = 0
BEGIN
    SET @error = 1;
    SET @msg   = 'El parámetro {param} no es un JSON válido.';
    RETURN;
END

-- Validamos que {param} tenga al menos un elemento
IF NOT EXISTS (SELECT 1 FROM OPENJSON(@{param}))
BEGIN
    SET @error = 1;
    SET @msg   = 'Debe enviar al menos un elemento en {param}.';
    RETURN;
END
```

### 8. Validación de negocio propia (patrón genérico)

```sql
-- {Descripcion corta en espanol de la regla de negocio}
IF {condicion_de_falla}
BEGIN
    SET @error = 1;
    SET @msg   = '{Mensaje de error en espanol para el usuario}.';
    RETURN;
END
```

## Ejemplo: base + negocio, en orden de dependencia

`Sp_Acc_Calcular_Nomina` — mezcla validaciones base y de negocio:

```sql
CREATE OR ALTER PROCEDURE nom.Sp_Acc_Calcular_Nomina
    @nomina_id      INT,
    @empleado_id    INT,
    @error          BIT            OUTPUT,
    @msg            VARCHAR(500)   OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @error = 0;
    SET @msg   = '';

    BEGIN TRY

        -- Validamos que el empleado exista y este activo (base)
        IF NOT EXISTS (SELECT 1 FROM per.EMPLEADO WHERE empleado_id = @empleado_id AND estado = 1)
        BEGIN
            SET @error = 1;
            SET @msg   = 'El empleado no existe o se encuentra inactivo.';
            RETURN;
        END

        -- Validamos que el empleado tenga un contrato vigente
        -- (negocio — depende de la validacion anterior)
        IF NOT EXISTS (
            SELECT 1 FROM per.CONTRATO
            WHERE empleado_id = @empleado_id AND estado = 1 AND vigente = 1
        )
        BEGIN
            SET @error = 1;
            SET @msg   = 'El empleado no tiene un contrato vigente.';
            RETURN;
        END

        -- Validamos que el empleado tenga su tipo de nomina configurado
        -- (negocio — se deriva del contrato validado arriba)
        IF NOT EXISTS (
            SELECT 1 FROM nom.EMPLEADO_TIPONOMINA
            WHERE empleado_id = @empleado_id AND estado = 1
        )
        BEGIN
            SET @error = 1;
            SET @msg   = 'El empleado no tiene un tipo de nomina configurado.';
            RETURN;
        END

        -- Validamos que la nomina tenga asientos contables registrados
        -- (negocio — independiente del empleado, se deja al final por costo)
        IF NOT EXISTS (
            SELECT 1 FROM cnt.ASIENTO WHERE nomina_id = @nomina_id AND estado = 1
        )
        BEGIN
            SET @error = 1;
            SET @msg   = 'La nomina no tiene asientos contables registrados.';
            RETURN;
        END

        -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
        IF (@error = 0)
        BEGIN

            -- Calculamos la nomina del empleado
            -- ... lógica de cálculo ...

            SET @msg = 'Nomina calculada correctamente.';

        END

    END TRY
    BEGIN CATCH

        SET @error = 1;
        SET @msg   = 'Error: ' + ERROR_MESSAGE();

    END CATCH
END
GO
```

## Tabla de mensajes estándar

| Contexto | Patrón |
|---|---|
| Insert exitoso | `'{Entidad} registrado correctamente.'` |
| Update exitoso | `'{Entidad} actualizado correctamente.'` |
| Delete exitoso | `'{Entidad} eliminado correctamente.'` |
| Select/Get exitoso | `'Consulta ejecutada correctamente.'` |
| Campo requerido | `'El campo {campo} es obligatorio.'` |
| No encontrado | `'El {entidad} con el ID especificado no existe.'` |
| Ya inactivo | `'El {entidad} ya se encuentra inactivo.'` |
| Padre inválido | `'La {entidad_padre} especificada no existe o se encuentra inactiva.'` |
| Bloqueo por dependencia | `'No se puede eliminar el {entidad} porque está asociado a {entidades_hijas} activas.'` |
| Catch | `'Error: ' + ERROR_MESSAGE()` |

## Manejo de errores (todo procedure)

```sql
SET NOCOUNT ON;
SET @error = 0;
SET @msg   = '';

BEGIN TRY
    -- Validaciones (base + de negocio, por dependencia, cada una comentada)

    -- Entra aqui solo si ninguna validacion anterior fallo (wrap defensivo)
    IF (@error = 0)
    BEGIN
        -- Logica de negocio
        SET @msg = '{mensaje de éxito}';
    END
END TRY
BEGIN CATCH
    SET @error = 1;
    SET @msg   = 'Error: ' + ERROR_MESSAGE();
END CATCH
```

Nunca `THROW` sin capturar — todo error se comunica vía `@error`/`@msg`.
