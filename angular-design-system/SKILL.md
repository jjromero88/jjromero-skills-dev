---
name: angular-design-system
description: Sistema de diseño visual para apps Angular — monocromático (inspirado en la documentación de OpenAI y en el estilo actual de shadcn), escala de grises invertida por tema, dark mode sin flash, tipografía Inter+JetBrains Mono, componentes (botones/cards/alerts/modales/tablas/notificaciones/KPIs/charts) construidos sobre tokens CSS. Úsala junto con `angular-feature-architecture` para darle estilo visual a cualquier vista nueva.
---

# Angular Design System

> ⚠️ Skill personal, no genérica de diseño. No aplicar a proyectos que no
> sigan este estilo.

Basada en un sistema de diseño real, revisado a fondo el 2026-07-19. Estilo
monocromático, inspirado en la documentación de OpenAI y en el estilo
actual de shadcn — el color es funcional y escaso, nunca decorativo. Ante
conflicto entre esta skill y tu código real, gana el código real.

## Motor

- Monocromático: escala de grises como base, un puñado de colores de acento
  con un único trabajo cada uno (nunca decorativos).
- La jerarquía visual se construye con **borde**, no con sombra — las
  sombras están limitadas a un puñado de casos concretos.
- Dark mode real (no solo "invertir colores"): tokens CSS que se redefinen
  por tema, mismo nombre de variable en ambos.
- Se usa junto a `angular-feature-architecture` (arquitectura/estructura) —
  esta skill cubre solo lo visual.

## Workflow

1. ¿Workspace nuevo o primer setup visual? → `references/tokens-typography-and-dark-mode.md` (tokens, tipografía, mecanismo de dark mode).
2. ¿Sidebar, topbar, nav secundaria de una vista seccionada, o login? → `references/layout-navigation-and-toolbar.md`.
3. ¿Botones, cards, alerts, modal de confirmación, o el foco de un input? → `references/components-buttons-cards-alerts-and-modals.md`.
4. ¿Una tabla/grilla/lista? → `references/grids-and-tables.md`.
5. ¿Notificaciones (toasts)? → `references/notifications.md`.
6. ¿KPIs con conteo animado, o un chart? → `references/kpi-cards-and-charts.md`.

## Reglas duras

- **Nunca un color fuera de la paleta funcional** — todo color con
  significado (éxito, error, advertencia, info) tiene un token con nombre;
  nunca un hex suelto en un componente.
- **Paridad dark/light solo vía `data-theme`** — los componentes nunca
  ramifican en TypeScript según el tema; todo el cambio de color ocurre por
  redefinición de custom properties CSS bajo `[data-theme=light]`/
  `[data-theme=dark]`.
- **Tipografía con un rol claro**: fuente de UI/prosa vs. fuente mono para
  todo identificador técnico/numérico (montos, códigos, IDs). Nunca mezclar
  sin motivo.
- **Borde antes que sombra.** Un puñado fijo de sombras permitidas en todo
  el sistema (ver `components-buttons-cards-alerts-and-modals.md`) — no
  agregues una sombra nueva sin ese mismo criterio.
- **Botón primario = inversión de escala de grises**, nunca un color de
  marca — refuerza el monocromatismo incluso en el elemento de mayor
  énfasis.
- **Botón async siempre con spinner inline** mientras la acción está en
  vuelo — deshabilitado durante la espera.
- **Colores de serie de un chart atados a tokens de acento con nombre** —
  nunca un hex hardcodeado en la configuración del chart.
- **Un componente reutilizable (KPI card, notificación, paginador) se
  reutiliza siempre** — ver regla de reutilización obligatoria en
  `angular-feature-architecture`.

## Referencias

- `references/tokens-typography-and-dark-mode.md` — escala de grises, tokens semánticos, paleta de acento, tipografía, mecanismo de dark mode.
- `references/layout-navigation-and-toolbar.md` — sidebar, topbar, nav secundaria de vistas seccionadas, login.
- `references/components-buttons-cards-alerts-and-modals.md` — botones, cards, alerts, modal de confirmación, foco, radios, sombras.
- `references/grids-and-tables.md` — tablas, badges, columnas responsive, KPI stat-box, fila expandible, empty state, skeleton.
- `references/notifications.md` — servicio y componente de toasts.
- `references/kpi-cards-and-charts.md` — KPI con conteo animado, Chart.js + ng2-charts.
- Skill `angular-feature-architecture` — arquitectura y estructura de carpetas donde vive todo esto.
