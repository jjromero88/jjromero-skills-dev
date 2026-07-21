---
name: sql-database-patterns
description: Genera stored procedures y DDL para SQL Server (o PostgreSQL) con estas convenciones — esquemas por módulo de negocio, CRUD vía 5 stored procedures con parámetros OUTPUT, secuencia fija de validaciones, auditoría, soft delete, catálogo genérico e IDs encriptados. Úsala cuando se pida crear tablas, procedures CRUD, o cualquier objeto T-SQL siguiendo estas convenciones.
---

# SQL Database Patterns

> ⚠️ Skill personal, no genérica de SQL Server. No aplicar a proyectos que no
> sigan este estilo.

Basada en un proyecto real de SQL Server con Dapper, revisado a fondo el
2026-07-18. Ante conflicto entre esta skill y tu código real, gana el
código real.

## Motor

- SQL Server (corporativo) o PostgreSQL (nuevos) — mismo patrón, equivalencias
  de sintaxis en `references/ddl-conventions.md`.
- Sin ORM. Prohibido Entity Framework. Solo Stored Procedures vía Dapper.

## Antes de crear (obligatorio)

Antes de generar cualquier tabla/procedure/function, pregunta siempre —
nunca asumas, no generes DDL/SP sin esta respuesta:
1. **¿Es sobre una base de datos ya existente, o hay que crear una
   nueva?**
2. Si es una BD nueva, pide el **nombre de la base de datos**.

El nombre alcanza para generar el DDL/SPs — no hace falta pedir datos de
conexión (server, docker, VPS, credenciales) en este punto. Esos datos
solo se solicitan más adelante, si hace falta ejecutar algo de verdad (ver
"Plan de ejecución — obligatorio con este trigger" en
`references/ddl-conventions.md`).

Aplica la primera vez que se usa la skill en la conversación — no hace
falta repreguntar en cada tabla subsiguiente si el contexto de la BD ya
quedó establecido.

## Workflow

1. ¿Tabla nueva? → `references/ddl-conventions.md`
2. ¿CRUD de una entidad? → `references/sp-templates.md`
3. ¿Proceso/acción de negocio o function? → `references/procedure-and-function-patterns.md`
4. ¿Ambas (tabla + procedures)? → tabla primero.
5. Siempre → secuencia de `references/validation-patterns.md` antes de todo DML.
6. ¿Passwords, IDs expuestos, dato sensible? → `references/encryption-patterns.md`

## Esquemas por módulo de negocio

| Esquema | Módulo | Ejemplo de tablas |
|---|---|---|
| `seg` | Seguridad (RBAC) | USUARIO, PERFIL, PERMISOS, SISTEMAOPCION, SOCIEDAD |
| `per` | Personal / RRHH | EMPLEADO, CONTRATO, DERECHOHABIENTE, CATALOGO |
| `nom` | Nómina | CONCEPTO, FORMULA, PARAMETRO_CALCULO, TARIFA_AFP |
| `cnt` | Contabilidad | CENTRO_COSTO, ASIENTO, PLAN_CUENTAS |
| `evt` | Eventos | EVENTO_REMUNERATIVO, FALTA, SUBSIDIO |
| `aud` | Auditoría/Transversal | configuraciones, logs de auditoría |

Nunca `dbo` — toda tabla/procedure/function va en su propio esquema de negocio.

PostgreSQL: mismo patrón con schemas nativos.

## Tabla de acciones de Stored Procedure

| Acción | Propósito | DML | Ejemplo |
|---|---|---|---|
| `Sel` | Listar activos | Ninguno | `Sp_Sel_Empleado` |
| `Get` | Por ID (sin filtrar `estado`) | Ninguno | `Sp_Get_Empleado` |
| `Ins` | Insertar | INSERT | `Sp_Ins_Empleado` |
| `Upd` | Actualizar | UPDATE | `Sp_Upd_Empleado` |
| `Del` | Eliminación lógica | UPDATE | `Sp_Del_Empleado` |
| `Ver` | Chequeo de apoyo, solo lectura | Ninguno (`#temp` ok) | `Sp_Ver_PreCalculo` |
| `Acc` | Acción principal invocada por el frontend | Variable | `Sp_Acc_Generar_Feriado` |

`Ver` vs `Acc` es semántico, no técnico: `Acc` es lo que el usuario pidió;
`Ver` es un chequeo de apoyo de otro flujo. Detalle en
`references/procedure-and-function-patterns.md`.

**Sufijos de variante**: `_Bulk` (masivo), `_Upsert` (insert-or-update),
`_By{Entidad}` (filtrado por padre), `_Auto` (derivado de otro proceso).

## Functions

`fn_svf_{Concepto}` (escalar) / `fn_tvf_{Concepto}` (tabla). Reglas y
templates en `references/procedure-and-function-patterns.md`.

## Reglas duras

