# Component Patterns

## Reglas base de todo componente

- **Standalone** siempre — nunca `NgModule`.
- **`ChangeDetectionStrategy.OnPush`** siempre.
- **`inject()`** para toda dependencia — nunca constructor DI.
- **Estado local con Signals** (`signal()`/`computed()`) — nunca
  `BehaviorSubject` para estado de UI. RxJS queda para el flujo async/HTTP.
- Control flow con `@if`/`@for`/`@else` — nunca `*ngIf`/`*ngFor`.
- No existe una base class compartida (tipo `BaseOverviewComponent`) — cada
  componente `-overview` implementa sus propios signals de `loading`,
  `items`, `searchTerm`, etc. Es boilerplate repetido a propósito: no
  extraigas una base class salvo que el usuario la pida explícitamente: no
  es el patrón real hoy.

## Las 4 variantes del split overview/list/fields

Elige la variante según la complejidad/anidamiento de la entidad — las 4
son válidas, no hay una única "correcta".

### Variante 1 — 3-tier estricto (entidad simple, lista con muchas filas)

- **`{entidad}-overview`**: página routeada, dueña del servicio y del
  diálogo (`p-dialog`), orquesta cargar/crear/editar/eliminar.
- **`{entidad}-list`**: presentacional puro — solo `input()`/`output()`,
  sin servicio propio, sin diálogo propio.
- **`{entidad}-fields`**: solo los campos del formulario — recibe el
  `FormGroup` por `input()`, sin servicio, sin diálogo.

```ts
@Component({ selector: 'app-producto-overview', standalone: true, changeDetection: ChangeDetectionStrategy.OnPush, ... })
export class ProductoOverviewComponent implements OnInit {
  private readonly _api = inject(ProductosService);
  private readonly _notif = inject(NotificationService);

  items = signal<Producto[]>([]);
  loading = signal(false);
  dialogVisible = signal(false);

  ngOnInit(): void { this.loadData(); }

  loadData(): void {
    this.loading.set(true);
    this._api.getAll().subscribe({
      next: res => { if (res.success) this.items.set(res.data ?? []); this.loading.set(false); },
      error: () => { this._notif.error('No se pudo cargar la lista.'); this.loading.set(false); },
    });
  }
}
```

### Variante 2 — 2-tier overview + form con diálogo (entidad simple, formulario corto)

- **`{entidad}-overview`**: dueña del servicio, lista + búsqueda + abre el
  diálogo que envuelve `{entidad}-form`.
- **`{entidad}-form`**: dueño del `FormGroup` Y del `<p-dialog>` (lo
  envuelve él mismo en su propio template) — no inyecta servicio, se
  comunica con el padre por `output()`/`model()`.

```ts
@Component({ selector: 'app-cliente-form', standalone: true, changeDetection: ChangeDetectionStrategy.OnPush, ... })
export class ClienteFormComponent {
  private readonly _fb = inject(FormBuilder);

  visible = model.required<boolean>();
  cliente = input<Cliente | null>(null);
  saved = output<ClienteInsDto>();
  cancelled = output<void>();

  form = this._fb.group({
    nombre: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
  });

  submit(): void {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    this.saved.emit(this.form.getRawValue());
  }
}
```

### Variante 3 — página enrutada (entidad grande, con muchas sub-secciones)

Para la entidad más compleja del dominio (muchas pestañas/paneles de
detalle): `{entidad}-overview` no abre diálogo, navega a rutas propias.

```ts
// dentro de {entidad}-overview
irACrear(): void { this._router.navigate(['/ventas/pedidos/nuevo']); }
irAEditar(id: string): void { this._router.navigate(['/ventas/pedidos', id]); }
```

`{entidad}-form` es entonces una página completa (`pages/{entidad}-form/`),
no un componente de diálogo, y vive en su propia ruta (`nuevo` / `:id` —
ver `routing-and-lazy-loading.md`).

### Variante 4 — excepción "subdominio anidado"

Dentro de una entidad grande (variante 3), cada sub-sección propia
(pestaña/panel independiente) es una mini-feature autónoma anidada dentro
de `components/`. Ahí, el `-list` de la sub-sección **sí** posee servicio +
diálogo + `FormGroup` — rompe la regla de "list presentacional puro" de la
variante 1, y es intencional: la sub-sección se comporta como su propia
`-overview` en miniatura, solo que vive dentro de `components/` en vez de
`pages/` porque no tiene ruta propia.

```
components/{seccion}/
├── models/{seccion}.model.ts
├── services/{seccion}.service.ts
├── {seccion}-list/       ← dueño de servicio + diálogo + FormGroup (excepción)
└── {seccion}-fields/     ← el único realmente presentacional aquí
```

No trates esto como un bug a corregir — es el patrón correcto para
sub-secciones anidadas sin ruta propia. Sí evita generalizarlo: fuera de
este caso (sub-sección sin ruta, anidada en una entidad grande), usa la
variante 1 o 2.

## Cuál elegir

| Situación | Variante |
|---|---|
| Entidad simple, formulario corto (pocos campos) | 2 — overview + form con diálogo |
| Entidad simple, lista con muchas columnas/filas o lógica de tabla compleja | 1 — 3-tier estricto |
| Entidad grande con muchas sub-secciones de detalle | 3 — página enrutada, + 4 para cada sub-sección |

Ante duda entre variante 1 y 2 para una entidad simple, pregunta al usuario
o replica la variante ya usada por entidades hermanas del mismo `{core}`.
