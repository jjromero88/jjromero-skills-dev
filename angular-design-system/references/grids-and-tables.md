# Grids and Tables

## Restyle de `p-table`

`p-table` de PrimeNG se usa para listados con paginación real, pero
**restyleado completo vía overrides globales** en `styles.css` — nunca se
usan los estilos por defecto de PrimeNG a la vista:

```css
.p-datatable-thead > tr > th {
  padding: 12px 16px; font-size: 12px; font-weight: 500;
  text-transform: uppercase; letter-spacing: 0.05em;
  color: color-mix(in srgb, var(--color-text-tertiary) 65%, transparent);
  background: var(--color-surface);
}
.p-datatable-tbody > tr {
  border-bottom: 1px solid color-mix(in srgb, var(--color-border) 50%, transparent);
  transition: background-color 150ms;
}
.p-datatable-tbody > tr:hover { background: var(--color-primary-ghost); }
```

**Sin zebra-striping** — solo resalte al hacer hover de la fila. Headers
en mayúsculas, 12px, mono-espaciado no (siguen en la fuente de UI); las
celdas numéricas sí van en la fuente mono (ver más abajo).

## Columnas responsive

Clases utilitarias globales, aplicadas directo en `<th>`/`<td>`:

```css
.col-hide-sm { }
.col-hide-md { }
@media (max-width: 768px)  { .col-hide-sm { display: none !important; } }
@media (max-width: 1024px) { .col-hide-md { display: none !important; } }
```

## Celdas

- **Celda compuesta avatar+nombre**: círculo de iniciales (32px) + nombre
  (14px/500) + subtítulo secundario debajo (correo, código, etc.).
- **Celda numérica** (moneda, cantidades): fuerza `font-family:
  var(--fuente-mono); font-weight: 580;` — nunca la fuente de UI para un
  número relevante.
- **Badges de estado** — pill-shaped, tokenizados (variante canónica; no
  uses la variante alternativa de `ring` con clases Tailwind literales,
  aunque la veas en código viejo — es una inconsistencia a evitar, no un
  caso de uso distinto):

```css
.badge { display: inline-flex; align-items: center; gap: 4px; padding: 2px 10px; border-radius: 9999px; font-size: 12px; }
.badge-success { color: #10b981; background: color-mix(in oklab, #10b981 12%, transparent); }
.badge-error   { color: #ef4444; background: color-mix(in oklab, #ef4444 12%, transparent); }
.badge-warning { color: #f59e0b; background: color-mix(in oklab, #f59e0b 12%, transparent); }
```

- **Columna de acciones**: botón de 3 puntos (32×32, transparente hasta
  hover) que abre un dropdown propio (o `p-menu` popup) con editar/eliminar
  — eliminar siempre en rojo, con un separador antes.

## KPI stat-box (dentro de una tabla/sección, no la KPI card principal)

Fila de 2-4 cajas resumen sobre la tabla: chip de ícono con color
semántico + número grande en mono + etiqueta pequeña en mayúsculas debajo.
Distinto de la KPI card de dashboard (ver `kpi-cards-and-charts.md`), pero
mismo lenguaje visual (color solo en el chip de ícono, el resto neutral).

## Fila expandible

Para mostrar el detalle de una fila sin navegar a otra vista — un `trigger()`
de Angular, no CSS Grid (a diferencia del sidebar):

```ts
export const expandRowAnimation = trigger('expandRow', [
  transition(':enter', [
    style({ opacity: 0, transform: 'translateY(-10px)' }),
    animate('700ms cubic-bezier(0.4, 0, 0.2, 1)', style({ opacity: 1, transform: 'translateY(0)' })),
  ]),
  transition(':leave', [
    animate('700ms cubic-bezier(0.4, 0, 0.2, 1)', style({ opacity: 0, transform: 'translateY(-10px)' })),
  ]),
]);
```

Vive en `shared/animations/` (ver skill `angular-feature-architecture`,
`references/project-structure.md`) — reutilizable en cualquier tabla que
necesite expandir una fila.

## Empty state

Toda tabla implementa su estado vacío explícitamente (nunca una tabla en
blanco sin mensaje): ícono grande neutral + título + subcopy corto,
centrado en el área de la tabla.

## Loading state

Barras de skeleton pulsante (`animate-pulse`, opacidad reducida) en el
área de la tabla mientras carga — no un spinner centrado que reemplace
toda la tabla.
