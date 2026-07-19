# Routing and Lazy Loading

## Lazy loading multi-nivel (3 niveles)

```
app.routes.ts               → nivel 1: shell autenticado + loadChildren por dominio
  features/{core}/{core}.routes.ts   → nivel 2: sin componente propio, solo loadChildren por entidad
    features/{core}/{entidad}/{entidad}.routes.ts  → nivel 3: hoja, loadComponent por página
```

`core/` y `layout/` siempre eager. `features/` siempre lazy — sin
excepciones.

### Nivel 1 — `app.routes.ts`

```ts
export const routes: Routes = [
  { path: 'auth', loadChildren: () => import('./features/auth/auth.routes').then(m => m.AUTH_ROUTES) },
  {
    path: '',
    component: ShellComponent,
    canActivate: [authGuard],
    children: [
      { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
      { path: 'dashboard', loadComponent: () => import('./features/dashboard/pages/dashboard-overview/dashboard-overview.component').then(m => m.DashboardOverviewComponent) },
      { path: 'ventas', loadChildren: () => import('./features/ventas/ventas.routes').then(m => m.VENTAS_ROUTES) },
      { path: 'inventario', loadChildren: () => import('./features/inventario/inventario.routes').then(m => m.INVENTARIO_ROUTES) },
    ],
  },
];
```

### Nivel 2 — `{core}.routes.ts` (sin componente propio)

```ts
export const VENTAS_ROUTES: Routes = [
  { path: '', redirectTo: 'pedidos', pathMatch: 'full' },
  { path: 'pedidos', loadChildren: () => import('./pedidos/pedidos.routes').then(m => m.PEDIDOS_ROUTES) },
  { path: 'clientes', loadChildren: () => import('./clientes/clientes.routes').then(m => m.CLIENTES_ROUTES) },
];
```

### Nivel 3 — `{entidad}.routes.ts` (hoja)

```ts
export const PEDIDOS_ROUTES: Routes = [
  { path: '', redirectTo: 'overview', pathMatch: 'full' },
  { path: 'overview', loadComponent: () => import('./pages/pedidos-overview/pedidos-overview.component').then(m => m.PedidosOverviewComponent) },
  { path: 'nuevo', loadComponent: () => import('./pages/pedido-form/pedido-form.component').then(m => m.PedidoFormComponent) },
  { path: ':id', loadComponent: () => import('./pages/pedido-form/pedido-form.component').then(m => m.PedidoFormComponent) },
];
```

Rutas de página propia (`nuevo`/`:id`) solo aplican a la variante de página
enrutada del split de componentes (ver `component-patterns.md`) — si la
entidad usa la variante de diálogo, no existen estas rutas hijas.

Cada archivo de rutas exporta un `Routes` const en **SCREAMING_SNAKE_CASE**
que coincide con su dominio/entidad (`VENTAS_ROUTES`, `PEDIDOS_ROUTES`) —
convención estricta, sin excepciones observadas.

## Guards — siempre funcionales

```ts
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  if (!auth.isAuthenticated() || auth.isTokenExpired()) {
    auth.logout();
    router.navigate(['/auth/login']);
    return false;
  }
  return true;
};

export const guestGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  if (auth.isAuthenticated() && !auth.isTokenExpired()) {
    router.navigate(['/dashboard']);
    return false;
  }
  return true;
};
```

`guestGuard` se usa en las rutas de `auth/` para evitar que un usuario ya
logueado vuelva a `/auth/login`. Nunca un guard basado en clase
(`CanActivate` de interfaz) — siempre `CanActivateFn` con `inject()`.

## Interceptor de autenticación

Un único interceptor funcional (`core/interceptors/auth.interceptor.ts`)
hace dos cosas, y solo esas dos:

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.token();
  const authReq = token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;
  return next(authReq).pipe(
    catchError(err => {
      if (err.status === 401) auth.logout();
      return throwError(() => err);
    }),
  );
};
```

No hay interceptor global de toasts de error ni de loading spinner — el
manejo de error y la notificación se hacen manualmente por componente (ver
`service-and-http-patterns.md`). No lo agregues salvo que el usuario lo
pida explícitamente; no es el patrón real actual.
