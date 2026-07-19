# KPI Cards and Charts

## KPI card con conteo animado

**No es un paquete npm** — es una directiva hecha a mano, sin dependencia
externa. Vive en `shared/directives/count-up.directive.ts` (ver skill
`angular-feature-architecture`, `references/project-structure.md`).

```ts
@Directive({ selector: '[appCountUp]', standalone: true })
export class CountUpDirective implements OnInit, OnDestroy {
  @Input('appCountUp') targetValue = 0;
  @Input() countDuration = 2000;

  private _intervalId?: ReturnType<typeof setInterval>;
  private readonly _el = inject(ElementRef<HTMLElement>);
  private _observer?: IntersectionObserver;

  ngOnInit(): void {
    this._observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) { this._animate(); this._observer?.disconnect(); }
    });
    this._observer.observe(this._el.nativeElement);
  }

  private _animate(): void {
    const end = this.targetValue;
    const fps = 60;
    const step = this.countDuration / fps;
    const increment = Math.max(1, Math.floor(end / (this.countDuration / step)));
    let current = 0;
    this._intervalId = setInterval(() => {
      current += increment;
      if (current >= end) { current = end; clearInterval(this._intervalId); }
      this._el.nativeElement.textContent = String(current);
    }, step);
  }

  ngOnDestroy(): void { if (this._intervalId) clearInterval(this._intervalId); this._observer?.disconnect(); }
}
```

Se dispara solo cuando la tarjeta entra al viewport (`IntersectionObserver`),
no al cargar la página — evita animar KPIs que el usuario nunca llega a
ver. Uso:

```html
<span class="kpi-valor" [appCountUp]="totalNumerico">{{ totalNumerico }}</span>
```

La tarjeta KPI en sí (`shared/ui/kpi-card`) es un componente reutilizable
con `input()` para título/valor/color/ícono — nunca duplicado por vista;
cualquier lista con datos relevantes (ventas, vacaciones, catálogos, etc.)
agrupa sus KPIs con este mismo componente arriba de la tabla.

## Charts — Chart.js + ng2-charts

Paquetes: `chart.js` + `ng2-charts` (`BaseChartDirective`). Todo chart va
envuelto en el mismo idioma de card que el resto del sistema (`rounded-xl
border border-[var(--color-border)] p-6`), con el canvas confinado a un
alto fijo.

**Regla dura**: los colores de cada serie son tokens de acento con nombre
(ver `tokens-typography-and-dark-mode.md`) — nunca un hex hardcodeado
dentro de la config del chart.

### Sparkline (KPI individual)

```ts
getSparklineConfig(serie: number[], color: string): ChartConfiguration<'line'> {
  return {
    type: 'line',
    data: { labels: this.labels, datasets: [{ data: serie, borderColor: color, borderWidth: 2, fill: false, pointRadius: 0, tension: 0.4 }] },
    options: {
      responsive: true, maintainAspectRatio: false,
      plugins: { legend: { display: false }, tooltip: { enabled: false } },
      scales: {
        x: { display: true, grid: { display: false }, ticks: { font: { size: 10, family: 'var(--fuente-ui)' }, maxTicksLimit: 4 }, border: { display: false } },
        y: { display: false },
      },
      animation: { duration: 800, easing: 'easeOutQuart' },
    },
  };
}
```

### Área (tendencia, dashboard)

- Eje X sin grid; eje Y con grid muy sutil (mismo valor que `--color-border`).
- Leyenda con swatches circulares (`pointStyle: 'circle'`), no los
  rectángulos por defecto de Chart.js.
- Tooltip con `cornerRadius: 8`, `padding: 10`, fondo casi negro en modo
  claro / gris oscuro en modo oscuro (leídos de `ThemeService`, ya que
  Chart.js no lee custom properties CSS directamente — única excepción
  real a "nunca ramificar por tema en TS").
- Fuente de ticks/leyenda/tooltip: la fuente de UI del sistema, tamaños
  10-12px.

### Doughnut con texto central (ej. distribución %)

```ts
readonly chartPlugins = [{
  id: 'centerText',
  afterDraw(chart: any): void {
    const { ctx, chartArea: { top, left, width, height } } = chart;
    const total = (chart.data.datasets[0]?.data as number[])?.reduce((a: number, b: number) => a + b, 0) ?? 0;
    ctx.save();
    ctx.font = '600 18px var(--fuente-mono)';
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--color-text-emphasis');
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillText(`${total}%`, left + width / 2, top + height / 2);
    ctx.restore();
  },
}];
```

- `cutout: '70%'` para un anillo delgado.
- Sin leyenda de Chart.js (`legend: { display: false }`) — se renderiza una
  leyenda propia en HTML al costado (punto de color + etiqueta), reusando
  el mismo array de colores de acento que las porciones del chart.
- Tamaño fijo (no responsive) cuando el chart vive dentro de una fila
  expandible o un espacio angosto — `responsive: false` con `width`/`height`
  explícitos.
