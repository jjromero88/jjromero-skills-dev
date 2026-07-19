# Project Structure

## Carpetas de primer nivel (`src/app/`)

```
src/app/
├── core/                  → transversal, eager, se carga siempre
│   ├── guards/            → auth.guard.ts, guest.guard.ts (funcionales)
│   ├── interceptors/      → auth.interceptor.ts (funcional)
│   ├── layout/            → shell/, sidebar/, topbar/ (un folder por componente)
│   ├── models/            → api-response.model.ts, auth.model.ts
│   └── services/          → api.service.ts, auth.service.ts, layout.service.ts, theme.service.ts
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

Regla de ubicación: si algo se usa en 2+ features, sube a `shared/`; si es
transversal a toda la app (auth, layout, http), va en `core/`. No existen
carpetas `pipes/`/`directives/`/`base/` por defecto — no se crean hasta que
haya un caso real que las necesite (ver `component-patterns.md` sobre por
qué no se fuerza una base class de antemano).

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
  environment.prod.ts    → prod, apiUrl: '/api'
```

`ApiService` lee `environment.apiUrl` una sola vez al construirse y
prefija cada request con ese valor.

**Verifica siempre** que `angular.json` tenga un `fileReplacements` en la
configuración `production` que reemplace `environment.ts` por
`environment.prod.ts` — sin esa entrada, un build de producción sigue
apuntando al backend de desarrollo. No es automático solo por crear el
archivo `environment.prod.ts`.

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
