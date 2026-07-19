# Notifications

## Servicio de toasts

```ts
export type TipoToast = 'exito' | 'error' | 'info';
export interface AccionToast { etiqueta: string; ejecutar: () => void; }
export interface Toast { id: number; tipo: TipoToast; texto: string; accion?: AccionToast; saliendo?: boolean; }

@Injectable({ providedIn: 'root' })
export class NotificationService {
  readonly toasts = signal<Toast[]>([]);

  exito(texto: string, accion?: AccionToast): void { /* auto-cierra a los 3s (6s si trae acción) */ }
  error(texto: string, accion?: AccionToast): void { /* nunca se auto-cierra — solo cierre manual */ }
  info(texto: string, accion?: AccionToast): void  { /* auto-cierra a los 5s (6s si trae acción) */ }

  cerrar(id: number): void { /* marca saliendo=true, espera 180ms de animación, luego remueve */ }
  pausar(id: number): void { /* pausa el contador al hacer hover */ }
  reanudar(id: number): void { /* reanuda el contador al salir el mouse */ }
}
```

Reglas de comportamiento:
- **Máximo 3 toasts visibles** — un 4to empuja fuera al más viejo.
- El nuevo toast se **antepone** (aparece arriba de la pila).
- Duración por severidad: éxito 3s / info 5s / con acción 6s / **error
  nunca se auto-cierra**, solo con el botón de cerrar.
- Hover pausa el contador; al salir el mouse, se reanuda con el tiempo
  restante (no reinicia desde cero).
- Construido 100% sobre `signal<Toast[]>` — sin dependencia de zone.js.

No hay un servicio de confirmación separado dentro de los toasts — un
toast puede llevar una acción inline (ej. "Deshacer"), pero eso es una
acción, no una confirmación bloqueante. Para confirmar una acción
destructiva, usa el modal de `components-buttons-cards-alerts-and-modals.md`.

## Componente visual

```html
<div class="toasts-host">
  @for (t of toasts(); track t.id) {
    <div class="toast toast-{{ t.tipo }}" [class.saliendo]="t.saliendo"
         [attr.role]="t.tipo === 'error' ? 'alert' : 'status'"
         (mouseenter)="notif.pausar(t.id)" (mouseleave)="notif.reanudar(t.id)">
      <i class="bx toast-icono"
         [class.bx-check-circle]="t.tipo === 'exito'"
         [class.bx-error-circle]="t.tipo === 'error'"
         [class.bx-info-circle]="t.tipo === 'info'"></i>
      <span class="toast-texto">{{ t.texto }}</span>
      @if (t.accion; as a) { <button class="toast-accion" (click)="a.ejecutar(); notif.cerrar(t.id)">{{ a.etiqueta }}</button> }
      <button class="toast-x" (click)="notif.cerrar(t.id)" aria-label="Cerrar aviso"><i class="bx bx-x"></i></button>
    </div>
  }
</div>
```

```css
.toasts-host {
  position: fixed; right: 24px; bottom: 24px; z-index: 1000;
  display: flex; flex-direction: column; gap: 10px;
  width: max-content; max-width: min(92vw, 420px); pointer-events: none;
}
@media (max-width: 600px) {
  .toasts-host { right: 16px; left: 16px; bottom: 16px; max-width: none; width: auto; }
}
.toast {
  pointer-events: auto; position: relative;
  display: flex; align-items: center; gap: 10px;
  padding: 12px 12px 12px 16px; border-radius: var(--radius-lg);
  background: var(--gray-75);   /* opaco, nunca transparente */
  border: 1px solid var(--color-border);
  color: var(--color-text);
  box-shadow: 0 6px 20px color-mix(in oklab, var(--alpha-base) 16%, transparent);
  animation: toastEntra .22s ease-out;
  overflow: hidden;
}
.toast::before { content: ''; position: absolute; inset: 0 auto 0 0; width: 3px; }
.toast-exito::before { background: #10b981; }  .toast-exito .toast-icono { color: #10b981; }
.toast-error::before { background: #ef4444; }  .toast-error .toast-icono { color: #ef4444; }
.toast-info::before  { background: #0285ff; }  .toast-info .toast-icono  { color: #0285ff; }
.toast.saliendo { animation: toastSale .18s ease-in forwards; }
@keyframes toastEntra { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: none; } }
@keyframes toastSale { to { opacity: 0; transform: translateY(6px); } }
```

Detalles clave:
- **Posición**: fija abajo-derecha (24px de los bordes); en mobile
  (≤600px) se estira de borde a borde abajo.
- **Fondo opaco** — un fondo semi-transparente hace que el contenido de
  detrás se vea a través y ensucia la lectura; se decidió opaco a
  propósito.
- El color de severidad se limita a una **franja de 3px a la izquierda** +
  el color del ícono — el cuerpo del toast se queda neutral (a diferencia
  de un `.alert`, que sí lleva fondo tintado).
- Respeta `prefers-reduced-motion` (desactiva animaciones).
- `role="alert"` en error, `role="status"` en éxito/info.
