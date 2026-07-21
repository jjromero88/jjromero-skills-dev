# DDL Conventions

## Patrón de creación de tabla

```sql
/* =========================
   {NOMBRE_TABLA}
   ========================= */

CREATE TABLE {esquema}.{NOMBRE_TABLA} (
    {entidad}_id   INT NOT NULL PRIMARY KEY IDENTITY(1,1),
    -- ...columnas de negocio...,
    estado         BIT          NULL DEFAULT 1,
    usuario_reg    VARCHAR(50)  NULL DEFAULT 'sys',
    fecha_reg      DATETIME     NULL DEFAULT GETDATE(),
    usuario_act    VARCHAR(50)  NULL,
    fecha_act      DATETIME     NULL
);
GO
```

Cabecera `/* ... */` con el nombre de tabla — no usar `-- ===`.

## Nomenclatura

| Elemento | Convención | Ejemplo |
|---|---|---|
| Esquema | minúsculas, por módulo | `seg`, `per`, `nom` |
| Tabla | MAYÚSCULAS, singular | `USUARIO`, `PERFIL` |
| Columna | `snake_case` | `usuario_id`, `descripcion_larga` |
| Primary key | `{tabla}_id` | `permiso_id` |
| Foreign key | `{tabla_referenciada}_id` | `perfil_id` |
| Auto-referencia | `padre_id` | `SISTEMAOPCION.padre_id` |

## Tipos de dato estándar

| Propósito | Tipo | Ejemplo |
|---|---|---|
| Primary key | `INT NOT NULL PRIMARY KEY IDENTITY(1,1)` | `permiso_id` |
| Foreign key | `INT NULL FOREIGN KEY REFERENCES {esquema}.{TABLA}` | `perfil_id` |
| Código/campo único (si aplica) | `VARCHAR(20)` o el tamaño de negocio — ver "Generación de `codigo`" abajo | `codigo` |
| Texto corto | `VARCHAR(50)`–`VARCHAR(100)` | `descripcion` |
| Texto largo | `VARCHAR(200)`–`VARCHAR(300)` | `descripcion_larga` |
| Email | `VARCHAR(150)`–`VARCHAR(200)` | `email` |
| URL/ruta | `VARCHAR(300)`–`VARCHAR(500)` | `opcion_url` |
| Username | `VARCHAR(50)` | `username` |
| Password (hash) | `VARCHAR(200)` | `password` |
| RUC/identificador fiscal | `VARCHAR(20)`, validar longitud en el SP | `numero_ruc` |
| Teléfono | `VARCHAR(50)` | `telefono` |
| Flag booleano | `BIT NULL DEFAULT 1` | `habilitado`, `estado` |
| Fecha/hora | `DATETIME NULL` | `fecha_reg` |
| Orden de despliegue | `INT NULL` | `opcion_orden` |
| JSON de entrada | `NVARCHAR(MAX)`, validar con `ISJSON()` | `sociedad_ids` |

## Columnas de auditoría (siempre, en este orden, al final)

Español (default):

```sql
estado       BIT          NULL DEFAULT 1,
usuario_reg  VARCHAR(50)  NULL DEFAULT 'sys',
fecha_reg    DATETIME     NULL DEFAULT GETDATE(),
usuario_act  VARCHAR(50)  NULL,
fecha_act    DATETIME     NULL
```

Inglés (si el proyecto decidió nomenclatura en inglés — ver
`SKILL.md`, "Antes de crear"): **traducción idiomática, no literal** — es
la convención estándar en inglés para estas columnas, no una traducción
palabra por palabra.

```sql
status       BIT          NULL DEFAULT 1,
created_by   VARCHAR(50)  NULL DEFAULT 'sys',
created_at   DATETIME     NULL DEFAULT GETDATE(),
updated_by   VARCHAR(50)  NULL,
updated_at   DATETIME     NULL
```

| Español | Inglés |
|---|---|
| `estado` | `status` |
| `usuario_reg` | `created_by` |
| `fecha_reg` | `created_at` |
| `usuario_act` | `updated_by` |
| `fecha_act` | `updated_at` |

