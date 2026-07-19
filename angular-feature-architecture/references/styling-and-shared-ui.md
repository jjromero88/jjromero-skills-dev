# Styling and Shared UI

## Tailwind CSS, sin SCSS

Un único `src/styles.css` global con `@import "tailwindcss"`. Cero archivos
`.scss` en el proyecto. Tokens de diseño (colores, espaciados) como CSS
custom properties en `:root`, y una pequeña catalogación de clases de
utilidad reusables (`.badge`, `.btn`, `.form-field`) directamente en ese
archivo global — se promueve un patrón a clase global recién a partir de
3+ apariciones repetidas del mismo bloque de utilidades Tailwind, no antes.

**Excepción real y puntual**: un archivo `.css` por componente se acepta
solo cuando Tailwind no puede expresar bien la técnica necesaria — el
único caso real es una transición `grid-template-rows: 0fr → 1fr` para un
acordeón de expandir/colapsar. No es el patrón por defecto; no crees un
`.css` de componente salvo un caso equivalente y justificado.

## PrimeNG — librería de UI obligatoria

Todo componente de formulario/tabla/diálogo usa PrimeNG
(`p-table`, `p-dialog`, `p-select`, `p-datepicker`, etc. — ver
`forms-and-validation.md` para el detalle de inputs). Tema custom vía
`definePreset()` sobre el preset base `Aura` de `@primeuix/themes`,
configurado una sola vez en `app.config.ts`:

```ts
providePrimeNG({
  theme: {
    preset: definePreset(Aura, {
      semantic: {
        primary: { /* paleta custom */ },
      },
    }),
  },
  translation: { /* i18n de fechas/meses en español, centralizado aquí */ },
}),
```

La localización de fecha/calendario (nombres de día/mes, formato) se
configura una sola vez en este objeto — no por componente.

## Iconos

PrimeIcons para lo cubierto por el set estándar de PrimeNG; Lucide Angular
para lo que falte, importado explícitamente por componente (no global) vía
el provider multi de iconos:

```ts
providers: [
  { provide: LUCIDE_ICONS, multi: true, useValue: new LucideIconProvider({ Trash2, Pencil, Plus }) },
],
```

Cada componente declara solo los íconos Lucide que realmente usa — no un
import global de todo el set.

## `shared/ui/` — componentes presentacionales genéricos

Solo entran aquí componentes verdaderamente reutilizables entre features
(ej. una tarjeta de KPI, un diálogo de notificación/confirmación global, un
paginador). El diálogo de notificación es un singleton real: se instancia
una sola vez en el componente raíz (`app.component.ts`), junto al
`<router-outlet />`, y reacciona a los signals de `NotificationService` —
cualquier componente de cualquier feature lo dispara con
`notificationService.success(msg)` / `.error(msg)` / `.warn(msg)` /
`.confirm({...})` sin tener que renderizarlo él mismo.

No existen carpetas `pipes/`/`directives/` — el formateo (fechas, moneda,
etc.) se implementa como método del propio componente que lo necesita, no
como pipe compartido. No extraigas un pipe compartido salvo que el usuario
lo pida explícitamente: duplicar el método es el patrón real hoy.
