# jjromero-skills-dev

Colección personal de [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) para Claude Code — convenciones de arquitectura y código, por stack tecnológico, tal como las aplico en mis propios proyectos.

> ⚠️ **No son skills genéricas.** Cada una documenta un estilo personal y opinionado (nomenclatura, estructura de carpetas, patrones de código) validado contra proyectos reales propios — no un estándar de la industria. Si tu proyecto no sigue este estilo, no las apliques tal cual; úsalas como referencia y adáptalas.

## Skills disponibles

| Skill | Stack | Qué genera |
|---|---|---|
| [`sql-database-patterns`](./sql-database-patterns) | SQL Server / PostgreSQL | Stored procedures y DDL: esquemas por módulo, CRUD vía 5 SPs con parámetros `OUTPUT`, secuencia fija de validaciones, auditoría, soft delete, catálogo genérico, IDs encriptados. |
| [`dotnet-clean-architecture`](./dotnet-clean-architecture) | .NET | Backend Clean Architecture: 10 proyectos, Dapper + stored procedures (sin EF), DTOs por operación CRUD, `ApiResponse` unificado, IDs encriptados, validación centralizada, auditoría desde JWT. |
| [`business-domain-grouping`](./business-domain-grouping) | Transversal (BD + backend + frontend) | Método para derivar carpetas de negocio desde los esquemas de BD y aplicarlas consistentemente en todas las capas. |

Más skills se irán agregando a medida que documente otros stacks (Angular, PostgreSQL puro, etc.).

## Cómo usarlas

Cada carpeta es una skill independiente: un `SKILL.md` de entrada (nombre, descripción, reglas duras) más una carpeta `references/` con el detalle por tema.

Para usar una en un proyecto con Claude Code:

1. Copia la carpeta de la skill dentro de `.claude/skills/` en tu proyecto (o `~/.claude/skills/` si la quieres disponible en todos tus proyectos).
2. Pídele a Claude que la use — por ejemplo: *"usa la skill `dotnet-clean-architecture` para crear el CRUD de la entidad Producto"*.
3. Revisa el resultado contra tu propio código real antes de asumir que aplica tal cual — estas skills están pensadas para adaptarse, no para copiarse a ciegas.

## Estructura de una skill

```
nombre-skill/
  SKILL.md              → entrypoint: workflow, reglas duras, tabla de referencias
  references/
    tema-a.md           → detalle y templates de código de un tema puntual
    tema-b.md
```

## Filosofía

- **Código real como fuente de verdad.** Cada regla se verificó contra proyectos propios reales, no se inventó desde la teoría. Ante conflicto entre una idea y el código, gana el código.
- **Un archivo por responsabilidad.** Nada de archivos gigantes que mezclan capas — cada `references/*.md` cubre un tema para poder cargar solo lo necesario.
- **Ejemplos genéricos.** Los ejemplos usan dominios de negocio inventados (ventas, pedidos, productos) — ninguna skill depende de que exista un proyecto específico como referencia viva.

## Contribuir

Es una colección de mantenimiento personal, pero los PRs son bienvenidos — sobre todo correcciones, o skills nuevas de otros stacks que sigan el mismo formato (`SKILL.md` + `references/`).

## Licencia

[MIT](./LICENSE) — úsalas, cópialas, adáptalas libremente.