En ambos casos, el resto de tablas/columnas/PK/FK sigue el mismo patrón de
nomenclatura (mayúsculas para tabla, snake_case para columna, `{tabla}_id`)
— solo cambia el idioma de las palabras (`PRODUCTO`→`PRODUCT`,
`categoria_id`→`category_id`). El esquema fijo de la skill (`Sp_`,
`Ins`/`Upd`/`Del`/`Sel`/`Get`/`Ver`/`Acc`, `fn_svf_`/`fn_tvf_`) nunca
cambia — es la sintaxis propia de la skill, no vocabulario de negocio.

## Foreign Key

Inline, calificada por esquema:

```sql
perfil_id    INT NULL FOREIGN KEY REFERENCES seg.PERFIL,
empresa_id   INT NULL FOREIGN KEY REFERENCES seg.EMPRESA,
padre_id     INT NULL FOREIGN KEY REFERENCES seg.SISTEMAOPCION
```

## Tablas puente (muchos-a-muchos)

Nombre = concatenación de ambas entidades, con sus propias columnas de auditoría:

```sql
CREATE TABLE seg.USUARIOPERFIL (
    usuarioperfil_id  INT NOT NULL PRIMARY KEY IDENTITY(1,1),
    usuario_id        INT NULL FOREIGN KEY REFERENCES seg.USUARIO,
    perfil_id         INT NULL FOREIGN KEY REFERENCES seg.PERFIL,
    estado            BIT          NULL DEFAULT 1,
    usuario_reg       VARCHAR(50)  NULL DEFAULT 'sys',
    fecha_reg         DATETIME     NULL DEFAULT GETDATE(),
    usuario_act       VARCHAR(50)  NULL,
    fecha_act         DATETIME     NULL
);
```

Ejemplos: `USUARIOPERFIL`, `PERFILOPCION`, `OPCIONPERMISOS`, `SOCIEDADPERFIL`.

## Patrón CATALOGO

Catálogos simples (tipo documento, estado civil, moneda...) van en una tabla
única, no una tabla por catálogo.

**Antes de asignarle esquema, decide**: ¿esta tabla (`CATALOGO` u otra
tabla maestra) la va a usar un solo módulo, o 2+ módulos de negocio? Nunca
asumas el esquema "de turno" (el que estabas usando para la última
entidad) solo por comodidad — pregúntalo si no es obvio.

- **Transversal** (2+ módulos la reusan) → esquema `cat`. Ejemplos:
  `TIPO_DOCUMENTO`, `MONEDA`, `UNIDAD_MEDIDA` — cualquier módulo puede
  necesitarlos.
- **Específica de un módulo** → el esquema de ese módulo. Ejemplo: un
  catálogo de tipos de movimiento que solo usa el módulo de Inventario va
  en `inv`, no en `cat`.

```sql
CREATE TABLE {esquema}.CATALOGO (
    catalogo_id    INT IDENTITY(1,1) PRIMARY KEY,
    tipo           VARCHAR(30)  NOT NULL,   -- discrimina el catálogo, siempre abreviado (ver abajo)
    codigo         VARCHAR(20)  NOT NULL,
    descripcion    VARCHAR(200) NOT NULL,
    abreviatura    VARCHAR(20)  NULL,
    orden          INT          NOT NULL DEFAULT 0,
    estado         BIT          NOT NULL DEFAULT 1,
    usuario_reg    VARCHAR(50)  NOT NULL,
    fecha_reg      DATETIME     NOT NULL DEFAULT GETDATE(),
    usuario_act    VARCHAR(50)  NULL,
    fecha_act      DATETIME     NULL
);
```

Tabla dedicada solo si el catálogo necesita columnas propias (ej.
`TIPO_DOCUMENTO`, `PAIS`, `MONEDA`). Se consume desde el frontend con cache —
ver skill de Angular.

**`tipo` siempre abreviado, nunca el nombre semántico completo** — la idea
es que la discriminación sea corta, no descriptiva:

| Catálogo (semántica) | `tipo` (abreviado) |
|---|---|
| Categoría de producto | `CAT_PROD` |
| Sexo | `SEXO` |
| Tipo de documento | `TIPO_DOC` |
| Método de pago | `MET_PAGO` |

## Generación de `codigo`