- Nunca `DELETE FROM` — siempre `UPDATE ... SET estado = 0`.
- Salida siempre por `OUTPUT` (`@error BIT`, `@msg VARCHAR(500)`) — nunca `SELECT` de resultado.
- Todo `Sp_Ins` retorna además `@{entidad}_id INT OUTPUT` (`SCOPE_IDENTITY()`), sin excepción.
- `Sp_Sel_*` filtra `estado = 1`, ordena por PK.
- `Sp_Get_*` NUNCA filtra por `estado`.
- `Sp_Upd_*` nunca toca `usuario_reg`/`fecha_reg`.
- `SET NOCOUNT ON;` + `SET @error=0; SET @msg='';` antes del `TRY`.
- Todo el cuerpo en `TRY/CATCH`, nunca `THROW` sin capturar. `ROLLBACK` solo
  aplica si hay `BEGIN TRANSACTION` explícito (multi-sentencia) — un solo
  INSERT/UPDATE ya es atómico y no lo necesita.
- Mensajes siempre en español.
- `codigo` (en cualquier tabla que tenga esta columna, no solo `CATALOGO`):
  se autogenera por defecto a partir del PK identity, zero-padded a 5
  dígitos (`1` → `'00001'`) — obligatorio salvo que el usuario pida
  explícitamente un formato distinto para esa entidad. **Solo se genera en
  `Sp_Ins`** — `Sp_Upd` nunca lo recibe como parámetro ni lo modifica por
  defecto (es un valor de solo-identificación para el usuario final, que
  nunca ve el PK; no es un dato de negocio editable), salvo que el usuario
  pida explícitamente que sea editable. Mecanismo completo en
  `references/ddl-conventions.md` y `references/sp-templates.md`.
- Validaciones extensibles: las 6 de `validation-patterns.md` son el piso, no
  el techo — agrega las de negocio en el mismo bloque, ordenadas por dependencia.
- Todo bloque (validación o lógica) lleva un comentario corto en español
  encima (`-- Validamos que el DNI exista`) — incluye functions. Si el bloque
  implementa un algoritmo no obvio o antecede una regla de negocio compleja,
  usa el comentario extendido tipo banner (`validation-patterns.md`).
- Tras las validaciones, el cuerpo/lógica va envuelto en `IF (@error = 0)
  BEGIN ... END` — capa defensiva además del `RETURN` de cada validación. No
  aplica a functions (no admiten `OUTPUT`).
- Cabecera de documentación (`/* Procedimiento/Funcion, Descripcion, Autor,
  Fecha, INPUTS, OUTPUTS */`) obligatoria encima de TODO procedure y function,
  sin excepción.
- `_Bulk` invoca el `Sp_Ins_{Entity}`/`Sp_Upd_{Entity}` existente (loop sobre
  tabla temporal/TVP) en vez de reimplementar el INSERT/UPDATE — excepto
  cuando el volumen exige un `INSERT...SELECT` set-based por performance;
  documenta el motivo si tomas esa excepción.

## Columnas de auditoría

```sql
estado       BIT          NULL DEFAULT 1,
usuario_reg  VARCHAR(50)  NULL DEFAULT 'sys',
fecha_reg    DATETIME     NULL DEFAULT GETDATE(),
usuario_act  VARCHAR(50)  NULL,
fecha_act    DATETIME     NULL
```

Siempre en este orden, al final de la tabla. Detalle en `references/ddl-conventions.md`.

## Índices

Toda columna no-PK/FK usada en JOIN o filtro frecuente necesita un índice.
Si un `SELECT` tiene más de un JOIN y alguno compara una columna no-PK/FK, el
plan de ejecución es **obligatorio de revisar** — ejecútalo si tienes acceso
a la BD, o solicita credenciales/permiso al usuario si no lo tienes. Reporta
siempre el resultado (sin cuellos de botella, o con costo alto + índice
sugerido) — nunca en silencio. Detalle completo en `references/ddl-conventions.md`.

## Verificación tras generar CRUD

Tras crear los 5 SPs de una entidad, prueba (o sugiere probar) el ciclo
completo: insertar un registro de prueba → `Sel`/`Get` → `Upd` → `Del`. Los
procedures de proceso (`Sp_Acc_`/`Sp_Ver_`) quedan a decisión del usuario —
no se prueban automáticamente.

## Estructura de carpetas SQL

```
{proyecto}_db/
  database/
    schema_{modulo}.sql
  procedures/
    {Entidad}/
      Sp_Sel_{Entidad}.sql
      Sp_Get_{Entidad}.sql
      Sp_Ins_{Entidad}.sql
      Sp_Upd_{Entidad}.sql
      Sp_Del_{Entidad}.sql
  seeds/
    {entidad}_seed.sql
  functions/
    fn_svf_{Concepto}.sql
    fn_tvf_{Concepto}.sql
```

## Referencias

- `references/ddl-conventions.md` — tipos de dato, nomenclatura, FKs, catálogo genérico.
- `references/sp-templates.md` — 5 plantillas CRUD + variante multi-tabla con transacción.
- `references/validation-patterns.md` — secuencia de validaciones + snippets.
- `references/encryption-patterns.md` — límite BD/Service para datos sensibles.
- `references/procedure-and-function-patterns.md` — `Sp_Acc_`/`Sp_Ver_` + `fn_svf_`/`fn_tvf_`.
