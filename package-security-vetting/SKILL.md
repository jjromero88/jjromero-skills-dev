---
name: package-security-vetting
description: Verificación de seguridad obligatoria antes de instalar cualquier paquete NuGet o npm de terceros — whitelist de orígenes confiables (Microsoft, orgs verificadas, defaults ya vetados) con vía rápida, y chequeo obligatorio contra OSV.dev, GitHub Advisory Database, Socket.dev y las herramientas propias del ecosistema para todo lo demás. Úsala junto con `dotnet-clean-architecture`/`angular-feature-architecture` antes de cualquier `dotnet add package` o `npm install`.
---

# Package Security Vetting

> ⚠️ Skill personal, transversal a backend (.NET/NuGet) y frontend
> (Angular/npm). No aplica a SQL Server/PostgreSQL — ahí no hay gestor de
> paquetes de terceros en el sentido de esta skill.

Basada en herramientas y fuentes reales de la industria de seguridad de
software — OSV.dev, GitHub Advisory Database, Socket.dev, `npm audit`,
`dotnet list package --vulnerable` — investigadas y verificadas el
2026-07-27. Ante conflicto entre esta skill y el estado vigente de esas
fuentes (una API cambia de forma, un servicio deja de estar disponible),
gana la fuente oficial vigente.

## Motor

Whitelist de **2 niveles**, no scraping abierto de la web:

1. **Origen confiable** (`references/trusted-origins.md`) — paquetes de
   publicadores ya verificados o ya vetados como default en las otras
   skills → vía rápida, solo un chequeo barato de que el nombre no es un
   typosquat.
2. **Fuentes de verificación fijas** (`references/vetting-sources.md`) —
   para todo lo demás, 5 fuentes estructuradas obligatorias (nunca
   búsqueda web abierta ni scraping de páginas arbitrarias).

## Decisiones persistentes entre sesiones

Verifica si existe `.claude/skill-decisions.md` en el proyecto, sección
`## package-security-vetting` → `### Paquetes ya vetados`. Un paquete que
ya pasó el chequeo (o cuyo riesgo ya fue aceptado explícitamente por el
usuario) en este proyecto no se re-vetea cada vez que se vuelve a
mencionar en la misma sesión o en sesiones futuras. Formato:

```markdown
## package-security-vetting

### Paquetes ya vetados
- {ecosistema}/{paquete}: {resultado — "sin hallazgos" | "riesgo aceptado por el usuario el {fecha}: {resumen}"}
```

Append-only: un paquete nuevo se agrega a la lista, nunca se reescribe un
veredicto ya registrado salvo que el usuario pida explícitamente
re-evaluarlo (ej. tras una actualización de versión mayor).

## Antes de instalar un paquete nuevo (obligatorio)

Se dispara siempre que un paquete NuGet o npm de terceros esté a punto de
agregarse al proyecto — lo pida el usuario explícitamente o lo decida
Claude para implementar una feature — **incluyendo un bump de versión
mayor** de una dependencia ya instalada (un account-takeover o una versión
maliciosa inyectada en una actualización es un vector real, no solo en la
instalación inicial). Nunca instales primero y avises después.

1. ¿El paquete está en la whitelist de origen confiable? →
   `references/trusted-origins.md` — vía rápida + chequeo de typosquat.
2. ¿No está? → `references/vetting-sources.md` — las 5 fuentes
   obligatorias.
3. Siempre, con o sin hallazgos → `references/severity-and-reporting.md`
   para informar el resultado y dejar la decisión al usuario.
4. Persiste el veredicto en `.claude/skill-decisions.md` (arriba).

## Reglas duras

- **Nunca instalar en silencio.** Con o sin hallazgos, el resultado del
  chequeo se reporta siempre al usuario — igual que el principio ya usado
  en `sql-database-patterns` para el plan de ejecución.
- **Nunca scraping abierto ni búsqueda web libre.** Solo las fuentes
  fijas de `references/vetting-sources.md`.
- **El fast path no es "sin chequeo alguno"** — siempre valida que el
  nombre del paquete coincide exactamente con el paquete real conocido
  (defensa mínima contra typosquatting de un nombre confiable).
- **Un hallazgo nunca bloquea la instalación por sí solo** — se
  recomienda no instalar y se sugiere alternativa, pero la decisión final
  (instalar bajo su responsabilidad, o descartar) es siempre del usuario,
  nunca asumida por Claude en ningún sentido.
- **Sin acceso a herramientas de chequeo (web/shell) en la sesión**: decirlo
  explícitamente y preguntar al usuario cómo proceder — nunca simular que
  el chequeo se hizo.
- **Un bump de versión mayor de una dependencia existente dispara el mismo
  gate** que una instalación nueva — no es exclusivo del primer install.

## Referencias

- `references/trusted-origins.md` — whitelist de origen confiable por
  ecosistema, chequeo de typosquat, cruce con los defaults ya vetados en
  `dotnet-clean-architecture`/`angular-feature-architecture`/
  `angular-design-system`.
- `references/vetting-sources.md` — las 5 fuentes obligatorias con su
  formato exacto de consulta (OSV.dev, GitHub Advisory Database,
  `npm audit`/`dotnet list package --vulnerable`, Socket.dev, metadata del
  registro).
- `references/severity-and-reporting.md` — clasificación de severidad,
  plantilla de aviso, patrón de consentimiento "instalar bajo tu
  responsabilidad", flujo de sugerir alternativa, fallback sin acceso.
- Skill `dotnet-clean-architecture` — dispara este gate antes de
  `dotnet add package`.
- Skill `angular-feature-architecture` — dispara este gate antes de
  `npm install`.