Aplica a **cualquier tabla que tenga columna `codigo`**, no solo
`CATALOGO`. Default obligatorio: se autogenera a partir del PK identity,
con ceros a la izquierda a 5 dígitos (`1` → `'00001'`, `182` → `'00182'`).
Solo se abandona este default si el usuario pide explícitamente un
formato de `codigo` distinto para esa entidad puntual.

Como el identity solo se conoce después del `INSERT`, se setea en un
segundo paso (`UPDATE`) inmediatamente después — ver plantilla `Sp_Ins` en
`references/sp-templates.md`:

```sql
UPDATE {esquema}.{TABLA}
SET codigo = RIGHT('00000' + CAST(@{entity}_id AS VARCHAR(5)), 5)
WHERE {entity}_id = @{entity}_id;
```

**Solo se genera en `Sp_Ins` — nunca en `Sp_Upd`.** `codigo` existe para
que el usuario final identifique un registro sin ver el PK real; no es un
dato de negocio editable. Por defecto, `Sp_Upd` no recibe `codigo` como
parámetro ni lo toca. Solo se vuelve editable si el usuario lo pide
explícitamente para una entidad puntual.

## `habilitado` vs `estado`

- `estado` = existencia lógica (0 = eliminado).
- `habilitado` = toggle operacional, independiente de la eliminación.

Ejemplo real con ambas: `SISTEMAOPCION`, `PERFILOPCION`, `OPCIONPERMISOS`.

## Índices para columnas de JOIN

Toda columna que participe en un JOIN o filtro frecuente y NO sea PK/FK
necesita un índice — si no, el plan de ejecución escanea la tabla completa.

```sql
CREATE NONCLUSTERED INDEX IX_{TABLA}_{columna} ON {esquema}.{TABLA} ({columna});
```

- No-clusterado por defecto. Clusterado solo si esa columna es el patrón de
  acceso dominante de la tabla (raro — la PK ya suele ser clusterada).
- Ejemplos típicos: `codigo`, `numero_documento`, columnas de fecha usadas en
  rangos de filtro.

### Plan de ejecución — obligatorio con este trigger

Revisar el plan de ejecución NO es opcional cuando el `SELECT` cumple AMBAS:

1. Tiene **más de un JOIN**.
2. Al menos un JOIN/WHERE compara una columna que **no es PK ni FK**.

Un `SELECT` simple (0-1 join, o joins solo por PK/FK) no lo requiere.

Cuando el trigger se cumple:
- Si tienes acceso/credenciales a la BD, ejecuta el plan de ejecución tú
  mismo y agrega el índice si detectas un escaneo costoso.
- Si no tienes acceso, **solicítalo explícitamente al usuario** (credenciales,
  connection string, o permiso de conexión) antes de dar el SP por
  terminado — no lo omitas en silencio ni lo dejes como sugerencia pasiva.

**Reporta siempre el resultado, sea cual sea:**
- **Sin cuellos de botella**: confirma explícitamente que revisaste el plan
  de ejecución y no encontraste JOINs de costo alto.
- **Con cuellos de botella**: reporta qué JOIN/columna tiene costo alto y
  sugiere la solución — típicamente un índice (clustered o non-clustered,
  según corresponda) sobre el/los campos involucrados, u otra solución que
  infieras si el índice no es la causa raíz. Nunca ejecutes el chequeo en
  silencio sin comunicar qué encontraste.

## Equivalencias SQL Server / PostgreSQL

| SQL Server | PostgreSQL |
|---|---|
| `SCOPE_IDENTITY()` | `RETURNING {entity}_id` |
| `IDENTITY(1,1)` | `GENERATED ALWAYS AS IDENTITY` |
| `NVARCHAR(MAX)` + `OPENJSON`/`ISJSON` | `JSONB` + `->`/`->>`/`jsonb_array_elements` |
| `GETDATE()` | `NOW()` |
| Procedure con `OUTPUT` | Función con `RETURNS TABLE` o `OUT` |
| `BEGIN TRY/CATCH` | `BEGIN ... EXCEPTION WHEN OTHERS THEN ...` (PL/pgSQL) |

Contrato `@error`/`@msg` se preserva conceptualmente: `0`/`false` = éxito,
mensaje siempre en español.
