# Component Patterns

## Reglas base de todo componente

- **Standalone** siempre — nunca `NgModule`.
- **`ChangeDetectionStrategy.OnPush`** siempre.
- **`inject()`** para toda dependencia — nunca constructor DI.
- **Estado local con Signals** (`signal()`/`computed()`) — nunca
  `BehaviorSubject` para estado de UI. RxJS queda para el flujo async/HTTP.
- Control flow con `@if`/`@for`/`@else` — nunca `*ngIf`/`*ngFor`.
- No existe una base class compartida (tipo `BaseOverviewComponent`) —
  composición, no herencia. Cada `-overview` conserva su estado de negocio
  (`items`, `loading`, modal abierto, fila por eliminar) y sus llamadas al
  servicio. Lo que **no** se repite es el listado en sí (tarjeta, buscador,
  filtro local, estados vacío/carga/sin resultados, tabla y menú de
  acciones por fila): va en el componente compartido
  `shared/ui/tabla-listado` — ver "Listado reutilizable" más abajo.
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

## Listado reutilizable — `shared/ui/tabla-listado`

Regla dura: **todo listado de un `-overview` usa este componente**. La
página nunca vuelve a escribir la tarjeta, el buscador, los estados ni el
`p-table`. Se crea en `shared/` en cuanto exista el primer listado.

Excepción: si el proyecto ya tiene un grid de una librería corporativa
(ej. `ejep-grid` de `@ejep/component`), se usa ese y no se crea este.

| Lo pone el componente | Lo pone cada página |
|---|---|
| Tarjeta con título, conteo "x de y" y buscador | Columnas (`ColumnaTabla<T>[]`) |
| Filtro local (`buscarEn`) | Plantillas de las celdas ricas |
| Estados: carga (skeleton), vacío (con botón crear), sin resultados | Textos: buscador, vacío, sin resultados |
| `p-table`, cabeceras, celdas simples | `acciones: (fila) => MenuItem[]` |
| Paginación local (de 10 en 10, selector 10/15/20, "1–10 de N") | Solo si necesita otra cosa: `[paginar]="false"`, `filasPorPagina`, `opcionesFilasPorPagina` |
| Botón ⋮ y `p-menu` popup por fila | Qué hace cada acción (abrir modal, eliminar…) |

### Modelo híbrido (por qué no todo en TS)

Las **celdas simples** se declaran en TS; las **celdas ricas** (badges,
árbol con sangría, siglas, celdas de dos líneas) se escriben como
`<ng-template>` en el HTML de la página. Un modelo 100 % TS (estilo
`eModel` con `btn`/`badge`/`classFn`/`textFn`) obliga a ampliar la
interfaz con cada tipo de celda nuevo y saca el marcado del template:
se pierde el chequeo de tipos de la plantilla y la reutilización literal
de las clases del design system.

```ts
// shared/models/tabla.model.ts
export interface ColumnaTabla<T> {
  id: string;                                                     // columna y plantilla que la pinta
  titulo: string;
  valor?: (fila: T) => string | number | null | undefined;        // celda simple
  mono?: boolean;                                                 // clase .mono
  derecha?: boolean;                                              // text-right en cabecera y celda
  ocultarEnMd?: boolean;                                          // col-hide-md en cabecera y celda
}
export interface EstadoVacioTabla { icono: string; titulo: string; texto: string; accion: string; }
export interface CeldaContexto<T> { $implicit: T; busqueda: string; }
```

```ts
// shared/ui/tabla-listado/celda.directive.ts — marca la plantilla de una columna
@Directive({ selector: 'ng-template[{prefijo}Celda]' })
export class CeldaDirective {
  readonly columna = input.required<string>({ alias: '{prefijo}Celda' });
  readonly plantilla = inject<TemplateRef<CeldaContexto<unknown>>>(TemplateRef);
}
```

Componente (`{prefijo}-tabla-listado`, genérico `<T>`, OnPush): inputs
`filas`, `columnas`, `cargando`, `titulo` (default "Listado"),
`etiquetaBuscar`, `placeholder`, `buscarEn: (fila) => (string|number|null)[]`,
`etiquetaCarga`, `filasCarga` (default 4), `vacio: EstadoVacioTabla`,
`sinResultados` (ej. "Ninguna sede" → `Ninguna sede coincide con "x".`),
`claveFila` (dataKey), `anchoMinimo`, `etiquetaColAcciones`,
`etiquetaFila: (fila) => string` (aria-label del ⋮), `acciones: (fila) =>
MenuItem[]`, `paginar` (default `true`), `filasPorPagina` (default `10`),
`opcionesFilasPorPagina` (default `[10, 15, 20]`); output `crear` (botón
del estado vacío). Internamente:
`contentChildren(CeldaDirective)` → mapa id→plantilla, `busqueda` y la
fila seleccionada para el menú son signals propios.

