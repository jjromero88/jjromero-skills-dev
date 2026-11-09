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

## Atomicidad obligatoria — un modal = un componente; un detalle = `-list` + `-form`; una sección = un componente

**Regla dura, sin excepción. Violarla es un error FATAL** (la feature no se
entrega): la arquitectura de esta skill se caracteriza por su atomicidad, y
un componente que acumula la tabla, el modal y la lógica de otra
responsabilidad la rompe.

1. **Un modal es siempre su propio componente**, con su estructura
   habitual (`{x}-form.component.ts` + `.html` [+ `.spec.ts`]) y envuelto en
   `<{prefijo}-modal>`. El `<{prefijo}-modal>` **solo** aparece en el html
   de ese componente; quien lo abre pone únicamente la etiqueta del
   componente del modal.
2. **Un detalle / sub-CRUD dentro de un formulario** (direcciones,
   contactos, cuentas bancarias, documentos, cuotas…) se parte siempre en
   `components/{seccion}/` con:
   - `{seccion}-list/` → título de la sección (con su conteo y botón
     "Agregar"), tabla, estado vacío, quitar con confirmación y apertura del
     modal. Es el único que conoce al modal.
   - `{seccion}-form/` → el modal de registrar/editar (mismo componente para
     ambas operaciones), con su propio `FormGroup`, validaciones y mensajes.
   - `models/` → los tipos y el armado del `FormGroup` **propios de la
     sección** (ej. el tipo de la fila y su `crear{Fila}()`), a la altura de
     la carpeta que agrupa a `-list` y `-form`, nunca en el modelo de la
     entidad padre ni en la página.
   - `services/` → solo si la sección tiene API propia (ej. la sub-entidad se
     guarda sola contra su endpoint); también a la altura del grupo.
   El contrato de la API de la entidad padre (ej. `ClienteDireccion` dentro de
   `ClienteInsDto`) se queda en el modelo del padre: es su DTO, no de la
   sección.
3. **Un formulario seccionado se compone de secciones.** Si un formulario
   (página de vista independiente o modal) agrupa sus campos en secciones
   con título (Identificación, Contacto, Datos comerciales, Crédito,
   Tributario, Observaciones…), **cada sección es su componente**
   `components/{entidad}-{seccion}/` y el formulario solo las renderiza en
   orden. El html del formulario no contiene campos ni títulos de sección:
   solo la cabecera, el `<form>` con las etiquetas de las secciones y el pie.
   Cada sección:
   - recibe por `input` el `FormGroup` del formulario (`[form]="form"`) y las
     listas que usan sus campos; el `FormGroup`, sus validadores y el
     guardado siguen siendo del formulario (un solo `form`, un solo request);
   - pinta su propio título y sus campos dentro de
     `<div class="contents" [formGroup]="form()">`, con
     `host: { style: 'display: contents' }`, para que sus campos se sumen a
     la grilla del formulario como si estuvieran escritos en él;
   - se lleva la lógica de pantalla de sus campos (habilitar/deshabilitar
     según otro campo, cálculos de solo lectura, contadores) y la aplica
     también al iniciar (al editar llega con los datos ya cargados).
   Un detalle con tabla (punto 2) es una sección más: su `-list` es el
   componente de la sección.
4. **Un html no se desborda**: más de **300 líneas** en un `.html` obliga a
   partirlo en componentes antes de entregar.

### NO — así nunca (ComercialMs, `cliente-form` antes del 2026-11-08)

La página de crear/editar cliente acumuló las tablas de direcciones y
contactos **y los dos modales**, con su `FormGroup`, sus validaciones, el
radio/switch de principal, la confirmación de quitar y el armado de filas.
El html llegó a ~720 líneas y el ts a ~650:

```
features/com/cliente/
├── pages/cliente-form/
│   ├── cliente-form.component.html    ✗ ~720 líneas: datos del cliente + 2 tablas + 2 <comercialms-modal>
│   └── cliente-form.component.ts      ✗ capturaDireccion, capturaContacto, editandoDireccion,
│                                         guardarDireccion(), quitarContacto(), ubicacion(), cargoDe()…
└── (sin components/)                  ✗ ni direcciones/ ni contactos/
```

```html
<!-- ✗ pages/cliente-form/cliente-form.component.html -->
</form>
@if (capturaDireccionAbierta()) {
  <comercialms-modal [titulo]="..." (cerrar)="cancelarCapturaDireccion()">   <!-- ✗ modal dentro de la página -->
    <form id="direccion-form" [formGroup]="capturaDireccion"> … </form>
  </comercialms-modal>
}
```

