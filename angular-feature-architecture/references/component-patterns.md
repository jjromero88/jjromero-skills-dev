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
- **Reutilización antes que nada**: antes de crear un componente nuevo,
  revisa si ya existe uno reutilizable (en el propio feature o en
  `shared/ui`) — ver regla dura en `SKILL.md`.

## Antes de construir el CRUD: modal o vista independiente

Pregunta siempre al usuario (ver `SKILL.md`, "Antes de un CRUD nuevo") —
nunca lo asumas. Puedes sugerir un default:

- **Modal** — sugerido cuando la entidad tiene pocos campos o poca
  complejidad visual.
- **Vista independiente** (página propia para crear/editar) — sugerida
  cuando el volumen de datos/campos es alto (ej. un registro con muchas
  secciones/pestañas).

Ambas variantes comparten una regla dura: **crear y editar son siempre el
mismo componente** (mismo modal, o misma página) — nunca un componente
separado por operación.

## Caso típico — CRUD con modal

Ejemplo de referencia: un catálogo simple como "categorías de producto".

```
features/{core}/{entidad}/
├── {entidad}.routes.ts
├── services/{entidad}.service.ts       → consume ApiService (core), ver service-and-http-patterns.md
├── models/{entidad}.model.ts
├── pages/{entidad}-overview/
│   ├── {entidad}-overview.component.ts   → dueño del servicio, orquesta cargar/abrir modal/eliminar
│   ├── {entidad}-overview.component.html
│   └── {entidad}-overview.component.css  → SOLO estilo particular de esta vista
└── components/{entidad}-form/            → el modal de crear/editar (mismo componente para ambas operaciones)
    ├── {entidad}-form.component.ts
    ├── {entidad}-form.component.html
    └── {entidad}-form.component.css      → si lo necesita
```

Regla dura de límites: el html/css/ts del modal **nunca** vive en la
carpeta de `{entidad}-overview` — siempre en su propio componente, junto a
su propio service/models si el modal los necesita (ej. catálogos que solo
usa el formulario). La vista `-overview` (listado) solo tiene su propio
html/css/ts — nada del modal se filtra ahí.

```ts
@Component({ selector: 'app-categoria-overview', standalone: true, changeDetection: ChangeDetectionStrategy.OnPush, ... })
export class CategoriaOverviewComponent implements OnInit {
  private readonly _api = inject(CategoriaService);
  private readonly _notif = inject(NotificationService);

  items = signal<Categoria[]>([]);
  loading = signal(false);
  dialogVisible = signal(false);
  editing = signal<Categoria | null>(null);   // null = crear, con valor = editar — mismo modal

  ngOnInit(): void { this.loadData(); }

  loadData(): void {
    this.loading.set(true);
    this._api.getAll().subscribe({
      next: res => { if (res.success) this.items.set(res.data ?? []); this.loading.set(false); },
      error: () => { this._notif.error('No se pudo cargar la lista.'); this.loading.set(false); },
    });
  }

  abrirCrear(): void { this.editing.set(null); this.dialogVisible.set(true); }
  abrirEditar(item: CentroCosto): void { this.editing.set(item); this.dialogVisible.set(true); }
}
```

Si `pages/` tiene más vistas además del overview y estas reutilizan un
componente, y esa reutilización es exclusiva de esta entidad (no de toda la
app), el componente compartido vive dentro de `components/` de esta misma
entidad — nunca duplicado entre las vistas que lo usan.

## Caso complejo — vista independiente seccionada

Ejemplo de referencia: un registro extenso con muchas secciones (ej. un
expediente con datos personales, contacto, historial, documentos, etc.).

```
features/{core}/{entidad}/
├── models/{entidad}.model.ts
├── services/{entidad}.service.ts
├── pages/
│   ├── {entidad}-overview/            → lista
│   └── {entidad}-form/                → página de crear/editar, siempre la misma para ambas operaciones
│       ├── {entidad}-form.component.ts   → orquesta con un signal `activeSection` + @switch
│       └── {entidad}-form.component.html
└── components/                        → un componente por sección, plano al mismo nivel por defecto
    ├── {seccion-a}/
    ├── {seccion-b}/
    └── {seccion-c}/
```

`{entidad}-form` no abre diálogo — es una página propia (rutas `nuevo` /
`:id`, ver `routing-and-lazy-loading.md`). Orquesta las secciones con un
signal y `@switch`:

```ts
activeSection = signal<'seccion-a' | 'seccion-b' | 'seccion-c'>('seccion-a');
```
```html
@switch (activeSection()) {
  @case ('seccion-a') { <div @fadeIn><app-seccion-a [form]="form" /></div> }
  @case ('seccion-b') { <div @fadeIn><app-seccion-b [form]="form" /></div> }
  @case ('seccion-c') { <div @fadeIn><app-seccion-c [form]="form" /></div> }
}
```

**Agrupar los componentes de sección en subcarpetas es atípico** — el
default es dejarlos todos planos al mismo nivel dentro de `components/`.
Si el usuario pide agruparlos (por ejemplo, separar secciones de solo
lectura de secciones con sus propios sub-registros), pregúntalo siempre
antes de hacerlo — no lo decidas por tu cuenta.

Cada componente de sección tiene su propia estructura repetitiva completa:
su `.html`, su `.ts`, su `.css` si lo necesita, sus `models/`, y sus
`services/`.

### Sub-CRUD anidado dentro de una sección

Si una sección es en sí misma un CRUD (ej. un catálogo anidado dentro del
expediente), sigue el patrón típico internamente — `-fields` (formulario) y
`-list` (tabla) como componentes hermanos — pero sus `models/`/`services/`
viven a la altura de la carpeta que agrupa a ambos hermanos, no anidados un
nivel más adentro:

```
components/{seccion}/
├── models/{seccion}.model.ts       ← a la altura del grupo, no dentro de -list ni de -fields
├── services/{seccion}.service.ts
├── {seccion}-list/
│   ├── {seccion}-list.component.ts    → posee servicio + diálogo + FormGroup en este caso anidado
│   └── {seccion}-list.component.html
└── {seccion}-fields/                  → solo los campos del formulario, sin servicio propio
    ├── {seccion}-fields.component.ts
    └── {seccion}-fields.component.html
```

Aquí el `-list` sí posee servicio + diálogo + `FormGroup` (a diferencia del
`-list` presentacional puro de un split de 3 niveles clásico) porque esta
sub-sección funciona como su propia mini-`-overview`, solo que anidada
dentro de `components/` en vez de tener ruta propia. No generalices este
caso fuera de una sub-sección anidada sin ruta.

## Estilo visual de la nav secundaria (caso complejo)

Ver skill `angular-design-system`, `references/layout-navigation-and-toolbar.md`
para el detalle visual exacto (nav sticky de 220px, grupos con encabezado en
mayúsculas, transición entre secciones, superficie de errores de validación
por sección, barra de acciones sticky al pie).
