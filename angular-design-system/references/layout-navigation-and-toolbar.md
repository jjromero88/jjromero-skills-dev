# Layout, Navigation and Toolbar

## Sidebar — 3 niveles, data-driven, acordeón CSS Grid

Se construye un árbol `NavItem[]` a partir de un array plano de opciones de
menú que devuelve el backend (`padre_id`/`orden`) — nunca rutas
hardcodeadas en el sidebar. Profundidad confirmada: **padre → hijo →
nieto** (3 niveles), el nivel más profundo siempre es una hoja con
`routerLink`.

**Acordeón exclusivo por nivel**: al expandir un item, se colapsan todos
sus hermanos (recursivo) antes de abrir el clickeado — nunca multi-expand
en el mismo nivel. Al navegar, se re-sincroniza el árbol para expandir la
rama que contiene la ruta activa.

**Animación — CSS Grid, no `trigger()` de Angular**:

```css
.sidebar-submenu {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 200ms ease-out;
}
.sidebar-submenu--open { grid-template-rows: 1fr; }
.sidebar-submenu-inner { overflow: hidden; }
```

```html
<div class="sidebar-submenu" [class.sidebar-submenu--open]="isExpanded(item.id)">
  <div class="sidebar-submenu-inner ml-4 mt-0.5 border-l ...">...</div>
</div>
```

El chevron rota con una clase Tailwind (`[class.rotate-90]="isExpanded(item.id)"`
+ `transition-transform duration-200`), no con una animación de Angular.

Sí existe una animación real de Angular (`trigger()`), pero para otro caso:
filas de tabla que se expanden (ver `grids-and-tables.md`), fade+slide de
700ms — y para el fade entre secciones de una vista seccionada (ver más
abajo), 340ms.

## Topbar

Barra oscura (`background: var(--header-bg)`, negro en ambos temas — ver
tokens). Estructura izquierda→derecha:

- Botón hamburguesa (toggle del sidebar).
- Toggle de tema (ícono sol/luna según `themeService.actual()`).
- **Selector de entidad/compañía** — solo si la app es multi-tenant (ver
  skill `angular-feature-architecture`, `references/auth-and-security.md`)
  y el usuario tiene acceso a 2+; oculto en mobile.
- Avatar con iniciales (sin foto), nombre (oculto en mobile), chevron.
- Dropdown (no un componente de PrimeNG, uno propio): cabecera con
  nombre+correo, "Mi Perfil", "Configuración", separador, "Cerrar Sesión"
  en rojo (`color: #ef4444`, hover con fondo tintado del mismo color).

Sin campana de notificaciones ni breadcrumbs en el topbar — los
breadcrumbs van al inicio de cada página, no en la barra superior.

## Nav secundaria sticky — vistas seccionadas extensas

Para el caso complejo de `angular-feature-architecture`
(`references/component-patterns.md`, "Caso complejo"): una vista con muchas
secciones (formulario de alta/edición extenso) usa una nav secundaria
propia, no tabs de PrimeNG ni un stepper.

- **Desktop**: columna sticky de `220px` (`sticky top-6`) a la izquierda,
  contenido a la derecha (`flex-1`). Los links de la nav se agrupan en
  bloques con encabezado en mayúsculas pequeñas (`uppercase text-xs
  tracking-wide`), ej. "Datos", "Detalle", "Registros".
- **Mobile**: la misma nav colapsa a una fila de tabs horizontal
  scrolleable (`overflow-x-auto`, subrayado en el activo).
- **Error surfacing**: cada item de la nav se pinta en rojo si la sección
  correspondiente del formulario tiene campos inválidos — se computa un
  set de "secciones con error" reactivamente a partir del estado del
  formulario.
- **Transición entre secciones**: fade + slide corto al cambiar de sección
  (`trigger('fadeIn', [transition(':enter', [style({opacity:0,
  transform:'translateY(1px)'}), animate('340ms ease-in', style({opacity:1,
  transform:'translateY(0)'}))])])`), aplicado al contenedor de la sección
  activa.
- **Barra de acciones sticky al pie**: Cancelar/Guardar, siempre visible
  (`sticky bottom-0`), independiente del scroll del contenido.
- Cada sub-sección de contenido es una card (`rounded-xl border p-6`,
  nunca sombra — ver `components-buttons-cards-alerts-and-modals.md`), con
  un título + subtítulo corto arriba, y un grid responsive de 2 columnas
  para los campos (`grid-cols-1 sm:grid-cols-2 gap-x-6 gap-y-5`).

## Login — clásico, centrado, monocromático

El login **no** replica el patrón de vistas internas (sidebar/topbar) —
es una pantalla clásica: card centrada verticalmente, campos de
usuario/contraseña, botón primario de ancho completo, mismo estilo
monocromático (fondo `--page-bg`, card con borde `--color-border`, sin
imágenes/ilustraciones decorativas). La lógica de autenticación (login en
2 fases, multi-tenant) vive en la skill `angular-feature-architecture` —
esto es solo el layout visual de la pantalla.