```html
<!-- ✗ pages/cliente-form/cliente-form.component.html: cada sección escrita dentro del formulario -->
<form id="cliente-form" class="form-pagina form-grid" [formGroup]="form" (ngSubmit)="guardar()">
  <h3 class="form-seccion-titulo col-span-full">Identificación</h3>   <!-- ✗ título de sección en el form -->
  <div class="campo"> <label for="tipo_documento_id">Tipo de documento</label> <p-select … (onChange)="cambiarTipo()" /> </div>
  <div class="campo"> … numero_documento … </div>
  …
  <h3 class="form-seccion-titulo col-span-full">Contacto</h3>         <!-- ✗ -->
  …
  <h3 class="form-seccion-titulo col-span-full">Datos comerciales</h3> <!-- ✗ -->
  …
  <h3 class="form-seccion-titulo col-span-full">Tributario</h3>       <!-- ✗ -->
  …
</form>
```

### SÍ — así siempre

Referencia de estructura: `nomina_web_v1`, `features/personal/empleados/`
(cada sub-CRUD del empleado — contratos, derechohabientes, retenciones… — es
`components/{seccion}/{seccion}-list` + `{seccion}-fields`). Corregido aquí:
la sección también se lleva su `models/` (y `services/` si tiene API
propia), y el modal usa el armazón compartido `<{prefijo}-modal>` y la tabla
el lenguaje compartido del proyecto.

```
features/com/cliente/
├── pages/cliente-form/
│   ├── cliente-form.component.html          ✓ cabecera + <form> con las etiquetas de las secciones + pie
│   └── cliente-form.component.ts            ✓ dueño del FormGroup del cliente, sus validadores y el guardado
└── components/
    ├── cliente-identificacion/               ✓ una sección = un componente (título + campos + su lógica)
    ├── cliente-contacto/
    ├── cliente-datos-comerciales/
    ├── cliente-credito/                      ✓ con sus cálculos (disponible, sobregirado)
    ├── cliente-tributario/
    ├── cliente-observaciones/                ✓ con su contador de caracteres
    ├── direcciones/
    │   ├── models/cliente-direccion-fila.model.ts   ✓ tipo de la fila + crearFilaDireccion()
    │   ├── cliente-direccion-list/                   ✓ título, tabla, vacío, principal, quitar, abre el modal
    │   └── cliente-direccion-form/                   ✓ <comercialms-modal> + su form y validaciones
    └── contactos/
        ├── models/cliente-contacto-fila.model.ts
        ├── cliente-contacto-list/
        └── cliente-contacto-form/
```

```html
<!-- ✓ pages/cliente-form/cliente-form.component.html: el formulario solo compone sus secciones -->
<form id="cliente-form" class="form-pagina form-grid" [formGroup]="form" (ngSubmit)="guardar()" novalidate>
  <comercialms-cliente-identificacion [form]="form" [tiposDocumento]="tiposDocumento" [esEdicion]="esEdicion" />
  <comercialms-cliente-contacto [form]="form" />
  <comercialms-cliente-datos-comerciales [form]="form" [condicionesVenta]="condicionesVenta" … />
  <comercialms-cliente-credito [form]="form" [deuda]="deuda" />
  <comercialms-cliente-tributario [form]="form" />
  <comercialms-cliente-direccion-list class="col-span-full" [filas]="form.controls.direcciones" … />
  <comercialms-cliente-contacto-list class="col-span-full" [filas]="form.controls.contactos" … />
  <comercialms-cliente-observaciones [form]="form" />
</form>

<!-- ✓ components/cliente-tributario/cliente-tributario.component.html: título y campos de la sección -->
<div class="contents" [formGroup]="form()">
  <h3 class="form-seccion-titulo col-span-full">Tributario</h3>
  <div class="campo-switch"> <p-toggleswitch inputId="entidad_publica" formControlName="entidad_publica" /> … </div>
  …
</div>
```

```ts
// ✓ components/cliente-tributario/cliente-tributario.component.ts
@Component({
  selector: 'comercialms-cliente-tributario',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule, InputTextModule, ToggleSwitchModule],
  templateUrl: './cliente-tributario.component.html',
  host: { style: 'display: contents' },   // sus campos se suman a la grilla del formulario
})
export class ClienteTributarioComponent {
  readonly form = input.required<FormGroup>();   // el FormGroup del formulario: validadores y guardado son de él
  protected readonly hasFormError = hasFormError;
  protected readonly getFormError = getFormError;
}
```

