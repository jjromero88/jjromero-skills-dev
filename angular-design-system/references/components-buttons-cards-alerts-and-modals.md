# Buttons, Cards, Alerts and Modals

## Radios y bordes

4 pasos de radio: `--radius-sm` (6px) · `--radius-md` (8px) · `--radius-lg`
(10px) · `--radius-xl` (12px).

Bordes de 1px en todas partes, siempre `color-mix(in oklab, var(--alpha-base) 6%|12%, transparent)`
— nunca un gris plano en hex, nunca `rgba()`, nunca más grueso que 1px. Es
el rasgo más identitario del estilo: **la jerarquía se construye con
borde, no con sombra.**

## Sombras — solo un puñado permitido en todo el sistema

Casos reales: header al hacer scroll, un dropdown de búsqueda, un botón
flotante de "volver arriba", y el toast (ver `notifications.md`). Todas
con la misma fórmula: `0 Npx Mpx color-mix(in oklab, var(--alpha-base) 10-16%, transparent)`
— nunca una sombra negra plana, nunca coloreada. El modal reutiliza la
misma fórmula (`0 8px 24px alpha 12%`) aunque no es uno de los 4 casos
"canónicos" — mantiene la misma gramática visual.

## Botones

```css
.btn {
  display: inline-flex; align-items: center; gap: 8px; padding: 8px 16px;
  border: 1px solid var(--color-border); border-radius: var(--radius-md);
  background: none; color: var(--color-text); font: 500 var(--font-sm) var(--fuente-ui);
  cursor: pointer; transition: background .15s;
}
.btn:hover:not(:disabled) { background: var(--color-primary-ghost); color: var(--color-text-emphasis); }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.btn-primario {
  background: var(--color-text-emphasis); color: var(--page-bg);
  border-color: transparent; font-weight: 600;
}
.btn-tonal {
  background: color-mix(in oklab, var(--alpha-base) 9%, transparent);
  color: var(--color-text-emphasis); border-color: transparent; font-weight: 600;
}
.btn-danger { color: #ef4444; border-color: color-mix(in oklab, #ef4444 30%, transparent); }
.btn-danger-solido { background: #ef4444; color: #fff; border-color: transparent; font-weight: 600; }
```

Taxonomía de variantes: **default** (`.btn`, bordeado, transparente) →
**primario** (`.btn-primario`, inversión sólida de la escala de grises —
nunca un color de marca, ni siquiera para el botón de mayor énfasis) →
**tonal** (`.btn-tonal`, un escalón debajo del primario) → **ghost**
(borde transparente) → **danger outline** (`.btn-danger`) → **danger
sólido** (`.btn-danger-solido`, el único botón realmente "a color" del
sistema, reservado para confirmar una acción destructiva).

**Botón async — spinner obligatorio**:

```css
.spinner {
  display: inline-block; width: 14px; height: 14px;
  border: 2px solid var(--color-border); border-top-color: var(--color-text-emphasis);
  border-radius: 50%; animation: girar .7s linear infinite;
}
@keyframes girar { to { transform: rotate(360deg); } }
```

Patrón: un signal `cargando`, `[disabled]="cargando()"`, y
`@if (cargando()) { <span class="spinner"></span> } @else { <i class="bx ..."></i> }`.
Todo botón que dispara una acción async lo sigue, sin excepción.

## Cards

```css
.card {
  border: 1px solid var(--color-border); border-radius: var(--radius-xl);
  background: color-mix(in oklab, var(--alpha-base) 2%, transparent);
}
.card-header {
  background: color-mix(in oklab, var(--alpha-base) 4%, transparent);
  border-bottom: 1px solid var(--color-border-subtle);
  padding: 14px 18px;
}
```

Jerarquía por superficie (tinte más oscuro/claro), nunca por color ni por
sombra pesada.

## Alerts (por severidad)

```css
.alert { display: flex; gap: 10px; padding: 12px 14px; border: 1px solid; border-radius: var(--radius-md); font-size: var(--font-sm); }
.alert-info    { color:#0285ff; background:color-mix(in oklab,#0285ff 8%,transparent); border-color:color-mix(in oklab,#0285ff 25%,transparent); }
.alert-warn    { color:#f97316; background:color-mix(in oklab,#f97316 8%,transparent); border-color:color-mix(in oklab,#f97316 25%,transparent); }
.alert-danger  { color:#ef4444; background:color-mix(in oklab,#ef4444 8%,transparent); border-color:color-mix(in oklab,#ef4444 25%,transparent); }
.alert-tip     { color:#10b981; background:color-mix(in oklab,#10b981 8%,transparent); border-color:color-mix(in oklab,#10b981 25%,transparent); }
```

**Anti-patrón**: nunca apilar varios alerts de colores distintos en la
misma vista — usa un contenedor neutral con un ícono de color por fila en
vez de varios bloques coloreados uno tras otro.

## Modal de confirmación destructiva — "escribe el nombre para confirmar"

Para eliminar un recurso con nombre (no un simple registro de catálogo,
sino algo con peso — ej. un proyecto, un expediente completo): el botón de
eliminar deshabilitado hasta que el texto escrito coincide exactamente con
el nombre del recurso.

```html
<div class="modal-overlay" (click)="cerrarEliminar()">
  <div class="modal" (click)="$event.stopPropagation()">
    <div class="modal-cab">Eliminar {{ recurso.nombre }}</div>
    <div style="padding: 18px;">
      <div class="alert alert-danger">Esto elimina el recurso completo. <strong>No se puede deshacer.</strong></div>
      <div class="campo" style="margin-top: 14px;">
        <label>Escribe <span class="nombre-confirmar">"{{ recurso.nombre }}"</span> para confirmar</label>
        <input type="text" [(ngModel)]="confirmacion" [placeholder]="recurso.nombre" autocomplete="off" />
      </div>
    </div>
    <div class="modal-pie">
      <button class="btn" (click)="cerrarEliminar()">Cancelar</button>
      <button class="btn btn-danger-solido" [disabled]="!confirmaCoincide() || eliminando()" (click)="eliminar()">
        @if (eliminando()) { <span class="spinner"></span> } Eliminar
      </button>
    </div>
  </div>
</div>
```

```css
.modal-overlay { position: fixed; inset: 0; z-index: 60; background: color-mix(in oklab, #000 55%, transparent); display: grid; place-items: center; padding: 24px; }
.modal { width: min(560px, 92vw); max-height: 76vh; display: flex; flex-direction: column;
  background: var(--page-bg); border: 1px solid var(--color-border); border-radius: var(--radius-xl); overflow: hidden;
  box-shadow: 0 8px 24px color-mix(in oklab, var(--alpha-base) 12%, transparent); }
.modal-cab { display: flex; align-items: center; gap: 10px; padding: 14px 18px; border-bottom: 1px solid var(--color-border-subtle); }
.modal-pie { display: flex; gap: 10px; justify-content: flex-end; padding: 12px 18px; border-top: 1px solid var(--color-border-subtle); }
```

Para eliminaciones de bajo riesgo (un registro de catálogo simple), un
modal de confirmación estándar (sin type-to-confirm) es suficiente — este
patrón reforzado se reserva para acciones de alto impacto.

## Foco

```css
:focus-visible {
  outline: none;
  box-shadow: 0 0 0 3px color-mix(in oklab, var(--color-ring) 35%, transparent);
  border-radius: var(--radius-sm);
}
```

Universal, no se define por componente — `--color-ring` es el único color
"estructural" (no semántico) de la paleta.