Detalles que garantizan que el marcado sea idéntico al de un listado
escrito a mano:

- Celda simple con `[textContent]` (`<td [class]="…" [textContent]="…">`):
  no arrastra los espacios que deja el formateo del template.
- Orden de clases fijo: celda `mono` → `text-right` → `col-hide-md`;
  cabecera `text-right` → `col-hide-md`.
- Host con `display: contents`: el componente no aporta caja propia.
- El `p-menu` popup (`appendTo="body"`) vive dentro del componente.

Paginación (local, sobre las filas ya filtradas):

- **Siempre activa por defecto**, de 10 en 10, con selector 10/15/20 y el
  reporte `{first}–{last} de {totalRecords}`. Los defaults viven solo en el
  componente: una página **no** los repite; pasa el input únicamente cuando
  necesita otro valor (`[paginar]="false"` para listas cortas fijas,
  `[opcionesFilasPorPagina]="[25, 50]"`…).
- `porPagina` y `primera` son `linkedSignal`: el selector cambia las filas
  por página sin tocar el input, y la página actual se conserva mientras
  siga dentro de la lista (si un borrado o filtro la deja fuera, vuelve a
  la primera).
- Una búsqueda nueva siempre vuelve a la primera página.
- `paginatorDropdownAppendTo="body"` para que el selector no quede
  recortado dentro de la tarjeta.

Uso en una página:

```html
<{prefijo}-tabla-listado
  [filas]="items()" [columnas]="columnas" [cargando]="loading()"
  etiquetaBuscar="Buscar sede" placeholder="Buscar por nombre, código o número"
  [buscarEn]="buscarEn" etiquetaCarga="Cargando sedes" [vacio]="vacio"
  sinResultados="Ninguna sede" claveFila="sede_id" anchoMinimo="44rem"
  [etiquetaFila]="etiquetaFila" [acciones]="acciones" (crear)="abrirCrear()">
  <!-- Solo las celdas ricas llevan plantilla; let-busqueda si la celda usa el término -->
  <ng-template {prefijo}Celda="estado" let-sede>
    @if (sede.habilitado) { <span class="badge badge-ok">Habilitada</span> }
    @else { <span class="badge badge-pendiente">Cierre temporal</span> }
  </ng-template>
</{prefijo}-tabla-listado>
```

```ts
protected readonly columnas: ColumnaTabla<Sede>[] = [
  { id: 'codigo', titulo: 'Código', valor: s => s.codigo, mono: true },
  { id: 'telefono', titulo: 'Teléfono', valor: s => s.telefono || '—', mono: true, ocultarEnMd: true },
  { id: 'estado', titulo: 'Estado' },                       // rica: plantilla en el HTML
];
protected readonly vacio: EstadoVacioTabla = { icono: 'bx-store-alt', titulo: 'Aún no hay sedes', texto: '…', accion: 'Nueva sede' };
protected readonly buscarEn = (s: Sede) => [s.nombre, s.codigo, s.numero];
protected readonly etiquetaFila = (s: Sede) => s.nombre;
protected readonly acciones = (sede: Sede): MenuItem[] => [
  { label: 'Editar', icon: 'bx bx-edit-alt', command: () => this.abrirEditar(sede) },
  { separator: true },
  { label: 'Eliminar', icon: 'bx bx-trash', styleClass: 'menu-item--peligro', disabled: sede.es_hub, command: () => this.porEliminar.set(sede) },
];
```

- Las filas pueden ser cualquier forma, no solo la entidad (ej. nodos de
  un árbol `{ opcion, nivel, camino }` ya aplanados para un menú).
- Una fecha u otro formato en una celda simple se resuelve en `valor`
  con `formatDate`/`formatNumber` (sin pipes compartidos).
- ¿Una necesidad nueva (paginación de **servidor**, selección múltiple, fila
  expandible)? Se **amplía el componente** con un input/output nuevo —
  nunca se vuelve a un `p-table` suelto en la página.
- Al migrar listados existentes, verificar "sin cambio visual"
  comparando el DOM normalizado antes y después (sin comentarios ni
  atributos internos de Angular/PrimeNG, clases ordenadas).

Origen: ComercialMs (2026-11-06), 5 listados migrados con DOM idéntico.

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
