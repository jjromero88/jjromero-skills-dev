# Service and HTTP Patterns

## `ApiService` — wrapper único sobre `HttpClient`

Un solo servicio central envuelve `HttpClient`. Ningún componente ni
servicio de feature inyecta `HttpClient` directamente — siempre pasan por
`ApiService`.

```ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly _http = inject(HttpClient);
  private readonly _baseUrl = environment.apiUrl;

  get<T>(path: string): Observable<ApiResponse<T>> {
    return this._http.get<ApiResponse<T>>(`${this._baseUrl}${path}`).pipe(catchError(e => this._handleError(e)));
  }
  post<T>(path: string, body: unknown): Observable<ApiResponse<T>> {
    return this._http.post<ApiResponse<T>>(`${this._baseUrl}${path}`, body).pipe(catchError(e => this._handleError(e)));
  }
  put<T>(path: string, body: unknown): Observable<ApiResponse<T>> {
    return this._http.put<ApiResponse<T>>(`${this._baseUrl}${path}`, body).pipe(catchError(e => this._handleError(e)));
  }
  delete<T>(path: string): Observable<ApiResponse<T>> {
    return this._http.delete<ApiResponse<T>>(`${this._baseUrl}${path}`).pipe(catchError(e => this._handleError(e)));
  }
  deleteWithBody<T>(path: string, body: unknown): Observable<ApiResponse<T>> {
    return this._http.delete<ApiResponse<T>>(`${this._baseUrl}${path}`, { body }).pipe(catchError(e => this._handleError(e)));
  }

  private _handleError(err: HttpErrorResponse): Observable<ApiResponse<unknown>> {
    if (err.error?.success !== undefined) return throwError(() => err.error as ApiResponse<unknown>);
    const message = err.status === 0 ? 'No se pudo conectar con el servidor.' : `Error inesperado (${err.status}).`;
    return throwError(() => ({ success: false, message, data: null, errors: null } as ApiResponse<unknown>));
  }
}
```

Punto clave: **todo consumidor puede asumir que un error siempre llega ya
en forma `ApiResponse`** — nunca un `HttpErrorResponse` crudo. Esto se
logra en un único lugar (`_handleError`), no en cada llamada.

## `ApiResponse<T>` — envelope único

```ts
export interface ApiResponse<T> {
  success: boolean;
  message: string;
  data: T | null;
  errors: string[] | null;
}
```

Coincide con el `ApiResponse<T>` del backend (ver skill
`dotnet-clean-architecture`) — mismo contrato en ambos lados.

## Servicio por entidad

Un servicio por entidad, inyectando `ApiService` (nunca `HttpClient`
directo). Métodos extra a los 5 CRUD son válidos si responden a un caso de
uso real (ej. una acción de negocio puntual) — no hace falta forzarlos a
encajar en el CRUD genérico.

```ts
@Injectable({ providedIn: 'root' })
export class ProductosService {
  private readonly _api = inject(ApiService);

  getAll(categoria_id?: string): Observable<ApiResponse<Producto[]>> {
    const query = categoria_id ? `?categoria_id=${categoria_id}` : '';
    return this._api.get<Producto[]>(`/productos${query}`);
  }
  getById(id: string): Observable<ApiResponse<Producto>> {
    return this._api.get<Producto>(`/productos/${id}`);
  }
  create(dto: ProductoInsDto): Observable<ApiResponse<Producto>> {
    return this._api.post<Producto>('/productos', dto);
  }
  update(dto: ProductoUpdDto): Observable<ApiResponse<Producto>> {
    return this._api.put<Producto>(`/productos/${dto.producto_id}`, dto);
  }
  delete(id: string): Observable<ApiResponse<void>> {
    return this._api.delete<void>(`/productos/${id}`);
  }
}
```

`GetAll` con filtros: parámetros opcionales sueltos concatenados a la
query string (como arriba) — no un objeto `SelDto` dedicado. Ese patrón
existe en el backend (ver `dotnet-clean-architecture`) pero en el frontend
casi no se usa en la práctica; no lo fuerces salvo que el usuario lo pida.

## Modelos/DTOs — convención de nombres

```ts
export interface Producto {
  producto_id: string;
  nombre: string;
  precio: number;
  categoria_id: string | null;
  estado: boolean;
}

export interface ProductoInsDto {
  nombre: string;
  precio: number;
  categoria_id: string | null;
}

export interface ProductoUpdDto extends ProductoInsDto {
  producto_id: string;
}
```

- `Entidad` — espejo de la fila de BD, todo lo nullable tipado
  `T | null`.
- `EntidadInsDto` — solo los campos editables al crear.
- `EntidadUpdDto extends EntidadInsDto` — agrega el PK.
- DTOs de acciones puntuales (no CRUD) son válidos y esperables (ej.
  `EntidadGenerarDto`) — no hace falta forzarlos al molde Ins/Upd.
- **Todo ID es `string`**, nunca `number` — el frontend nunca sabe si un ID
  está encriptado o no, lo trata siempre como opaco (ver
  `auth-and-security.md`).
- Evita anidar arrays de entidades relacionadas directo en el modelo
  principal (`producto.detalles: Detalle[]`) salvo un caso real que lo
  justifique (ej. un editor visual que necesita el árbol completo en una
  sola respuesta) — no es el patrón por defecto.

## Catálogos con cache

Listas de referencia (categorías, unidades de medida, estados) se sirven
desde un único servicio con cache en memoria, para no repetir la misma
llamada HTTP en cada componente que necesita el mismo catálogo:

```ts
@Injectable({ providedIn: 'root' })
export class CatalogosService {
  private readonly _api = inject(ApiService);
  private readonly _cache: Record<string, Observable<CatalogoItem[]>> = {};

  getCategorias(): Observable<CatalogoItem[]> { return this._getCached('categorias', '/catalogos/categorias'); }
  getUnidadesMedida(): Observable<CatalogoItem[]> { return this._getCached('unidades-medida', '/catalogos/unidades-medida'); }

  private _getCached(key: string, path: string): Observable<CatalogoItem[]> {
    if (!this._cache[key]) {
      this._cache[key] = this._api.get<CatalogoItem[]>(path).pipe(
        map(res => res.data ?? []),
        tap(() => {}),
        shareReplay(1),
      );
    }
    return this._cache[key];
  }
}
```

## Manejo y visualización de errores

No hay interceptor global de error/toast (ver `routing-and-lazy-loading.md`).
Cada componente maneja error de transporte (HTTP) Y error de negocio
(`success: false` con HTTP 200) por separado, siempre:

```ts
this._api.getAll().subscribe({
  next: res => {
    if (res.success) { this.items.set(res.data ?? []); }
    else { this._notif.error(res.message); }
  },
  error: () => { this._notif.error('No se pudo conectar con el servidor.'); },
});
```

Dos canales de error distintos, ambos obligatorios de manejar en cada
llamada: el `catchError`/`error` (transporte) y el `if (!res.success)`
(regla de negocio) — un `ApiResponse` con HTTP 200 puede igual traer
`success: false`.

`.subscribe()` sin `takeUntilDestroyed()` es aceptable para llamadas HTTP
one-shot (completan solas, no hay leak). Reserva `takeUntilDestroyed()`
para suscripciones de larga duración (ej. un stream, un polling) — no lo
agregues por defecto a cada llamada CRUD simple.
