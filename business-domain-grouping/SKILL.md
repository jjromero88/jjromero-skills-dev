---
name: business-domain-grouping
description: Método de agrupación lógica de entidades de negocio al estilo de Juan José Romero — cómo derivar carpetas de negocio desde los esquemas de BD, y aplicarlas consistentemente en backend (.NET) y frontend (Angular). Consúltala junto con sql-database-patterns/dotnet-clean-architecture/angular-* para saber en qué carpeta de negocio va cada entidad nueva.
---

# Business Domain Grouping — Estilo Juan José Romero

> ⚠️ Skill personal, transversal a BD/backend/frontend.

Un proyecto puede tener 100+ entidades de negocio. Sin agrupación lógica
consistente, es difícil navegar el código. Esta skill define el MÉTODO para
derivar carpetas de negocio ("cores") a partir de los esquemas de BD — para
que BD, backend y frontend usen el mismo vocabulario. Reaplica este método
en cada proyecto nuevo; no reutilices nombres de core de un proyecto anterior
si el dominio de negocio es distinto.

## Regla de decisión

1. Cada esquema de BD (ver `sql-database-patterns`) se convierte en una
   carpeta de negocio con el mismo nombre conceptual, en el idioma del
   proyecto.
2. **Si un esquema agrupa dos responsabilidades de negocio claramente
   distintas**, se parte en dos carpetas en vez de una — el límite técnico
   del esquema no obliga a una única carpeta. Ejemplo del criterio: un
   esquema de seguridad que mezcla "cuentas de usuario/autenticación" con
   "configuración de roles y permisos (RBAC)" son dos preocupaciones
   distintas — una carpeta para cada una, aunque vivan en el mismo esquema.
   Mismo criterio para un esquema de "datos maestros" que mezcla entidades
   propias de un módulo con catálogos genéricos reutilizados por varios
   módulos — se separan.
3. La carpeta de negocio (`{Core}`) NUNCA afecta el namespace/paquete —
   solo organiza carpetas físicas.

## Dónde aplica esta agrupación

- BD: el esquema mismo (`sql-database-patterns`) — 1:1 con el core salvo
  cuando la regla 2 aplica.
- Backend: `Persistence/Repositories/{Core}/`, `Application/Services/{Core}/`,
  `Application/Interfaces/{Repository,Service}/{Core}/`,
  `Application/DTOs/{Core}/`, `Validator/Validators/{Core}/`
  (`dotnet-clean-architecture`).
- Frontend: carpeta de feature de primer nivel (`features/{core}/`), cuando
  se ajuste la skill de Angular.

## Excepción: `Mapper/Profiles` es plano

Los perfiles de AutoMapper (`Mapper/Profiles/{Entidad}Profile.cs`) NO se
agrupan por core — quedan planos por entidad, sin excepción.

## Ejemplo aplicado (sistema de ventas)

Así se aplicaría el método en un sistema de ventas típico, con esquemas
`seg` (seguridad), `ven` (ventas), `inv` (inventario), `fac` (facturación),
`cat` (catálogo):

| Carpeta | Esquema(s) | Por qué se separó así |
|---|---|---|
| `Auth` | `seg` (subset) | Cuentas de usuario y autenticación — distinto de la configuración RBAC |
| `Seguridad` | `seg` (subset) | Roles, permisos, perfiles (RBAC) |
| `Ventas` | `ven` | 1:1 — pedidos, líneas de pedido, clientes |
| `Inventario` | `inv` | 1:1 — productos, stock, almacenes |
| `Facturacion` | `fac` | 1:1 — facturas, pagos |
| `Catalogo` | `cat` | 1:1 — catálogos genéricos (unidades de medida, monedas, categorías) |

`seg` se parte en dos carpetas (regla 2 — autenticación vs. RBAC son
responsabilidades distintas aunque vivan en el mismo esquema);
`ven`/`inv`/`fac`/`cat` quedan 1:1 con su esquema (regla 1). Un gap típico
en frontend: si no se aplica el método completo ahí, `Catalogo` termina
anidado dentro de otro feature (ej. dentro de `Ventas`) en vez de tener su
propia carpeta de feature — aplica el método completo también en frontend,
no lo simplifiques a mitad de camino.

## Agregar una entidad nueva: elegir su core

1. ¿A qué esquema de BD pertenece la tabla?
2. ¿Ese esquema agrupa una sola responsabilidad de negocio, o dos claramente
   distintas? Si es una sola → la carpeta = el esquema. Si son dos → decide
   a cuál de las dos pertenece la entidad nueva, o si amerita partir el
   esquema en dos carpetas (regla de decisión, arriba).