```html
<!-- ✓ components/direcciones/…: un detalle con tabla es una sección más -->
<comercialms-cliente-direccion-list
  class="col-span-full"
  [filas]="form.controls.direcciones"
  [originales]="cliente()?.direcciones ?? []"
/>

<!-- ✓ components/direcciones/cliente-direccion-list/cliente-direccion-list.component.html -->
<div class="detalle"> … tabla, estado vacío, botón Agregar … </div>
@if (modalAbierto()) {
  <comercialms-cliente-direccion-form
    [direccion]="editando()"
    [numero]="indiceEditando()"
    (guardar)="aplicar($event)"
    (cerrar)="modalAbierto.set(false)"
  />
}

<!-- ✓ components/direcciones/cliente-direccion-form/cliente-direccion-form.component.html -->
<comercialms-modal [titulo]="titulo()" [ancho]="true" (cerrar)="cancelar()">
  <form id="direccion-form" class="modal-cuerpo form-grid form-grid-tres" [formGroup]="form" (ngSubmit)="aceptar()">
    …
  </form>
  <ng-container ngProjectAs="[pie]"> … </ng-container>
</comercialms-modal>
```

**Detalle que se guarda con el padre** (un solo request, como las
direcciones del cliente): el `-list` recibe por `input` el `FormArray` del
formulario padre y lo modifica (agregar, reemplazar, quitar) —así el padre
queda "con cambios" y su guardado lo envía—; el `-form` recibe la fila a
editar (o `null` para una nueva) y emite el valor validado; no hay
`services/`. **Detalle con API propia** (cada fila se guarda sola, como los
contratos de un empleado ya creado): el `-list` posee el servicio y recarga
tras guardar; ver "Sub-CRUD anidado" más abajo.

### Verificación (obligatoria al terminar cada feature)

Desde `src/app/` del proyecto (con el prefijo real en vez de `{prefijo}`):

```bash
# 1a. Ningún <{prefijo}-modal> en una página, un overview ni un -list (ahí solo va la etiqueta del componente del modal)
grep -rln '<{prefijo}-modal' features --include='*.html' | grep -E '/pages/|-overview\.component|-list\.component'
# 1b. En components/, el <{prefijo}-modal> es la raíz de su componente (el componente ES el modal, no lo contiene)
for f in $(grep -rl '<{prefijo}-modal' features --include='*.html' | grep '/components/'); do
  primera=$(grep -v '^[[:space:]]*$' "$f" | grep -v '^[[:space:]]*<!--' | head -1)
  case "$primera" in *'<{prefijo}-modal'*) ;; *) echo "$f";; esac
done
# 2. Ningún html con más de un título de sección: cada sección en su componente (en este estilo el título es
#    .form-seccion-titulo; si el proyecto usa otra clase para el título de sección, esa)
grep -rl 'form-seccion-titulo' features --include='*.html' | while read f; do
  n=$(grep -c 'form-seccion-titulo' "$f"); [ "$n" -gt 1 ] && echo "$f ($n secciones)"
done
# 3. Ningún html de más de 300 líneas
find features -name '*.html' -exec wc -l {} + | awk '$2 != "total" && $1 > 300'
```

Si alguno de los dos imprime algo, la feature no está terminada: se parte
en componentes con la estructura del "SÍ" y se vuelve a correr.

Origen: ComercialMs (2026-11-08). La ficha de cliente se construyó con las
tablas y los modales de direcciones y contactos dentro de la página, y con
todas sus secciones (Identificación, Contacto, Datos comerciales,
Tributario…) escritas en el mismo html; el usuario lo marcó como error
fatal de atomicidad y se refactorizó a la estructura de arriba.

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
└── components/{entidad}-form/            → el modal de crear/editar (mismo componente para ambas operaciones), envuelto en `<{prefijo}-modal>`
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
  editandoId = signal<string | null>(null);   // null = crear, con ID = editar — mismo modal; el modal pide el getById

  ngOnInit(): void { this.loadData(); }

  loadData(): void {
    this.loading.set(true);
    this._api.getAll().subscribe({
      next: res => { if (res.success) this.items.set(res.data ?? []); this.loading.set(false); },
      error: () => { this._notif.error('No se pudo cargar la lista.'); this.loading.set(false); },
    });
  }

  abrirCrear(): void { this.editandoId.set(null); this.dialogVisible.set(true); }
  // Solo el ID: el modal trae el registro con getById y sus selects al abrir (forms-and-validation.md, "Carga al abrir")
  abrirEditar(item: Categoria): void { this.editandoId.set(item.categoria_id); this.dialogVisible.set(true); }
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
  ordenable?: boolean;                                            // cabecera clicable: ASC -> DESC -> sin orden
  campoOrden?: string;                                            // nombre que viaja al back (por defecto `id`)
  valorOrden?: (fila: T) => string | number | null | undefined;   // orden en memoria (por defecto `valor`)
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

