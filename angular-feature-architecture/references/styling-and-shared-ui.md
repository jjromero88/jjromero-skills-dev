# Styling and Shared UI

Este archivo cubre la mecánica (qué se instala, cómo se conecta). El
detalle visual completo — colores, tipografía, dark mode, botones, cards,
tablas, notificaciones, KPIs y charts — vive en la skill
`angular-design-system`; aquí se linkea, no se duplica.

## Tailwind CSS, sin SCSS

Un único `src/styles.css` global con `@import "tailwindcss"`. Cero archivos
`.scss` en el proyecto. Tokens de diseño (colores, espaciados) como CSS
custom properties en `:root` (contenido exacto en skill
`angular-design-system`, `references/tokens-typography-and-dark-mode.md`).

**Regla de promoción a global**: un bloque de utilidades Tailwind que se
repite en 2+ vistas se promueve a una clase global en `styles.css` — no
antes. Ni un componente se queda con un estilo particular que en realidad
comparte con otro; eso viola la regla de reutilización obligatoria (ver
`SKILL.md`/`project-structure.md`).

**Excepción real y puntual**: un archivo `.css` por componente se acepta
solo cuando Tailwind no puede expresar bien la técnica necesaria — el
caso real es una transición `grid-template-rows: 0fr → 1fr` para un
acordeón de expandir/colapsar (ver skill `angular-design-system`,
`references/layout-navigation-and-toolbar.md`). No es el patrón por
defecto; no crees un `.css` de componente salvo un caso equivalente y
justificado.

## PrimeNG — librería de UI por defecto (confirmar siempre)

Ver "Antes de crear" en `SKILL.md`: PrimeNG es la sugerencia por defecto
(última estable compatible con la versión de Angular elegida), pero se
confirma siempre con el usuario antes de instalarla.

Todo componente de formulario/tabla/diálogo usa PrimeNG (`p-table`,
`p-dialog`, `p-select`, `p-datepicker`, etc. — ver `forms-and-validation.md`
para el detalle de inputs). Tema custom vía `definePreset()` sobre el
preset base `Aura` de `@primeuix/themes`, configurado una sola vez en
`app.config.ts`, alineado a los tokens de la skill `angular-design-system`:

```ts
providePrimeNG({
  theme: {
    preset: definePreset(Aura, { semantic: { primary: { /* paleta del design system */ } } }),
    options: { darkModeSelector: '[data-theme="dark"]' },
  },
  translation: { /* i18n de fechas/meses en español, centralizado aquí */ },
}),
```

La localización de fecha/calendario (nombres de día/mes, formato) se
configura una sola vez en este objeto — no por componente.

## Iconos — Boxicons por defecto (confirmar siempre)

Ver "Antes de crear" en `SKILL.md`: Boxicons es la sugerencia por defecto,
pero se confirma siempre con el usuario (y si confirma, se pregunta también
la versión).

Uso vía clase CSS, sin wrapper de componente:

```html
<i class="bx bx-trash"></i>
<i class="bx bx-pencil"></i>
<i class="bx bx-plus"></i>
```

Se instala como paquete `boxicons` e importa su CSS una sola vez en
`styles.css` o `angular.json` (`styles: ["node_modules/boxicons/css/boxicons.min.css"]`).
Si el usuario prefiere otra librería de iconos, se usa esa en su lugar con
el mismo criterio (una sola vez configurada, nunca mezclando 2 librerías de
iconos sin motivo).

## Charts — Chart.js + ng2-charts por defecto

Paquetes por defecto para gráficos estadísticos: `chart.js` + `ng2-charts`
(directiva `BaseChartDirective`, `<canvas baseChart [type] [data] [options]>`).
Configuración y estilo exactos (colores de serie atados a tokens, grillas,
tooltips, fuente) en skill `angular-design-system`,
`references/kpi-cards-and-charts.md` — no se hardcodean colores de serie
directamente en el componente.

## `shared/directives/` — directiva de conteo animado

Parte del esqueleto inicial: una directiva `[appCountUp]` hecha a mano
(sin paquete npm) para el efecto de conteo ascendente en tarjetas KPI.
Detalle completo de implementación en skill `angular-design-system`,
`references/kpi-cards-and-charts.md`.

## `shared/ui/` — componentes presentacionales genéricos

Solo entran aquí componentes verdaderamente reutilizables entre features
(ej. una tarjeta de KPI, el host de notificaciones, un paginador). El host
de notificaciones es un singleton real: se instancia una sola vez en el
componente raíz (`app.component.ts`), y reacciona a los signals de un
servicio de notificaciones — cualquier componente de cualquier feature lo
dispara con `notificaciones.exito(msg)` / `.error(msg)` / `.info(msg)` sin
tener que renderizarlo él mismo. API completa y estilo exacto en skill
`angular-design-system`, `references/notifications.md`.

No existe carpeta `pipes/` — el formateo (fechas, moneda, etc.) se
implementa como método del propio componente que lo necesita, no como pipe
compartido. No extraigas un pipe compartido salvo que el usuario lo pida
explícitamente. `shared/directives/` sí existe (ver arriba) — es la única
excepción real a "sin pipes/directives por defecto".
