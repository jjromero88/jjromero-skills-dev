---
name: angular-feature-architecture
description: Genera frontend Angular feature-based con estas convenciones — standalone components (sin NgModules), Signals, lazy loading multi-nivel, Reactive Forms + PrimeNG obligatorio, ApiService unificado, Tailwind sin SCSS. Úsala para crear un workspace Angular o una feature/entidad nueva siguiendo estas convenciones.
---

# Angular Feature Architecture

> ⚠️ Skill personal, no genérica de Angular. No aplicar a proyectos que no
> sigan este estilo.

Basada en un frontend Angular real, revisado a fondo el 2026-07-18 leyendo
directamente el código fuente (no la documentación interna del proyecto,
que en varios puntos describe patrones aspiracionales no implementados).
Ante conflicto entre esta skill y tu código real, gana el código real.

## Motor

- Angular (última mayor estable), **100% standalone** — cero `NgModule`.
- Arquitectura feature-based: `core/` (transversal, eager) + `features/`
  (por dominio de negocio, lazy) + `shared/` (reutilizable, eager).
- Sin ORM ni Data Access propio — consume una API vía `HttpClient` envuelto
  en un `ApiService` único (ver `references/service-and-http-patterns.md`).
- Entidades agrupadas por carpeta de negocio — ver skill
  `business-domain-grouping` para el mapeo esquema↔carpeta, aplicado aquí a
  `features/{core}/`.

## Decisiones persistentes entre sesiones

Antes de "Antes de crear", verifica si existe `.claude/skill-decisions.md`
en el proyecto:
- Sección **`## Transversal`** → si ya tiene "Idioma de nomenclatura", no
  la vuelvas a preguntar (es compartida con `sql-database-patterns` y
  `dotnet-clean-architecture` — la fija la primera skill que se use en el
  proyecto).
- Sección **`## angular-feature-architecture`** → si existe, léela y
  aplica esas decisiones (versión, prefijo, librería UI, librería de
  iconos) directamente.
- Si ninguna existe todavía, es la primera vez en este proyecto: haz las
  preguntas obligatorias de abajo y crea/completa ambas secciones.

Es un archivo del **proyecto que usa la skill** (no de la skill en sí),
para que una sesión nueva (otra terminal, otro IDE) continúe sobre lo ya
decidido en vez de volver a preguntar todo como si fuera la primera vez.
Formato:

```markdown
## Transversal

- Idioma de nomenclatura: {español|inglés}

## angular-feature-architecture

- Versión de Angular: {versión}
- Prefijo/nombre del proyecto: {prefijo}
- Librería UI: {PrimeNG|otra}
- Librería de iconos: {Boxicons|otra} — versión: {versión}

### Excepciones por entidad
- {Entidad}: {excepción puntual}
```

Append-only: una excepción nueva se agrega a la lista, nunca se reescribe
una decisión ya tomada salvo que el usuario pida explícitamente cambiarla.

## Antes de crear (obligatorio)

Antes de generar un workspace nuevo, pregunta siempre las 5 — nunca asumas
ninguna, son obligatorias antes de crear cualquier archivo:
1. **Versión de Angular** — sugiere por defecto la última estable, pero
   pregunta si el usuario quiere esa u otra.
2. **Prefijo/nombre del proyecto** (para selectores de componente y el
   workspace, ej. `tu-app`).
3. **Librería de widgets UI** — sugiere PrimeNG (última estable compatible
   con la versión de Angular elegida) por defecto, pero confirma siempre;
   si el usuario prefiere otra, úsala.
4. **Librería de iconos** — sugiere Boxicons por defecto, confirma siempre;
   si confirma Boxicons, pregunta también qué versión usar; si prefiere
   otra librería, úsala.
5. **¿Idioma de nomenclatura?** (si no está ya fijado en `## Transversal`)
   Español (default) o inglés — aplica a las propiedades de los modelos
   TypeScript (`producto_id`→`product_id`, `estado`→`status`, etc., mismo
   `snake_case`, solo cambia la palabra — ver equivalencias en
   `sql-database-patterns`) y a los comentarios de código. **No afecta el
   texto de la UI** (labels, botones, notificaciones) — eso es un idioma
   de producto/UX aparte, no una decisión de esta skill.

## Antes de un CRUD nuevo (obligatorio)

Antes de construir el CRUD de una entidad, pregunta siempre: **¿modal o
vista independiente?** Puedes sugerir un default según el volumen/
complejidad de los datos (ej. una entidad con muchos campos o
sub-secciones sugiere vista independiente), pero nunca lo asumas sin
preguntar — ver `references/component-patterns.md`.

## Workflow

1. ¿Workspace nuevo desde cero? → `references/project-structure.md` (estructura de carpetas, `environments`, `angular.json`).
2. ¿Feature/entidad nueva? → sigue el checklist de abajo, un archivo por
   responsabilidad: `references/service-and-http-patterns.md`,
   `references/component-patterns.md`, `references/forms-and-validation.md`,
   `references/routing-and-lazy-loading.md`. Usa `business-domain-grouping`
   para decidir la carpeta `{Core}`.
3. ¿Auth, guards, o JWT? → `references/auth-and-security.md`.
4. ¿Estilos, tema PrimeNG, o componentes compartidos? → `references/styling-and-shared-ui.md`.
5. Siempre → nomenclatura de `references/project-structure.md`.

