# Auth and Security

## Login en 2 fases (multi-tenant)

1. **`login(usuario, password)`** — credenciales puras. Si son válidas, el
   backend responde con la lista de empresas/entidades ("sociedades") a las
   que ese usuario tiene acceso — todavía sin JWT.
2. **`authorize({ usuario_id, sociedad_id })`** — el cliente elige una de
   esas entidades; el backend responde recién ahí con el JWT + el árbol de
   menú/permisos para ese contexto específico.

`switchSociedad()` vuelve a llamar `authorize()` con otra `sociedad_id`
para cambiar de contexto sin pedir credenciales de nuevo.

```ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly _api = inject(ApiService);

  private readonly _user = signal<AuthUser | null>(this._loadFromStorage());
  isAuthenticated = computed(() => this._user() !== null);
  token = computed(() => this._user()?.token ?? null);

  login(usuario: string, password: string): Observable<ApiResponse<Sociedad[]>> {
    return this._api.post<Sociedad[]>('/auth/login', { usuario, password });
  }

  authorize(usuario_id: string, sociedad_id: string): Observable<ApiResponse<AuthUser>> {
    return this._api.post<AuthUser>('/auth/authorize', { usuario_id, sociedad_id }).pipe(
      tap(res => { if (res.success && res.data) this._persist(res.data); }),
    );
  }

  isTokenExpired(): boolean {
    const token = this.token();
    if (!token) return true;
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.exp * 1000 < Date.now();
  }

  logout(): void {
    localStorage.removeItem('auth-user');
    this._user.set(null);
  }

  private _persist(user: AuthUser): void {
    localStorage.setItem('auth-user', JSON.stringify(user));
    this._user.set(user);
  }

  private _loadFromStorage(): AuthUser | null {
    const raw = localStorage.getItem('auth-user');
    return raw ? JSON.parse(raw) : null;
  }
}
```

Expiración de token chequeada 100% client-side, decodificando el payload
del JWT a mano (`atob` + `JSON.parse`) — no se usa ninguna librería de JWT
para esto, es intencionalmente simple.

## Permisos — visibilidad de menú, no gating de UI por botón

El backend devuelve el árbol de menú ya filtrado según lo que el usuario
puede ver — la UI solo renderiza las opciones de menú que llegaron, no
oculta botones/acciones individuales según un código de permiso. Si el
usuario pide gating fino por botón/acción (ej. "solo mostrar el botón
Eliminar si tiene el permiso X"), es una feature nueva a construir
explícitamente, no un patrón ya existente para copiar.

## IDs — opacos, sin encriptar en frontend

Todo ID es `string` en todo modelo TypeScript (ver
`service-and-http-patterns.md`). El frontend **nunca** encripta, desencripta,
ni interpreta un ID — ese es un límite exclusivo del backend (ver skill
`dotnet-clean-architecture`, `references/security-patterns.md`). No agregues
lógica de crypto en el cliente por ningún motivo.

## Guards

Ver `routing-and-lazy-loading.md` — `authGuard`/`guestGuard`, siempre
funcionales, nunca basados en clase.
