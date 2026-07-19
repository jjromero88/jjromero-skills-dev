# Tokens, Typography and Dark Mode

## Filosofía

La interfaz es una escala de grises pura. El color es funcional y escaso:
solo aparece para significar algo (éxito, error, advertencia, foco) —
nunca decorativo. Todo custom property CSS, nunca SCSS compilado, para que
el toggle de tema funcione sin recompilar.

## Escala de grises — 24 pasos, invertida por tema

El truco central: el **mismo nombre de token** resuelve a un valor distinto
según el tema — los componentes nunca ramifican en TS, solo referencian
`var(--gray-XXX)`.

```css
:where([data-theme=light]) {
  --gray-0:#fff; --gray-25:#fcfcfc; --gray-50:#f9f9f9; --gray-75:#f3f3f3;
  --gray-100:#ededed; --gray-150:#dfdfdf; --gray-200:#cdcdcd; --gray-250:#b9b9b9;
  --gray-300:#afafaf; --gray-350:#9f9f9f; --gray-400:#8f8f8f; --gray-450:#767676;
  --gray-550:#4f4f4f; --gray-600:#414141; --gray-650:#393939; --gray-700:#303030;
  --gray-750:#282828; --gray-800:#212121; --gray-850:#1c1c1c; --gray-900:#181818;
  --gray-925:#161616; --gray-950:#131313; --gray-975:#101010; --gray-1000:#0d0d0d;
  --alpha-base:#0d0d0d; --page-bg:#fff; --sidebar-bg:#fff; --header-bg:#fff;
}
:where([data-theme=dark]) {
  --gray-0:#0d0d0d; --gray-25:#101010; --gray-50:#131313; --gray-75:#161616;
  --gray-100:#181818; --gray-150:#1c1c1c; --gray-200:#212121; --gray-250:#282828;
  --gray-300:#303030; --gray-350:#393939; --gray-400:#414141; --gray-450:#4f4f4f;
  --gray-550:#767676; --gray-600:#8f8f8f; --gray-650:#9f9f9f; --gray-700:#afafaf;
  --gray-750:#b9b9b9; --gray-800:#cdcdcd; --gray-850:#dcdcdc; --gray-900:#ededed;
  --gray-925:#f3f3f3; --gray-950:#f3f3f3; --gray-975:#f9f9f9; --gray-1000:#fff;
  --alpha-base:#fff; --page-bg:#000; --sidebar-bg:#000; --header-bg:#000;
}
```

`--alpha-base` es la clave: siempre igual al color de texto del tema activo
(`#0d0d0d` en light, `#fff` en dark). Toda superficie/borde se construye
con `color-mix(in oklab, var(--alpha-base) N%, transparent)` — **nunca
`rgba()` a mano**:

- Fondos suaves: 2–4%.
- Hover/pills: 5–8%.
- Énfasis: 10–12%.
- Bordes: 6% (sutil) / 12% (normal).

## Tokens semánticos

```css
--color-text:           var(--gray-850);
--color-text-secondary: var(--gray-700);
--color-text-tertiary:  var(--gray-550);
--color-text-emphasis:  var(--gray-1000);
--color-surface:        var(--gray-200);
--color-surface-raised: var(--gray-150);
--color-border:         color-mix(in oklab, var(--alpha-base) 12%, transparent);
--color-border-subtle:  color-mix(in oklab, var(--alpha-base) 6%, transparent);
--color-primary-ghost:        color-mix(in oklab, var(--alpha-base) 6%, transparent);
--color-primary-ghost-hover:  color-mix(in oklab, var(--alpha-base) 8%, transparent);
--color-code-bg:  var(--gray-75);
--color-ring:     #0285ff;
--radius-sm:.375rem; --radius-md:.5rem; --radius-lg:.625rem; --radius-xl:.75rem;
```

## Paleta de acento funcional — únicos colores permitidos, cada uno con un solo trabajo

| Color | Hex | Uso exclusivo |
|---|---|---|
| Azul | `#0285ff` | Anillo de foco (`--color-ring`), info |
| Verde | `#10b981` | Éxito |
| Ámbar | `#f59e0b` | Advertencia leve/pendiente |
| Naranja | `#f97316` | Alerta de advertencia — nunca error/eliminación |
| Rojo | `#ef4444` | Peligro/error/eliminación — el único color de esa función |
| Violeta | `#a855f7` | Uso puntual (ej. un tipo de badge secundario) |

Aplicación siempre igual: `background: color-mix(in oklab, HEX 8-15%, transparent); border-color: color-mix(in oklab, HEX 25-30%, transparent); color: HEX;` —
nunca un bloque de color sólido, salvo el botón de peligro sólido (ver
`components-buttons-cards-alerts-and-modals.md`).

## Tipografía

```css
--fuente-ui:   'Inter', system-ui, -apple-system, sans-serif;
--fuente-mono: 'JetBrains Mono', 'Cascadia Code', Consolas, monospace;
```

Cargadas vía Google Fonts (`<link>` en `index.html`, pesos 400/500/600/700
para Inter, 400/600/700 para JetBrains Mono) — sin auto-hospedar.

**Regla mono**: todo identificador técnico o numérico (montos, códigos,
IDs, celdas de tabla numéricas) va en JetBrains Mono. Prosa y UI en Inter.
Si el usuario prefiere otra pareja de fuentes, mantén la misma disciplina
de roles (UI/prosa vs. técnico/numérico), solo cambia la fuente.

Escala tipográfica: 7 pasos, de `--font-xs` (.75rem) a `--font-3xl`
(1.875rem). Base del cuerpo: `--font-sm` (14px).

## Dark mode — mecanismo (clase + atributo + localStorage + anti-flash)

Default oscuro. Antes de que Angular bootstree, un script inline en
`index.html` fija el tema para evitar el flash:

```html
<html lang="es" data-theme="dark" class="dark">
<head>
  <script>
    (function () {
      try {
        var t = localStorage.getItem('theme');
        if (!t) t = window.matchMedia('(prefers-color-scheme: light)').matches ? 'light' : 'dark';
        document.documentElement.setAttribute('data-theme', t);
        document.documentElement.classList.toggle('dark', t === 'dark');
      } catch (e) { /* dark por defecto */ }
    })();
  </script>
```

Servicio de tema, signal-based, con el truco de "congelar transiciones" al
alternar (evita el cross-fade desordenado de decenas de propiedades
transicionando a la vez):

```ts
@Injectable({ providedIn: 'root' })
export class ThemeService {
  actual = signal<'dark' | 'light'>(
    (document.documentElement.getAttribute('data-theme') as 'dark' | 'light') ?? 'dark'
  );

  alternar(): void {
    const nuevo = this.actual() === 'dark' ? 'light' : 'dark';
    const congelar = document.createElement('style');
    congelar.textContent = '* { transition: none !important; }';
    document.head.appendChild(congelar);
    document.documentElement.setAttribute('data-theme', nuevo);
    document.documentElement.classList.toggle('dark', nuevo === 'dark');
    void getComputedStyle(congelar).opacity; // fuerza reflow
    congelar.remove();
    localStorage.setItem('theme', nuevo);
    this.actual.set(nuevo);
  }
}
```

Ningún componente ramifica en TypeScript por tema (salvo casos puntuales de
config de terceros que no leen CSS custom properties, ej. Chart.js — ver
`kpi-cards-and-charts.md`) — todo el cambio de color ocurre por
redefinición de tokens bajo `[data-theme=...]`.