## Reglas duras

- **Sin NgModules.** Todo componente/directiva/pipe es standalone.
- **Signals para estado local** (`signal()`/`computed()`) — nunca
  `BehaviorSubject` para estado de UI. RxJS se reserva para el flujo
  async/HTTP (`Observable`, `finalize`, `forkJoin`, `catchError`).
- **`inject()` en vez de constructor DI**, en todos los componentes,
  servicios, guards e interceptors.
- **`OnPush`** en todo componente.
- **Control flow nuevo** (`@if`/`@for`/`@else`/`@switch`) — nunca
  `*ngIf`/`*ngFor`.
- **Un servicio por entidad**, inyectando siempre `ApiService` — nunca
  `HttpClient` directo en un componente ni en un servicio de feature.
- **Reactive Forms únicamente** para captura de datos (`FormBuilder`,
  `FormGroup`, `Validators`). `[(ngModel)]` solo se acepta para estado de UI
  suelto (ej. un input de búsqueda), nunca para un formulario de alta/edición.
- **Todo input de formulario usa un componente/directiva de PrimeNG**
  (`pInputText`, `p-select`, `p-datepicker`, etc.) — nunca un `<input>`/
  `<select>` nativo; PrimeNG pierde el estilo de error en inválido si no.
- **Guards e interceptors son funcionales** (`CanActivateFn`,
  `HttpInterceptorFn`) — nunca basados en clase.
- **Sin barrel files** (`index.ts`) en ningún nivel — imports directos por
  ruta relativa al archivo.
- **IDs como `string` opaco** en todo modelo TypeScript — el frontend nunca
  encripta/desencripta nada; ese es un límite exclusivo del backend (ver
  `references/auth-and-security.md`).
- **Tailwind CSS, sin SCSS** — cero archivos `.scss` en el proyecto. Un CSS
  por componente solo se acepta como excepción real y puntual (ej. una
  transición de `grid-template-rows` que Tailwind no expresa bien) — no es
  el patrón por defecto.
- **Sufijo `.component.ts`** en todo componente de feature, co-ubicado con
  su `.html` en su propia carpeta.
- **Reutilización obligatoria e inquebrantable.** Antes de crear cualquier
  componente, servicio, pipe o bloque de estilo, revisa si ya existe algo
  reutilizable. Se promueve a `shared/` en cuanto 2+ puntos de la app lo
  necesitan — un componente, un servicio, un pipe, o incluso un bloque de
  estilo Tailwind que se repite. Cero duplicación de código como regla, no
  como aspiración.
- **Prohibido el hardcode.** Ni una lista estática va inline en un
  componente — toda lista/constante de negocio vive centralizada en un
  servicio de `shared/`.
- **Create y edit son siempre el mismo componente** (modal o página) cuando
  sea técnicamente posible — nunca un diálogo separado para crear y otro
  para editar.
- **`environments/` con 3 entornos**: `environment.ts` (dev),
  `environment.qa.ts`, `environment.prod.ts`.

## Checklist: agregar una feature/entidad nueva

`{Core}` = carpeta de negocio (ver skill `business-domain-grouping`).

1. `features/{core}/{entidad}/models/{entidad}.model.ts` — `Entidad`,
   `EntidadInsDto`, `EntidadUpdDto` — `references/service-and-http-patterns.md`.
2. `features/{core}/{entidad}/services/{entidad}.service.ts` — `references/service-and-http-patterns.md`.
3. `features/{core}/{entidad}/pages/{entidad}-overview/` +
   `components/{entidad}-list/` o `pages/{entidad}-form/` (según la
   variante que corresponda) — `references/component-patterns.md`.
4. `features/{core}/{entidad}/components/{entidad}-fields/` (formulario) —
   `references/forms-and-validation.md`.
5. `features/{core}/{entidad}/{entidad}.routes.ts` (`{ENTIDAD}_ROUTES`) —
   `references/routing-and-lazy-loading.md`.
6. Registrar la ruta en `features/{core}/{core}.routes.ts` (`loadChildren`) —
   `references/routing-and-lazy-loading.md`.
7. Si la feature es nueva a nivel de dominio, registrar también en
   `app.routes.ts` — `references/routing-and-lazy-loading.md`.

## Referencias

- `references/project-structure.md` — `core/features/shared`, `environments`, convenciones de nombres.
- `references/routing-and-lazy-loading.md` — lazy loading multi-nivel, guards funcionales, rutas.
- `references/component-patterns.md` — standalone/OnPush/Signals, variantes de split overview/list/fields.
- `references/forms-and-validation.md` — Reactive Forms, helpers de validación, PrimeNG, campos de fecha.
- `references/service-and-http-patterns.md` — `ApiService`, servicio por entidad, `ApiResponse`, modelos/DTOs, catálogos con cache.
- `references/auth-and-security.md` — JWT, login multi-tenant en 2 fases, guards, IDs opacos.
- `references/styling-and-shared-ui.md` — Tailwind, tema PrimeNG, `shared/ui`, iconos.
- Skill `business-domain-grouping` — mapeo esquema↔carpeta de negocio (`{Core}`), aplicado a `features/`.
- Skill `angular-design-system` — todo lo visual: colores, tipografía, dark
  mode, botones/cards/alerts/modales, grillas, notificaciones, KPIs y
  charts.
