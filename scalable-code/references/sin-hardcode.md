# Sin hardcode de valores de negocio

Regla dura, transversal a BD, backend y frontend. Complementa a
`regla-codigo-escalable.md`: allí se dice que lo compartido se escribe una
sola vez; aquí se dice **dónde vive un valor de negocio**: en un dato, nunca
en el código.

## La regla

Todo conjunto de valores o código de negocio que el usuario elige o que el
sistema distingue (tipos, estados, criterios, métodos, sentidos, códigos
SUNAT con regla propia) vive en:

- una **TABLA** de la BD con su **carga inicial** (seed), aunque no tenga
  CRUD; o
- un **ATRIBUTO** (columna `BIT` o de valor) de la tabla que ya existe.

El código **lee ese dato**. El comportamiento diferente por valor se expresa
como una columna (`costo_incluye_igv`, `es_compra`, `exige_detalle`), no como
una comparación contra un texto.

## Prohibido

- `CHECK (... IN (...))` con valores de negocio.
- `enum`, constantes o arreglos de códigos en TypeScript o C#
  (`CODIGOS_TIPO_CON_IGV = ['03','12']`).
- `IF` / `CASE` / `switch` / `IIF` que comparan contra textos o códigos
  escritos (`'02'`, `'00'`, `'OTRO'`).
- Duplicar la misma regla en front y BD.

Si una lista fija pareciera la mejor opción, **solo se sugiere** al usuario,
con argumentos; él decide. Jamás se asume.

## Excepción acotada

Un `CASE` por código es aceptable **solo** cuando cada código es un cálculo
distinto programado (ej. criterios de cálculo) **y** el código sale de su
tabla (con FK), no de un `CHECK` ni de una constante. Se documenta por qué en
el propio SP o en la decisión del proyecto.

## El front nunca compara contra códigos

Recibe el atributo ya resuelto desde la API (`t.costo_incluye_igv`,
`t.trae_importes`), dentro del DTO de respuesta. Si el front necesita
distinguir algo, falta una columna en la tabla y un campo en el DTO.

## Los 6 casos (sistema de ventas/inventario)

### 1. Costo con IGV

Así NO (front):

```ts
const CODIGOS_TIPO_CON_IGV = ['03', '12'];
const conIgv = CODIGOS_TIPO_CON_IGV.includes(t.codigo_sunat);
```

Así SÍ: columna `costo_incluye_igv BIT` en `cat.TIPO_COMPROBANTE` (1 en 03 y
12 por seed); la API la entrega en el DTO.

```ts
const conIgv = t.costo_incluye_igv;
```

### 2. Tipos que muestran "Total del documento"

Así NO (front):

```ts
const CODIGOS_TIPO_CON_IMPORTES = ['01', '03', '04', '07', '08', '12'];
```

Así SÍ: atributo `trae_importes BIT` en `cat.TIPO_COMPROBANTE`.

```ts
const muestraTotal = t.trae_importes;
```

### 3. "Es una compra"

Así NO (SQL, repetido en varios SPs):

```sql
IIF(o.codigo_sunat = '02', 1, 0)
```

Así SÍ: atributo `es_compra BIT` en `cat.TIPO_OPERACION`.

```sql
o.es_compra
```

Marcar otro tipo (ej. 18 importación) no exige tocar ningún SP.

### 4. "Sin documento"

Así NO:

```sql
IIF(@tipo_codigo = '00', 1, 0)
```

```ts
const CODIGO_SIN_DOCUMENTO = '00';
```

Así SÍ: atributo `exige_serie_numero BIT` (0 en el tipo 00). SQL y front
leen el atributo:

```sql
IF @exige_serie_numero = 1 AND (@serie IS NULL OR @numero IS NULL) ...
```

```ts
const pideSerie = t.exige_serie_numero;
```

### 5. Motivo de anulación "Otro" exige detalle

Así NO: los motivos viven en un catálogo genérico que no admite columnas, y
la regla se escribe a mano en SQL y en front.

```sql
IF @tipo_motivo = 'OTRO' AND @detalle IS NULL ...
```

```ts
const CODIGO_MOTIVO_OTRO = 'OTRO';
```

Así SÍ: tabla propia `cat.MOTIVO_ANULACION_INGRESO` con `exige_detalle BIT`.
Lo que tiene reglas sale del catálogo genérico: una tabla, una function.

```sql
IF @exige_detalle = 1 AND @detalle IS NULL ...
```

```ts
const pideDetalle = motivo.exige_detalle;
```

### 6. Criterio del costo propuesto

Así NO: tres sitios para la misma lista.

```sql
CHECK (costo_propuesto IN ('ULTIMA_COMPRA','PROMEDIO'))
```

```csharp
private static readonly string[] Criterios = { "ULTIMA_COMPRA", "PROMEDIO" };
```

```sql
CASE @costo_propuesto WHEN 'ULTIMA_COMPRA' THEN ... WHEN 'PROMEDIO' THEN ... END
```

Así SÍ: tabla `cat.CRITERIO_COSTO` (carga inicial, sin CRUD) con FK
`criterio_costo_id` desde `cat.MOTIVO_MOVIMIENTO`. El validador solo
verifica que el criterio exista y esté activo. El `CASE` por código del SP
queda como **excepción acotada**: cada criterio es un cálculo distinto y el
código sale de la tabla.

```sql
CASE c.codigo WHEN 'ULTIMA_COMPRA' THEN ... WHEN 'PROMEDIO' THEN ... END
FROM cat.CRITERIO_COSTO c WHERE c.id = @criterio_costo_id
```

## Caso positivo de referencia: `cat.FORMA_REPARTO`

Tabla con carga inicial y sin CRUD; las formas pueden estar activas o
inactivas (`estado`). El SP calcula según el código de la forma, que sale de
la tabla. Es el modelo a copiar cuando aparezca un conjunto nuevo.

## Verificación (checklist)

Búsquedas que deben dar **cero resultados nuevos** fuera de las excepciones
documentadas:

```bash
grep -rnE "CHECK.*\bIN \(" --include=*.sql .
grep -rnE "(const|readonly).*\[\s*['\"][0-9A-Z_]+['\"]\s*," --include=*.ts --include=*.cs .
grep -rnE "(=|<>|!=)\s*'[0-9]{2}'" --include=*.sql .
grep -rnE "(===|!==|==)\s*['\"][0-9A-Z_]{2,}['\"]" --include=*.ts .
```

Los patrones se ajustan a las rutas reales del repo; el criterio no cambia.
