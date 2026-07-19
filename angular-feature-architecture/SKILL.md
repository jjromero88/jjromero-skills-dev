---
name: angular-feature-architecture
description: Genera frontend Angular feature-based al estilo de Juan José Romero — standalone components (sin NgModules), Signals, lazy loading multi-nivel, Reactive Forms + PrimeNG obligatorio, ApiService unificado, Tailwind sin SCSS. Úsala para crear un workspace Angular o una feature/entidad nueva siguiendo estas convenciones.
---

# Angular Feature Architecture — Estilo Juan José Romero

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

## Antes de crear (obligatorio)

Antes de generar un workspace nuevo, pregunta siempre:
1. **Versión de Angular** — sugiere por defecto la última estable, pero
   pregunta si el usuario quiere esa u otra.
2. **Prefijo/nombre del proyecto** (para selectores de componente y el
   workspace, ej. `tu-app`).

No asumas ninguna de las dos — son obligatorias antes de crear cualquier
archivo.

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