### Orden por columna

Obligatorio en todo listado con columnas ordenables (pregunta "Antes de un
listado" del `SKILL.md`; "Ninguna" es válida). Lo resuelve `tabla-listado`;
la página solo marca las columnas y, en modo servidor, consulta.

| Pieza | Ruta | Qué hace |
|---|---|---|
| Modelo | `shared/models/orden.model.ts` | `SentidoOrden = 'ASC' \| 'DESC'`; `OrdenListado { campo; sentido }` |
| Helpers | `shared/helpers/orden.helpers.ts` | `siguienteOrden` (ciclo ASC → DESC → `[]`), `ordenarFilas` (en memoria), `agregarOrden` (query string) |
| Tabla | `shared/ui/tabla-listado` | `[(orden)]` / `[orden]` + `(cambioOrden)`; cabecera `<button class="tabla-orden">` con `aria-sort` e íconos `bx-sort-up` / `bx-sort-down` / `bx-sort-alt-2` |

Reglas del componente:

- **Sin orden al entrar** (`orden: []`): ningún ícono activo; la lista sale
  como llegó (el servidor la da por PK).
- **Modo local** (sin `totalRegistros`): ordena en memoria **después de
  filtrar**, con `Intl.Collator('es', { numeric: true, sensitivity: 'base' })`
  (sin distinguir mayúsculas ni tildes, "P2" antes que "P10"), números como
  números, **vacíos al final en ambos sentidos**, sort estable; vuelve a la
  primera página. No llama a la API.
- **Modo servidor** (con `totalRegistros`): no ordena; emite `cambioOrden`
  con la lista (vacía = sin orden). Mientras `cargando`, el clic se ignora
  sin cambiar el orden (el ícono sigue mostrando el de la consulta en curso).
- Implementación propia y no `pSortableColumn`: el orden local de PrimeNG
  compara por nombre de campo (las columnas usan funciones), no manda los
  vacíos al final ni compara en español.

```ts
// shared/helpers/orden.helpers.ts (extracto)
export function siguienteOrden(actual: readonly OrdenListado[], campo: string): OrdenListado[] {
  const vigente = actual[0];
  if (vigente?.campo !== campo) return [{ campo, sentido: 'ASC' }];
  return vigente.sentido === 'ASC' ? [{ campo, sentido: 'DESC' }] : [];
}
// GET con [FromQuery]: orden[0].campo=descripcion&orden[0].sentido=ASC; sin orden no agrega nada
export function agregarOrden(parametros: URLSearchParams, orden: readonly OrdenListado[]): void {
  orden.forEach((c, i) => { parametros.set(`orden[${i}].campo`, c.campo); parametros.set(`orden[${i}].sentido`, c.sentido); });
}
```

Página con lista completa (en memoria): solo marca las columnas.

```ts
protected readonly columnas: ColumnaTabla<Zona>[] = [
  { id: 'codigo', titulo: 'Código', valor: (z) => z.codigo, mono: true, ordenable: true },
  { id: 'nombre', titulo: 'Nombre', valor: (z) => z.nombre, ordenable: true },
];
```

Página paginada en el servidor: guarda el orden con la búsqueda, los filtros
y la página, y consulta con la página 1.

```html
<{prefijo}-tabla-listado [totalRegistros]="total()" [orden]="orden()" (cambioOrden)="ordenar($event)" … />
```

```ts
protected readonly orden = signal<OrdenListado[]>([]);          // sin orden al entrar
protected ordenar(orden: OrdenListado[]): void {
  this.orden.set(orden);
  this.primera.set(0);                                           // un orden nuevo vuelve a la pagina 1
  this.consultar();
}
// servicio: agregarOrden(parametros, filtro.orden) junto a busqueda, pagina y filas_por_pagina
```

El back (lista blanca, `OrdenListado`) y el SP (`@orden_campo` /
`@orden_sentido`, sin SQL dinámico, PK al final) siguen
`dotnet-clean-architecture` (`domain-and-dtos.md`, "Orden por columna de los
listados") y `sql-database-patterns` (`sp-templates.md`, "Sp_Sel paginado con
orden por columna"). Pruebas obligatorias en `testing.md`.

Origen: ComercialMs (2026-11-08), heredado del `ejep-grid` de EJE-PENAL
(`sort: true` por columna, `sortFields` con lista blanca en el back) y
mejorado: ícono siempre sincronizado con el orden vigente, sin orden por
defecto y orden en memoria para las listas completas.


### Filtros por columna

Obligatorio en todo listado con columnas filtrables (pregunta "Antes de un
listado" del `SKILL.md`: qué columnas, su tipo de dato y el tipo de filtro;
"Ninguna" es válida). Lo resuelve `tabla-listado` con el componente
compartido `filtro-columna`; la página solo declara `filtro` en sus
columnas y, en modo servidor, consulta. Mismo diseño que el orden.

| Pieza | Ruta | Qué hace |
|---|---|---|
| Modelo | `shared/models/filtro.model.ts` | `FiltroListado { campo; operador; valores: string[] }`, `OpcionFiltro { valor; etiqueta }`, unión discriminada `FiltroColumna` |
| Helpers | `shared/helpers/filtro.helpers.ts` | `OPERADORES_TEXTO`/`OPERADORES_NUMERO`, `cumpleFiltro`/`filtrarFilas` (en memoria), `opcionesDistintas`, `opcionesBooleanas`, `reemplazarFiltro`, `rangoComoFiltro`/`rangoDeFiltro`, `errorFiltro`, `agregarFiltros` (query string) |
| Menú | `shared/ui/filtro-columna/` | `filtro-columna` (embudo + `p-popover` + Aplicar/Limpiar) y un componente por control: `filtro-condicion`, `filtro-rango`, `filtro-numero` (2 decimales con `comercialmsDosDecimales`, prefijo/sufijo), `filtro-opciones` (casillas, lista, booleano), `filtro-fecha` (simple o rango, `p-datepicker` dd/mm/aaaa) |
| Tabla | `shared/ui/tabla-listado` | `[(filtros)]` / `[filtros]` + `(cambioFiltros)`; cabecera con `.tabla-cab` (título u orden + embudo) |
| Pruebas | `shared/testing/filtro-columna.ts` | Abrir, marcar, escribir, aplicar, limpiar y cerrar el menú como el usuario |

```ts
// shared/models/filtro.model.ts (extracto) — el tipo de filtro de una columna
export type FiltroColumna =
  | { tipo: 'condicion'; dato: 'texto' | 'numero' | 'monto' | 'porcentaje'; prefijo?: string }
  | { tipo: 'rango'; dato: 'numero' | 'monto' | 'porcentaje'; prefijo?: string }
  | { tipo: 'lista'; opciones?: OpcionFiltro[] }        // una opcion
  | { tipo: 'casillas'; opciones?: OpcionFiltro[] }     // varias, buscador, Seleccionar todo, contador
  | { tipo: 'booleano'; si: string; no: string }       // casillas con etiquetas (Activo / Inactivo)
  | { tipo: 'fecha' }                                  // es igual a
  | { tipo: 'rangoFechas' };
```

**Contrato** (igual en memoria y en el servidor): una entrada por columna
filtrada, combinadas con **Y**; los valores siempre como texto.

| Control | `operador` | `valores` |
|---|---|---|
| `casillas`, `lista`, `booleano` | `en` | IDs opacos, textos o `'true'`/`'false'` |
| `condicion` texto | `contiene`, `no_contiene`, `empieza`, `termina`, `igual`, `diferente` | `[texto]` |
| `condicion` número | `igual`, `diferente`, `mayor`, `mayor_igual`, `menor`, `menor_igual` | `['12.50']` |
| `rango`, `rangoFechas` | `entre` (los dos extremos), `mayor_igual` (solo desde), `menor_igual` (solo hasta) | números con punto o fechas ISO |
| `fecha` | `igual` | `['2026-11-08']` |

En un `GET` viaja como lo enlaza `[FromQuery]`:
`filtros[0].campo=marca&filtros[0].operador=en&filtros[0].valores[0]=x&filtros[0].valores[1]=y`
(`agregarFiltros(parametros, filtros)`).

Reglas del componente:

- **Embudo** `<button class="tabla-filtro">` con `aria-label="Filtrar
  {columna}"`, `aria-haspopup="dialog"`, `aria-expanded`; `bx-filter-alt`
  sin filtro y `bxs-filter-alt` (relleno, `.tabla-filtro--activo`) con
  filtro. Va dentro del `th` junto al botón de orden.
- **Menú `p-popover`** con el control y el pie **Limpiar** (quita el filtro
  de esa columna) / **Aplicar**. Lo marcado es un **borrador**: el control
  se crea en cada apertura con el filtro aplicado, así **cerrar (clic
  afuera o Esc) cancela** sin código extra. Aplicar sin nada (casillas sin
  marcar, condición sin valor) = sin filtro. Un rango con desde > hasta
  muestra `"Desde" no puede ser mayor que "Hasta".` y deshabilita Aplicar.
- **Foco**: al abrir, el campo marcado con `data-foco` (buscador, valor,
  desde) o el primer campo; al aplicar, limpiar o Esc vuelve al embudo.
  Se usa `[focusOnShow]="false"` porque `p-select` marca su propio
  `autofocus` y ganaría.
- **Overlays dentro del menú** (`p-select` del operador, `p-datepicker`)
  con `appendTo="self"`: abiertos en el `body`, un clic en ellos cuenta
  como "afuera" y cerraría el menú.
- **Casillas**: buscador "Buscar…" sin mayúsculas ni tildes, "Seleccionar
  todo" de tres estados (`p-checkbox [indeterminate]`) sobre lo visible,
  lista con scroll, contador "N seleccionados", "Sin opciones" / "Ninguna
  opción coincide". Booleano: sin buscador ni "Seleccionar todo".
- **Opciones**: las que trae la columna (`opciones`); un booleano, sus
  etiquetas; **en memoria** sin `opciones`, los valores distintos de la
  columna. **En servidor** la columna debe traer sus opciones (catálogo
  cargado al entrar): números y textos libres usan `condicion`/`rango`.
- **Modo local** (sin `totalRegistros`): filtra en memoria **después de la
  búsqueda y antes del orden**, Y entre columnas, texto con `sinTildes`,
  números como números, fechas por día; vuelve a la primera página. No
  llama a la API. El valor de cada fila: `valorFiltro` (o `valorOrden`,
  `valor`, la propiedad); un catálogo con `opciones` devuelve su ID.
- **Modo servidor** (con `totalRegistros`): no filtra; emite
  `cambioFiltros`. Mientras `cargando` no hay cabeceras (esqueleto).
- **Sin coincidencias con filtros de columna**, la tabla se queda con sus
  cabeceras y pinta "Sin resultados" en el cuerpo (`#emptymessage`): sin
  cabeceras no se podría cambiar el filtro.
- Implementación propia y no `p-columnFilter`: el de PrimeNG filtra por
  nombre de campo (las columnas usan funciones), no ignora tildes, sus
  operadores y textos no son los de la regla, no tiene "cerrar = cancelar"
  y en lazy dispara `onLazyLoad`, que el modo servidor no usa.

Página con lista completa (en memoria): solo declara los filtros.

```ts
protected readonly columnas: ColumnaTabla<Vendedor>[] = [
  { id: 'nombre', titulo: 'Nombre', valor: (v) => v.nombre, filtro: { tipo: 'condicion', dato: 'texto' } },
  { id: 'zona', titulo: 'Zona', valor: (v) => v.zona, filtro: { tipo: 'casillas' } },   // valores distintos
  { id: 'comision', titulo: '% comisión', valor: (v) => formatearPorcentaje(v.comision),
    valorFiltro: (v) => v.comision, filtro: { tipo: 'rango', dato: 'porcentaje' } },
  { id: 'habilitado', titulo: 'Situación', filtro: { tipo: 'booleano', si: 'Activo', no: 'Inactivo' } },
];
```

Página paginada en el servidor: las opciones de catálogo que carga al
entrar van en la columna (por eso `columnas` es un `computed`), guarda los
filtros con la búsqueda, el orden y la página, y consulta con la página 1.

```ts
protected readonly columnas = computed<ColumnaTabla<ProductoFila>[]>(() => [
  { id: 'marca', titulo: 'Marca', valor: (p) => p.marca,
    filtro: { tipo: 'casillas', opciones: this._marcas().map((m) => ({ valor: m.catalogo_id, etiqueta: m.descripcion })) } },
  { id: 'habilitado', titulo: 'Situación', filtro: { tipo: 'booleano', si: 'Activo', no: 'Inactivo' } },
]);
protected readonly filtros = this._estado.filtros;                // signal<FiltroListado[]>([])
protected readonly filtrando = computed(() => !!this.busqueda().trim() || this.filtros().length > 0);
protected filtrar(filtros: FiltroListado[]): void {
  this.filtros.set(filtros);
  this.primera.set(0);                                             // un filtro nuevo vuelve a la pagina 1
  this.consultar();
}
// servicio: agregarFiltros(parametros, filtro.filtros) junto a busqueda, orden y pagina
```

```html
<{prefijo}-tabla-listado [columnas]="columnas()" [totalRegistros]="total()" [filtrando]="filtrando()"
  [filtros]="filtros()" (cambioFiltros)="filtrar($event)" … />
```

El back (`FiltroCampoDto`, `FiltroListado` con lista blanca por entidad) y
el SP (parámetros explícitos por columna, listas JSON con `OPENJSON`, sin
SQL dinámico) siguen `dotnet-clean-architecture` (`domain-and-dtos.md`,
"Filtros por columna de los listados") y `sql-database-patterns`
(`sp-templates.md`, "Sp_Sel paginado con filtros por columna"). Pruebas
obligatorias en `testing.md`.

Origen: ComercialMs (2026-11-08), estilo "filterMenu" de PrimeNG/Excel con
el contrato y la lista blanca del orden por columna. Primer uso:
Productos (Marca y Línea con casillas, Situación con booleano).


## Modal reutilizable — `shared/ui/modal`

Regla dura: **todo modal usa este armazón**. Un `{entidad}-form` (o una
confirmación) nunca vuelve a escribir el velo, la caja, la cabecera, la X
ni el pie: solo aporta su contenido y sus botones.

| Lo pone el componente | Lo pone cada modal |
|---|---|
| Velo (`.modal-overlay`) y caja (`.modal`, `[ancho]`, `[alto]`) | Contenido: pestañas, `<form class="modal-cuerpo">` |
| Cabecera con el título-pestaña e id generado (`aria-labelledby`) | `titulo` (texto o expresión) |
| X de cierre (`[conCerrar]`, default `true`) → output `cerrar` | Qué hace al cerrar |
| `rol` (`dialog` / `alertdialog`), `describeA` (`aria-describedby`) | El id del mensaje que describe |
| Pie `.modal-pie` y altura animada (`AlturaSuaveDirective`) | Botones, proyectados con `ngProjectAs="[pie]"` |

```html
<!-- components/{entidad}-form/{entidad}-form.component.html -->
<{prefijo}-modal [titulo]="esEdicion ? 'Editar categoría' : 'Nueva categoría'" (cerrar)="cerrar.emit()">
  <form id="categoria-form" class="modal-cuerpo form-grid" [formGroup]="form" (ngSubmit)="guardar()" novalidate>
    <!-- campos -->
  </form>
  <ng-container ngProjectAs="[pie]">
    <button type="button" class="btn" (click)="cerrar.emit()">Cancelar</button>
    <button type="submit" form="categoria-form" class="btn btn-primario" [disabled]="guardando()">Guardar</button>
  </ng-container>
</{prefijo}-modal>
```

- `ngProjectAs` sobre un `ng-container` proyecta los botones al pie **sin
  agregar elementos** al DOM (el marcado queda igual que escrito a mano).
- Host del componente con `display: contents`: no aporta caja; las
  animaciones de salida de la página (`animate.leave` sobre el host del
  form) siguen encontrando `.modal`.
- Cierre solo con la X o con el botón del pie: ni click en el velo ni
  `Escape` (ver skill `angular-design-system`, "Modales: armazón, cierre y
  altura").
- **El velo corta la propagación del `submit`** (`<div class="modal-overlay"
  (submit)="$event.stopPropagation()">`). Con la atomicidad obligatoria, el
  modal de un detalle vive dentro del `-list`, que a su vez está dentro del
  `<form>` de la página: sin ese corte, el `submit` del form del modal sube
  hasta el form de la página y dispara su guardado (agregar un contacto
  guardaría el cliente). Va en el armazón compartido, una sola vez, con su
  prueba de regresión (`modal.component.spec.ts`: un form de página con un
  modal dentro; enviar el del modal no envía el de la página).
- Altura automática con tope y cambio de altura animado: `[alto]="true"`
  en modales con pestañas; la animación la pone la directiva compartida
  `shared/directives/altura-suave.directive.ts` (selector `.modal`,
  `ResizeObserver` + Web Animations, ~220ms, respeta
  `prefers-reduced-motion`), aplicada dentro del armazón.
- Eliminar usa por defecto la confirmación estándar (`Cancelar` / `Sí,
  eliminar`) sobre este mismo armazón, en `shared/ui/confirmar-eliminacion`.
- Al migrar modales existentes, verificar "sin cambio visual" comparando
  el DOM normalizado antes y después (como en el listado).

Origen: ComercialMs (2026-11-06), 7 modales migrados con DOM idéntico.

## Cascada de ubigeo reutilizable — `shared/ui/ubigeo-cascada`

Toda entidad con ubicación en Perú (sede, cliente, proveedor) elige su
distrito con **un solo componente compartido**, nunca con tres `p-select`
encadenados escritos en cada formulario. Se crea desde el primer uso:
la ubicación es previsiblemente de varias entidades.

Contrato del backend (lo que la cascada espera):

- `GET /api/Ubigeos` → lista plana y completa
  `{ ubigeo_id, codigo, nombre, nivel (1 dep, 2 prov, 3 dist), padre_id }`,
  IDs encriptados con cifrado **determinístico** (el `padre_id` de un hijo
  es igual al `ubigeo_id` de su padre), sin auditoría.
- La entidad guarda solo `ubigeo_id` = **distrito** (FK, validado en el
  SP como nivel 3 vigente) y devuelve además `departamento`, `provincia`
  y `distrito` de solo lectura (en SQL Server, una `fn_tvf_` inline con
  `OUTER APPLY` en Sel/Get, compartida por todas las entidades).

Piezas del front:

| Pieza | Ruta | Qué hace |
|---|---|---|
| Modelos | `shared/models/ubigeo.model.ts` | `Ubigeo` y `UbicacionUbigeo` (`ubigeo_id` + nombres), que extiende el modelo de cada entidad |
| Servicio | `shared/services/ubigeo.service.ts` | `getAll()` con cache en memoria (`shareReplay(1)`): una llamada por sesión para todas las cascadas; si falla, **no** cachea el error |
| Helpers | `shared/helpers/ubigeo.helpers.ts` | `indexarUbigeos` (por ID e hijos por padre, ordenados con `localeCompare(…, 'es')`) y `rutaDistrito` (departamento y provincia de un distrito) |
| Componente | `shared/ui/ubigeo-cascada/` | `{prefijo}-ubigeo-cascada`, `ControlValueAccessor` |

```ts
// shared/services/ubigeo.service.ts
getAll(): Observable<Ubigeo[]> {
  this._cache$ ??= this._api.get<Ubigeo[]>('/Ubigeos').pipe(
    map(res => { if (!res.success) throw res; return res.data ?? []; }),
    catchError(err => { this._cache$ = null; return throwError(() => err); }), // reintenta la proxima vez
    shareReplay(1),
  );
  return this._cache$;
}
```

```html
<!-- en el form de la entidad, dentro de su .form-grid -->
<{prefijo}-ubigeo-cascada formControlName="ubigeo_id" />
```

```ts
ubigeo_id: this._fb.control<string | null>(null, Validators.required), // valor = distrito o null
```

Reglas del componente:

- **Valor = `ubigeo_id` del distrito**, o `null` mientras no haya uno.
  Departamento y provincia son estado interno (signals), no viajan.
- Al cambiar departamento o provincia, el distrito vuelve a `null` (y lo
  emite). Provincia se habilita con departamento; distrito con provincia.
- En edición, `writeValue(distrito)` ubica solo departamento y provincia
  (`rutaDistrito`); si el catálogo aún no llegó, lo hace al cargarlo.
- **Host con `display: contents`**: sus tres `.campo` se suman a la
  `form-grid` del formulario que lo usa (en 2 columnas: Departamento |
  Provincia, Distrito | siguiente campo). No trae grilla propia.
- **Error bajo Distrito, igual que el resto de campos**: en vez de
  `NG_VALUE_ACCESSOR` se inyecta `NgControl` (`{ self: true, optional:
  true }`) y se hace `ngControl.valueAccessor = this` en el constructor;
  se escucha `control.events` (incluye `markAllAsTouched` del submit) para
  `invalid && touched` y el texto sale de `getControlError`. `onTouched`
  se llama en el blur del distrito, no de los otros dos (si no, el error
  aparece apenas se elige el departamento).
- Los `p-select` internos usan `[ngModel]` + `(ngModelChange)` sobre los
  signals (no se registran en el `formGroup` padre), `[filter]`,
  `[showClear]`, `appendTo="body"` y `[invalid]` del propio PrimeNG.
- Input `idBase` (default `'ubigeo'`) para los `inputId`/`label for`, por
  si un formulario tiene dos ubicaciones.
- Nombres tal cual la fuente oficial (INEI, mayúsculas): no se pasan a
  "Formato Título" (deja mal "De", tildes, etc.).
- El listado de la entidad muestra la ubicación en una columna propia
  (distrito y, debajo, provincia · departamento; "Sin ubicación" si es
  null) y el buscador local también filtra por esos nombres.

Origen: ComercialMs (2026-11-06), CRUD de sedes; previsto para clientes
y proveedores.

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

**Obligatorio** (ver "Atomicidad obligatoria" arriba). Si una sección es en
sí misma un CRUD (ej. un catálogo anidado dentro del expediente), sigue el
patrón típico internamente — `-list` (tabla) y el modal de registrar/editar
(`-form` envuelto en `<{prefijo}-modal>`; `-fields` si esos mismos campos
también se usan fuera del modal, ej. en el alta del padre) como componentes
hermanos — y sus `models/`/`services/` viven a la altura de la carpeta que
agrupa a ambos hermanos, no anidados un nivel más adentro ni subidos al
nivel de la entidad padre:

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
