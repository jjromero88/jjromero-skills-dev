# Repo Structure and Deployment

## Por qué

Todo objeto SQL generado no solo se ejecuta contra la BD — se persiste
como `.sql` en el repo del proyecto. Dos motivos: (1) es el backup de la
fuente real de la BD, junto al resto del código; (2) al pasar una feature
a QA/prod, se entregan los `.sql` correspondientes para levantar esos
cambios — el propio historial de git (commits/diffs) dice qué `.sql`
pertenece a qué feature/pase, no una partición manual de archivos.

**Regla dura**: todo DDL/SP/function/type generado por esta skill se
persiste en la estructura de abajo — obligatorio, sin excepción, salvo
`seeds/` (ver más abajo, requiere confirmación explícita del usuario).

## Árbol de carpetas

```
{proyecto}_db/
  schemas/
    schemas.sql            → CREATE SCHEMA de todos los módulos, append-only
  tables/
    tables.sql             → CREATE TABLE + ALTER TABLE + CREATE INDEX, append-only
  procedures/
    {Core}/
      {Entidad}/            → CRUD: Sp_Sel/Get/Ins/Upd/Del_{Entidad}.sql
      Procesos/              → Sp_Acc_/Sp_Ver_ que no pertenecen a una sola entidad, planos
  functions/
    fn_svf_{Concepto}.sql
    fn_tvf_{Concepto}.sql
  types/
    TT_{Concepto}.sql
  seeds/
    {entidad}_seed.sql      → solo si el usuario confirmó que lo quiere persistir
```

## `schemas.sql` y `tables.sql` — scripts vivos, append-only

Un único script por tipo de objeto, que crece con cada tabla/esquema
nuevo o modificado — **nunca se reescribe una sentencia ya puesta**:

- Tabla nueva → se agrega su `CREATE TABLE` al final de `tables.sql`.
- Se modifica una columna de una tabla existente → se agrega un
  `ALTER TABLE` al final, **no** se edita el `CREATE TABLE` original.
- Se agrega un índice (ver `ddl-conventions.md`, "Índices para columnas de
  JOIN") → se agrega el `CREATE INDEX` al final de `tables.sql`, mismo
  modelo append-only — no es un archivo aparte.
- Igual con `schemas.sql`: cada `CREATE SCHEMA` se agrega al final, en el
  orden en que se van creando los módulos.

Correr el script de punta a punta reconstruye el estado actual
reproduciendo los cambios en el orden en que ocurrieron — es a la vez
backup y bitácora de cambios.

**Orden de dependencia obligatorio**:
- `schemas.sql` corre antes que `tables.sql` (un schema debe existir antes
  que sus tablas).
- Dentro de `tables.sql`, toda tabla padre (referenciada por FK) debe
  quedar **antes** que sus tablas hijas. Si se agrega una tabla hija antes
  de que su padre esté en el script, reordena antes de dar la tarea por
  terminada.

## `procedures/` — agrupado por `{Core}`

Mismo `{Core}` que usan backend y frontend (ver skill
`business-domain-grouping`) — no el nombre crudo del esquema SQL. Si un
esquema se parte en 2 cores (regla 2 de `business-domain-grouping`), sus
procedures también se separan en 2 carpetas.

- **CRUD de una entidad** → `procedures/{Core}/{Entidad}/` con los 5 SPs
  (`Sp_Sel`/`Sp_Get`/`Sp_Ins`/`Sp_Upd`/`Sp_Del`).
- **Proceso de negocio no-CRUD** (`Sp_Acc_`/`Sp_Ver_` que no pertenece a
  una sola entidad, ej. un proceso de cálculo o una verificación
  transversal) → `procedures/{Core}/Procesos/`, plano — todos los procesos
  de ese `{Core}` sueltos ahí, sin una subcarpeta por proceso (un proceso
  rara vez tiene más de 1-2 archivos, a diferencia de un CRUD).

## `functions/` y `types/` — planos, sin agrupar por `{Core}`

A diferencia de `procedures/`, tanto `functions/` como `types/` quedan
**planos**, sin agrupación por carpeta de negocio — son utilidades
reutilizables entre cores, no atadas a una sola entidad:

- `functions/fn_svf_{Concepto}.sql` (escalar) / `fn_tvf_{Concepto}.sql` (tabla).
- `types/TT_{Concepto}.sql` — table types (TVP), ej. `TT_ListaEnteros`
  para pasar un array de IDs, o `TT_{Entidad}Bulk` para un insert masivo.

## `seeds/` — la única excepción a "obligatorio"

Persistir un seed **requiere confirmación explícita del usuario** antes de
crear el archivo — nunca asumir que sí ni que no, en ningún sentido.

Caso típico donde preguntar: el usuario pide cargar una tabla de catálogo
con datos de referencia desde una fuente externa (ej. una lista de
valores tipo ubigeo, o un Excel con datos maestros) — pregunta si esa
carga debe quedar también como `seeds/{entidad}_seed.sql` para poder
reproducirla en otros ambientes.
