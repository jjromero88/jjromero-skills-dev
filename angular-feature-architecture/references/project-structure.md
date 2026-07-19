# Project Structure

## Carpetas de primer nivel (`src/app/`)

```
src/app/
├── core/                  → transversal, eager, se carga siempre
│   ├── guards/            → auth.guard.ts, guest.guard.ts (funcionales)
│   ├── interceptors/      → auth.interceptor.ts (funcional)
│   ├── layout/            → shell/, sidebar/, topbar/ (un folder por componente)
│   ├── models/            → api-response.model.ts, auth.model.ts (o models/{api}/ si hay 2+ APIs, ver nota abajo)
│   └── services/          → api.service.ts, auth.service.ts, layout.service.ts, theme.service.ts (o services/{api}/ si hay 2+ APIs)
├── features/              → por dominio de negocio, lazy
│   ├── auth/
│   ├── dashboard/
│   └── {core}/            → una carpeta por dominio (ver business-domain-grouping)
│       └── {entidad}/
│           ├── models/
│           ├── services/
│           ├── pages/
│           ├── components/
│           └── {entidad}.routes.ts
├── shared/                → reutilizable entre features, eager
│   ├── animations/        → `trigger()` de @angular/animations (ej. expand-row)
│   ├── helpers/           → funciones puras (ej. form-helpers.ts)
│   ├── models/            → catálogo, paginación, notificación
│   ├── services/          → catalogos.service.ts (cache), notification.service.ts
│   └── ui/                → componentes presentacionales genéricos (kpi-card, notification-dialog, etc.)
├── app.config.ts          → ApplicationConfig: router, http, tema PrimeNG, animaciones
├── app.routes.ts          → tabla de rutas de primer nivel
└── app.component.ts        → raíz, monta el shell + `<router-outlet />` + diálogo de notificación global
```

**`core/models/` y `core/services/` — una sola API vs. varias**: si la app
consume una única API backend, quedan planos como en el árbol de arriba. Si
consume 2+ APIs backend distintas, se agrupan en subcarpetas por nombre de
API (`core/models/{api}/`, `core/services/{api}/`) — cada una con su propio
`ApiResponse`/`ApiService` si sus contratos difieren.

Regla de ubicación: si algo se usa en 2+ features, sube a `shared/`; si es
transversal a toda la app (auth, layout, http), va en `core/`. No existe
`pipes/`/`base/` por defecto — no se crean hasta que haya un caso real que
las necesite (ver `component-patterns.md` sobre por qué no se fuerza una
base class de antemano). **`shared/directives/` sí forma parte del
esqueleto inicial** — aloja la directiva de conteo animado para KPIs (ver
skill `angular-design-system`, `references/kpi-cards-and-charts.md`).

**Reutilización obligatoria e inquebrantable** (ver también `SKILL.md`):
antes de crear un componente, servicio, pipe o bloque de estilo, revisa si
ya existe algo reutilizable — se promueve a `shared/` en cuanto 2+ puntos
de la app lo necesitan. Ni una lista estática va hardcodeada en un
componente: toda lista/constante de negocio vive centralizada en un
servicio de `shared/services/`.

## Entidad dentro de una feature

Caso simple (una sola entidad, sin sub-secciones):

```
features/{core}/{entidad}/
├── models/{entidad}.model.ts
├── services/{entidad}.service.ts
├── components/{entidad}-fields/       → o {entidad}-form/, según la variante (ver component-patterns.md)
├── pages/{entidad}-overview/
└── {entidad}.routes.ts
```

Caso complejo (una entidad con sub-secciones propias, ej. un detalle con
varias pestañas/paneles independientes): cada sub-sección vive dentro de
`components/` de la entidad padre, como una mini-feature autónoma con su
propio `models/`/`services/` local:

```
features/{core}/{entidad}/
├── models/{entidad}.model.ts
├── services/{entidad}.service.ts
├── pages/{entidad}-overview/, pages/{entidad}-form/
└── components/
    └── {seccion}/
        ├── models/{seccion}.model.ts     ← local a la sección, no sube al nivel de la entidad
        ├── services/{seccion}.service.ts
        ├── {seccion}-list/
        └── {seccion}-fields/
```

No subas un modelo/servicio de sección al nivel de la entidad padre solo
por prolijidad — si es exclusivo de esa sección, se queda anidado.

## `environments/`

```
src/environments/
  environment.ts        → dev, apiUrl: 'http://localhost:{puerto}/api'
  environment.qa.ts      → qa, apiUrl del ambiente de pruebas
  environment.prod.ts    → prod, apiUrl: '/api'
```

3 entornos siempre: dev, qa y prod — no solo dev/prod.

`ApiService` lee `environment.apiUrl` una sola vez al construirse y
prefija cada request con ese valor.

**Verifica siempre** que `angular.json` tenga un `fileReplacements` en cada
configuración (`production` y `qa`) que reemplace `environment.ts` por el
archivo correspondiente (`environment.prod.ts` / `environment.qa.ts`) — sin
esa entrada, un build sigue apuntando al backend de desarrollo. No es
automático solo por crear el archivo.

## Convenciones de nombres

| Elemento | Convención | Ejemplo |
|---|---|---|
| Carpetas | kebab-case, plural para colecciones | `productos/`, `clientes/`, `guards/` |
| Componente | `{nombre}.component.ts` + `.html` co-ubicados en su propia carpeta | `producto-list/producto-list.component.ts` |
| Archivo de rutas | `{scope}.routes.ts`, exporta un `Routes` const en SCREAMING_SNAKE_CASE | `PRODUCTOS_ROUTES` |
| Modelo | `{entidad}.model.ts`, singular | `producto.model.ts` |
| Servicio | `{entidad}.service.ts`, una clase por archivo, `@Injectable({providedIn:'root'})` | `productos.service.ts` |
| Campos privados de clase | prefijo `_` | `_api`, `_notif`, `_fb` |
| Interfaces/modelos | PascalCase | `Producto`, `ProductoInsDto` |
| Constantes de rutas/listas | UPPER_SNAKE_CASE | `PRODUCTOS_ROUTES`, `PRODUCTO_DATE_FIELDS` |

No hay barrel files (`index.ts`) en ningún nivel — todo import es una ruta
relativa directa al archivo, aunque sea profunda.
