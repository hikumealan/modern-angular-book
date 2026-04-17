# Chapter 35: Charts & Data Visualization

A portfolio overview without a visualization is a spreadsheet. FinancialApp's users need to *see* their money: a donut that splits their portfolio across equities, bonds, and cash; a line chart that tracks balance over ninety days; a bar chart that breaks spending down by category; a candlestick for a ticker they are considering. Charts are not decoration on the dashboard -- they *are* the product. A late-loading, flickering, or theme-mismatched chart is the difference between a trustworthy financial tool and a toy.

Charts touch almost every concern this book has covered. They consume design tokens ([Chapter 27](ch27-material-design-system.md)) so colors adapt to light and dark modes without component-level conditionals. They must be accessible ([Chapter 22](ch22-accessibility-aria.md)) to users who cannot parse pixels. They are among the heaviest rendering workloads in the app, so the performance techniques from [Chapter 24](ch24-performance.md) apply at full force, and the web workers from [Chapter 38](ch38-web-workers.md) become genuinely useful once datasets grow. Treat charts as first-class components, not widgets dropped into a template.

> **Companion code:** `financial-app/libs/shared/charts/` with `FinAllocationChart`, `FinPriceLineChart`, `FinSpendingBarChart`; portfolio-dashboard integrates them.

---

## Choosing a Charting Library

The Angular ecosystem offers a handful of credible charting options. The temptation is to pick one per feature as requirements arrive -- ApexCharts here, D3 there, Highcharts when someone asks for something fancy. Resist it. Every additional library adds bundle weight, a theming surface, a separate API, and a new place bugs can hide. Pick one default and justify every exception.

**ApexCharts with `ng-apexcharts`** is declarative, SVG-based, and ships sensible defaults for exactly the chart types FinancialApp needs: donut, line, area, bar, candlestick, and heatmap. The Angular wrapper exposes inputs that map one-to-one to ApexCharts' options object, so the options object itself is the mental model -- no hidden state inside directives. Theming respects CSS custom properties, and the library handles the annoying parts of responsive SVG charts (aspect ratio, tooltip positioning, legend wrapping).

**`ng2-charts`** wraps Chart.js, which renders to `<canvas>`. For datasets above a few thousand points, Canvas is substantially faster than SVG because the browser never materializes thousands of DOM nodes. The trade-off is that canvas drawings are opaque to the DOM -- CSS-driven theming needs imperative overrides, accessibility has to be bolted on manually, and there is no "inspect element" equivalent. Reach for Canvas when dataset size forces the decision.

**Highcharts** is the most feature-complete commercial option. It handles edge cases (log scales with negative values, synchronized tooltips across panes, complex annotations) that other libraries punt on. The licensing fee is real, but for a trading platform shipping synchronized multi-pane charts, the engineering time saved pays for itself quickly. `highcharts-angular` is the official wrapper.

**D3** is not a chart library at all -- it is a set of primitives for binding data to SVG and computing scales, axes, and interpolations. Unlimited flexibility at the cost of writing every pixel yourself. For a one-off visualization (a sunburst of category hierarchy, a Sankey of cash flows), D3 inside a single component is pragmatic. For standard charts, it is overkill that will haunt your maintainers.

| Library | Renderer | Bundle (gzipped) | License | Good for |
|---|---|---|---|---|
| ApexCharts + `ng-apexcharts` | SVG | ~95 KB | MIT | Default for dashboards; donut, line, bar, candlestick |
| `ng2-charts` (Chart.js) | Canvas | ~60 KB | MIT | Thousands of points; real-time streaming |
| Highcharts + `highcharts-angular` | SVG (Canvas for stocks) | ~180 KB | Commercial | Trading-platform features; synchronized charts |
| D3 | SVG (manual) | ~80 KB (tree-shaken) | ISC | Bespoke visualizations that do not fit standard types |

FinancialApp's decision: **ApexCharts as the default**, Canvas-based `ng2-charts` inside the transaction-heatmap feature (which renders years of daily data), D3 only in the one-off cash-flow Sankey. That gives us three libraries total, each load-split behind `@defer` blocks from [Chapter 17](ch17-defer-ssr-hydration.md) so they never land in the initial bundle.

---

## SVG vs Canvas Performance

The renderer question surfaces again once dataset sizes grow. A rough rule of thumb: **SVG scales linearly with DOM node count**. At a few hundred points per chart, SVG is fine and you gain CSS-driven theming, per-element accessibility, and free hit-testing. At a few thousand points, layout and paint costs dominate -- scrolling a page with three SVG charts of 5,000 points each drops frames on mid-tier hardware.

**Canvas** sidesteps the DOM entirely. The library draws to a bitmap, so 50,000 points cost the same at layout time as ten. The price is that every interaction (tooltip positioning, hover highlight, legend toggling) must be re-implemented against pixel coordinates. Chart.js does this work for you, but you lose per-element ARIA. **WebGL** is the escape hatch for true extremes -- intraday tick data with millions of candles. Libraries like `lightweight-charts` pair naturally with a web worker that streams parsed data across `postMessage`, keeping the main thread free for input ([Chapter 38](ch38-web-workers.md)).

For FinancialApp's normal flows, SVG via ApexCharts is the right default. If a product manager asks for a chart of "every transaction this year," that is the signal to switch renderers, not the signal to file a perf bug against ApexCharts.

---

## Integrating with the Design System

Design tokens from [Chapter 27](ch27-material-design-system.md) live as CSS custom properties on `:root`. Chart libraries expect JavaScript strings when they build their options object. Bridging the two is the most important integration decision: do it wrong and charts show hardcoded blues while the rest of the app is in dark mode.

The bridge lives in `ChartThemeService`. Every chart injects it, reads the theme as a signal, and recomputes its options whenever the theme changes:

```typescript
// libs/shared/charts/src/lib/chart-theme.ts
import { inject, Injectable, signal, effect } from '@angular/core';
import { DOCUMENT } from '@angular/common';
import { ThemeService } from '@fin/design-system';

export interface ChartTheme {
  primary: string;
  secondary: string;
  tertiary: string;
  positive: string;
  negative: string;
  surface: string;
  onSurface: string;
  outline: string;
  fontFamily: string;
  fontFamilyMono: string;
}

@Injectable({ providedIn: 'root' })
export class ChartThemeService {
  private readonly document = inject(DOCUMENT);
  private readonly theme = inject(ThemeService);
  readonly chartTheme = signal<ChartTheme>(this.readFromDom());

  constructor() {
    effect(() => {
      this.theme.theme();
      queueMicrotask(() => this.chartTheme.set(this.readFromDom()));
    });
  }

  private readFromDom(): ChartTheme {
    const styles = getComputedStyle(this.document.documentElement);
    const read = (name: string) => styles.getPropertyValue(name).trim();
    return {
      primary: read('--fin-color-primary'),
      secondary: read('--fin-color-secondary'),
      tertiary: read('--fin-color-tertiary') || read('--fin-color-secondary'),
      positive: read('--fin-color-success') || '#2e7d32',
      negative: read('--fin-color-danger'),
      surface: read('--fin-color-surface'),
      onSurface: read('--fin-color-on-surface'),
      outline: read('--fin-color-outline'),
      fontFamily: read('--fin-font-family-brand') || 'Inter, sans-serif',
      fontFamilyMono: read('--fin-font-family-mono') || 'Roboto Mono, monospace',
    };
  }
}
```

The `effect()` reacts to any change in `ThemeService.theme` and schedules the read on the next microtask, giving the DOM a chance to apply the new `data-theme` attribute before we ask for computed styles. Every chart consumes `chartTheme()` as a signal, so its options are automatically invalidated when the theme flips -- no manual subscriptions, no `MutationObserver`, no timing bugs where charts flash the wrong colors for a frame.

Typography alignment matters too. Mixing proportional and tabular numerals in a tooltip makes digits visually "jump" as values change. `ChartTheme.fontFamilyMono` feeds into ApexCharts' `tooltip.style.fontFamily` and `yaxis.labels.style.fontFamily`, so every axis tick and tooltip uses the same `Roboto Mono` with `tabular-nums` the rest of the app uses.

---

## `FinAllocationChart` -- Portfolio Donut

The portfolio allocation donut is the chart users see most. It splits total value across asset classes (equities, bonds, cash, alternatives) and, on drill-down, across holdings within a class. It needs to be interactive, themed, and accessible.

```typescript
// libs/shared/charts/src/lib/fin-allocation-chart.component.ts
import { Component, ChangeDetectionStrategy, inject, input, output, computed } from '@angular/core';
import { NgApexchartsModule, ApexOptions } from 'ng-apexcharts';
import { CurrencyPipe } from '@angular/common';
import { ChartThemeService } from './chart-theme';

export interface AllocationSlice {
  readonly id: string;
  readonly label: string;
  readonly value: number;
  readonly color?: string;
}

@Component({
  selector: 'fin-allocation-chart',
  imports: [NgApexchartsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: { 'role': 'img', '[attr.aria-label]': 'ariaLabel()' },
  template: `
    <apx-chart
      [series]="series()"
      [chart]="options().chart!"
      [labels]="labels()"
      [colors]="colors()"
      [plotOptions]="options().plotOptions!"
      [legend]="options().legend!"
      [dataLabels]="options().dataLabels!"
      [tooltip]="options().tooltip!"
      [stroke]="options().stroke!"
      [states]="options().states!" />
    <table class="visually-hidden">
      <caption>Portfolio allocation</caption>
      <thead><tr><th>Category</th><th>Value</th><th>Share</th></tr></thead>
      <tbody>
        @for (s of slicesWithPercent(); track s.id) {
          <tr>
            <td>{{ s.label }}</td>
            <td>{{ s.value | currency }}</td>
            <td>{{ s.percent | number:'1.1-1' }}%</td>
          </tr>
        }
      </tbody>
    </table>
  `,
  styles: `
    :host { display: block; }
    .visually-hidden {
      position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
      overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
    }
  `,
})
export class FinAllocationChartComponent {
  private readonly themeSvc = inject(ChartThemeService);
  private readonly currencyPipe = new CurrencyPipe('en-US');

  readonly slices = input.required<readonly AllocationSlice[]>();
  readonly height = input(320);
  readonly sliceSelected = output<AllocationSlice>();

  protected readonly series = computed(() => this.slices().map(s => s.value));
  protected readonly labels = computed(() => this.slices().map(s => s.label));

  protected readonly slicesWithPercent = computed(() => {
    const total = this.slices().reduce((acc, s) => acc + s.value, 0) || 1;
    return this.slices().map(s => ({ ...s, percent: (s.value / total) * 100 }));
  });

  protected readonly colors = computed(() => {
    const t = this.themeSvc.chartTheme();
    const palette = [t.primary, t.secondary, t.tertiary, t.positive, t.negative];
    return this.slices().map((s, i) => s.color ?? palette[i % palette.length]);
  });

  protected readonly ariaLabel = computed(() => {
    const n = this.slices().length;
    const total = this.slices().reduce((acc, s) => acc + s.value, 0);
    return `Portfolio allocation donut chart, ${n} categories, total ${this.currencyPipe.transform(total)}.`;
  });

  protected readonly options = computed<ApexOptions>(() => {
    const t = this.themeSvc.chartTheme();
    return {
      chart: {
        type: 'donut',
        height: this.height(),
        fontFamily: t.fontFamily,
        background: 'transparent',
        events: {
          dataPointSelection: (_e, _ctx, cfg) => {
            const slice = this.slices()[cfg.dataPointIndex];
            if (slice) this.sliceSelected.emit(slice);
          },
        },
      },
      plotOptions: {
        pie: {
          donut: {
            size: '68%',
            labels: {
              show: true,
              total: {
                show: true,
                label: 'Total',
                fontFamily: t.fontFamilyMono,
                color: t.onSurface,
                formatter: () => this.formatTotal(),
              },
            },
          },
        },
      },
      stroke: { colors: [t.surface], width: 2 },
      legend: { position: 'bottom', labels: { colors: t.onSurface } },
      dataLabels: { enabled: false },
      tooltip: {
        fillSeriesColor: false,
        style: { fontFamily: t.fontFamilyMono },
        y: { formatter: (v: number) => this.currencyPipe.transform(v) ?? String(v) },
      },
      states: {
        hover: { filter: { type: 'lighten', value: 0.08 } },
        active: { filter: { type: 'darken', value: 0.08 } },
      },
    };
  });

  private formatTotal(): string {
    const total = this.slices().reduce((acc, s) => acc + s.value, 0);
    return this.currencyPipe.transform(total, 'USD', 'symbol', '1.0-0') ?? '';
  }
}
```

Three patterns deserve attention. First, `options` is a `computed()` that depends on `slices`, `height`, and `chartTheme`. When the theme toggles, the options object is regenerated and `ng-apexcharts` re-renders with the new colors -- no imperative code required. Second, the component publishes a `sliceSelected` output that fires on slice click, driving drill-down at the dashboard level. Third, the hidden `<table>` is the accessibility fallback; we will cover why below.

---

## `FinPriceLineChart` -- Balance History with Crosshair

The balance history chart is a zoomable, pannable line chart with a crosshair tooltip that shows the exact value under the cursor. ApexCharts supports this out of the box through its `zoom`, `xaxis.tooltip`, and `stroke` configuration:

```typescript
// libs/shared/charts/src/lib/fin-price-line-chart.component.ts
@Component({
  selector: 'fin-price-line-chart',
  imports: [NgApexchartsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: { 'role': 'img', '[attr.aria-label]': 'ariaLabel()' },
  template: `
    <apx-chart
      [series]="series()"
      [chart]="options().chart!"
      [xaxis]="options().xaxis!"
      [yaxis]="options().yaxis!"
      [stroke]="options().stroke!"
      [markers]="options().markers!"
      [tooltip]="options().tooltip!"
      [grid]="options().grid!"
      [fill]="options().fill!"
      [colors]="options().colors!" />
  `,
  styles: `:host { display: block; }`,
})
export class FinPriceLineChartComponent {
  private readonly themeSvc = inject(ChartThemeService);
  private readonly currencyPipe = new CurrencyPipe('en-US');
  private readonly datePipe = new DatePipe('en-US');

  readonly points = input.required<readonly { t: number; v: number }[]>();
  readonly seriesName = input('Balance');
  readonly height = input(280);
  readonly zoomable = input(true);

  protected readonly series = computed(() => [{
    name: this.seriesName(),
    data: this.points().map(p => [p.t, p.v] as [number, number]),
  }]);

  protected readonly ariaLabel = computed(() => {
    const pts = this.points();
    if (!pts.length) return `${this.seriesName()} line chart, no data`;
    const first = pts[0], last = pts[pts.length - 1];
    const change = last.v - first.v;
    const direction = change >= 0 ? 'up' : 'down';
    return `${this.seriesName()} from ${this.datePipe.transform(first.t, 'mediumDate')} to ` +
           `${this.datePipe.transform(last.t, 'mediumDate')}, ${direction} ` +
           `${this.currencyPipe.transform(Math.abs(change))} over ${pts.length} points.`;
  });

  protected readonly options = computed<ApexOptions>(() => {
    const t = this.themeSvc.chartTheme();
    const pts = this.points();
    const lastVal = pts.at(-1)?.v ?? 0;
    const firstVal = pts[0]?.v ?? 0;
    const up = lastVal >= firstVal;
    return {
      chart: {
        type: 'area', height: this.height(), background: 'transparent',
        fontFamily: t.fontFamily,
        zoom: { enabled: this.zoomable(), type: 'x', autoScaleYaxis: true },
        toolbar: { show: this.zoomable(), tools: { download: false } },
        animations: { enabled: false },
      },
      colors: [up ? t.positive : t.negative],
      stroke: { curve: 'smooth', width: 2 },
      fill: {
        type: 'gradient',
        gradient: { shadeIntensity: 1, opacityFrom: 0.35, opacityTo: 0.02, stops: [0, 100] },
      },
      markers: { size: 0, hover: { size: 5 } },
      grid: { borderColor: t.outline, strokeDashArray: 3 },
      xaxis: {
        type: 'datetime',
        labels: { style: { colors: t.onSurface, fontFamily: t.fontFamilyMono } },
        axisBorder: { color: t.outline },
        crosshairs: { show: true, stroke: { color: t.outline, dashArray: 2 } },
      },
      yaxis: {
        labels: {
          style: { colors: t.onSurface, fontFamily: t.fontFamilyMono },
          formatter: (v: number) => this.currencyPipe.transform(v, 'USD', 'symbol', '1.0-0') ?? '',
        },
      },
      tooltip: {
        shared: true, intersect: false,
        x: { format: 'MMM d, yyyy' },
        y: { formatter: (v: number) => this.currencyPipe.transform(v) ?? '' },
        style: { fontFamily: t.fontFamilyMono },
      },
    };
  });
}
```

Two subtle choices. We disable `animations` because this chart routinely updates in response to a date-range selector, and animating every transition feels laggy. We set `markers.size` to zero by default -- visible dots on every data point clutter the line -- but size five on hover, so the crosshair tooltip lands on a discrete dot. The stroke flips between `theme.positive` and `theme.negative` based on whether the period ended up or down -- a domain-specific flourish that would feel wrong in a generic library but fits instantly once the chart is wrapped in an app-specific component.

---

## `FinSpendingBarChart` -- Category Breakdown

The spending-by-category chart is a responsive bar chart. A `stacked` input flips it between "one bar per month, stacked by category" and "one bar per category, summing across months" -- the same data visualized two ways, selected by a segmented control in the parent component.

```typescript
// libs/shared/charts/src/lib/fin-spending-bar-chart.component.ts
@Component({
  selector: 'fin-spending-bar-chart',
  imports: [NgApexchartsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: { 'role': 'img', '[attr.aria-label]': 'ariaLabel()' },
  template: `
    <apx-chart
      [series]="series()"
      [chart]="options().chart!"
      [xaxis]="options().xaxis!"
      [yaxis]="options().yaxis!"
      [plotOptions]="options().plotOptions!"
      [dataLabels]="options().dataLabels!"
      [legend]="options().legend!"
      [colors]="options().colors!"
      [tooltip]="options().tooltip!"
      [grid]="options().grid!" />
  `,
})
export class FinSpendingBarChartComponent {
  private readonly themeSvc = inject(ChartThemeService);
  private readonly currencyPipe = new CurrencyPipe('en-US');

  readonly buckets = input.required<readonly string[]>();
  readonly categories = input.required<readonly { name: string; values: readonly number[] }[]>();
  readonly stacked = input(false);
  readonly height = input(320);

  protected readonly series = computed(() =>
    this.categories().map(c => ({ name: c.name, data: [...c.values] })));

  protected readonly ariaLabel = computed(() =>
    `Spending by category, ${this.categories().length} categories across ` +
    `${this.buckets().length} periods${this.stacked() ? ', stacked' : ''}.`);

  protected readonly options = computed<ApexOptions>(() => {
    const t = this.themeSvc.chartTheme();
    return {
      chart: {
        type: 'bar', height: this.height(),
        stacked: this.stacked(), background: 'transparent',
        fontFamily: t.fontFamily, toolbar: { show: false },
      },
      colors: [t.primary, t.secondary, t.tertiary, t.positive, t.negative],
      plotOptions: {
        bar: {
          horizontal: false, columnWidth: '55%',
          borderRadius: 4, borderRadiusApplication: 'end',
        },
      },
      dataLabels: { enabled: false },
      xaxis: {
        categories: [...this.buckets()],
        labels: { style: { colors: t.onSurface, fontFamily: t.fontFamilyMono } },
      },
      yaxis: {
        labels: {
          style: { colors: t.onSurface, fontFamily: t.fontFamilyMono },
          formatter: (v: number) => this.currencyPipe.transform(v, 'USD', 'symbol', '1.0-0') ?? '',
        },
      },
      grid: { borderColor: t.outline, strokeDashArray: 3 },
      legend: { position: 'bottom', labels: { colors: t.onSurface } },
      tooltip: {
        y: { formatter: (v: number) => this.currencyPipe.transform(v) ?? '' },
        style: { fontFamily: t.fontFamilyMono },
      },
    };
  });
}
```

Responsiveness comes for free from SVG rendering -- the chart fills its container and ApexCharts handles label overflow by rotating x-axis ticks when space runs out. `columnWidth: '55%'` leaves visible gutters between bars, which reads better than flush-packed bars for categorical data.

---

## Accessibility

A chart is a picture of data. Users who cannot see the picture still need the data. WCAG 2.2 Level AA (the baseline from [Chapter 22](ch22-accessibility-aria.md)) has three concrete requirements for charts:

**A meaningful accessible name.** The host element carries `role="img"` and an `aria-label` computed from the data -- not the chart type, but its *content*. "Portfolio allocation donut chart, 4 categories, total $194,150" is useful; "Donut chart" is not. Each chart above derives its label through a `computed()` signal so it updates as data changes.

**A text alternative for the actual numbers.** The `aria-label` summarizes; a screen-reader user still needs to *read the values*. `FinAllocationChart` includes a `.visually-hidden` table with the same data the donut shows. Screen readers read it as ordinary tabular data -- `role="img"` on the host plus the hidden table gives both a visual summary and programmatic access to each row. The `.visually-hidden` CSS is the standard "screen-reader only" pattern: a zero-size clipped element that stays in the accessibility tree.

**Keyboard interactivity.** If clicking a slice triggers a drill-down, keyboard users must be able to trigger the same action. ApexCharts' default keyboard support is thin, so `FinAllocationChart` renders a visible focusable legend next to the chart (present in the companion code). Keyboard users Tab into the legend, press Enter or Space, and the component emits the same `sliceSelected` output as a click.

**No color-only encoding.** Color-blind users cannot rely on red-vs-green. `FinPriceLineChart` pairs its positive/negative colorway with a trend arrow (`▲` or `▼`) in the tooltip, and the allocation chart's hidden table makes quantitative values available without depending on slice color. The 4.5:1 text-contrast requirement applies to axis labels -- `--fin-color-on-surface` is already compliant against `--fin-color-surface` in both themes, which is exactly why we pull colors from the design system rather than ApexCharts' defaults. The Playwright `axe-core` smoke test from [Chapter 22](ch22-accessibility-aria.md) runs on every chart-bearing page and fails the build on new violations.

---

## Performance

Charts are among the most expensive widgets an Angular app renders. They dominate paint time on dashboard pages, their libraries are heavy (60-180 KB even tree-shaken), and naive integration breaks [Chapter 24](ch24-performance.md)'s change-detection discipline. Five techniques do most of the work.

**Lazy-load the chart library.** Chart libraries should never ship in the initial bundle. Wrap every chart in a `@defer` block keyed to `on viewport`:

```html
@defer (on viewport; prefetch on idle) {
  <fin-allocation-chart [slices]="slices()" />
} @placeholder (minimum 300ms) {
  <div class="chart-skeleton" style="height: 320px"></div>
}
```

The `prefetch on idle` trigger starts downloading the ApexCharts chunk as soon as the main content is idle, so by the time the user scrolls the chart into view, the library is cached. Users who never scroll down never download it.

**Memoize the options object with a stable `computed()`.** If you build options inline in a template expression, it re-evaluates on every change-detection tick and the chart re-renders. Routing through `computed()` means Angular only recomputes when actual inputs change, and the chart sees an identical object reference across unrelated CD passes -- an order-of-magnitude reduction in re-render work on a busy dashboard.

**Avoid deep input churn.** `ng-apexcharts` compares inputs by reference. If `points()` produces a new array every CD cycle (because of a `map` in the template), the chart treats each cycle as a data change and redraws. Derive arrays through `computed()` so identical inputs produce identical references -- the `series` `computed()` in `FinPriceLineChartComponent` only produces a new array when `points()` actually changes.

**Offload heavy computation to a web worker.** Rolling up a year of daily transactions into monthly category totals -- parsing dates, grouping, summing, sorting -- blocks input for 50-100 ms on mid-tier hardware. Move it to a worker ([Chapter 38](ch38-web-workers.md)) and the dashboard stays responsive:

```typescript
// spending.worker.ts
addEventListener('message', (e: MessageEvent<Transaction[]>) => {
  const buckets = rollupByMonthAndCategory(e.data);
  postMessage(buckets);
});

// spending-dashboard.component.ts
private readonly worker = new Worker(new URL('./spending.worker', import.meta.url), { type: 'module' });
private readonly spendingBuckets = signal<SpendingBuckets | null>(null);
constructor() {
  this.worker.onmessage = (e) => this.spendingBuckets.set(e.data);
  effect(() => this.worker.postMessage(this.rawTransactions()));
}
```

The component hands raw transactions to the worker whenever they change, the worker computes buckets off-thread, and `postMessage` delivers the result into a signal that drives `FinSpendingBarChart`. The main thread paints; it never computes.

**Virtualize dense datasets.** For a chart with 50,000 points, set `chart.animations.enabled = false` (animation cost is proportional to point count) and downsample with the largest-triangle-three-buckets (LTTB) algorithm inside the same worker. A 50,000-point series downsamples to 500 visually indistinguishable points in under 2 ms, and the chart renders instantly.

---

## Testing Charts

Pixel-perfect testing of rendered charts is a losing battle -- fonts, antialiasing, and ApexCharts version bumps all cause false diffs. Test charts at three layers instead.

**Unit-test the options builder as pure data.** Every chart exposes its `options` `computed()`. Render with a known input and snapshot the options object -- changes are reviewable in a PR diff, and unrelated DOM or rendering changes do not affect the test:

```typescript
// fin-allocation-chart.component.spec.ts
it('maps slices to series values', () => {
  const fixture = TestBed.createComponent(FinAllocationChartComponent);
  fixture.componentRef.setInput('slices', [
    { id: 'a', label: 'Equity', value: 100 },
    { id: 'b', label: 'Bond', value: 50 },
  ]);
  fixture.detectChanges();
  const component = fixture.componentInstance as any;
  expect(component.series()).toEqual([100, 50]);
  expect(component.labels()).toEqual(['Equity', 'Bond']);
  expect(component.options().chart!.type).toBe('donut');
});
```

**Snapshot the accessibility fallback.** The hidden data table is pure, deterministic HTML. Snapshot-test it to catch regressions in the accessible name or data-table contents without involving SVG or canvas:

```typescript
it('renders a hidden data table for screen readers', () => {
  // ...render component with known slices...
  const table = fixture.nativeElement.querySelector('table');
  expect(table).toMatchSnapshot();
});
```

**Visual regression via Playwright screenshots.** For the few tests where "does it actually look right" matters, use Playwright's `toHaveScreenshot()` with a pixel tolerance (`maxDiffPixelRatio: 0.02`) so small antialiasing differences do not flake, and pin the browser version in CI for consistent font rendering. Run visual tests in both light and dark themes -- that is where `ThemeService` integration regressions surface.

---

## Summary

Charts are a first-class concern in FinancialApp, not a late-stage decoration. Treating them seriously means treating the full system -- library choice, design-system integration, accessibility, performance, and tests -- with the same rigor as any other architectural decision.

- **Pick one default library.** ApexCharts + `ng-apexcharts` covers the standard dashboard vocabulary (donut, line, bar, candlestick) with declarative SVG and sensible defaults. Exceptions -- Canvas for large datasets, commercial for trading-platform complexity, D3 for bespoke visuals -- are justified per feature, not accumulated by accident.
- **SVG scales to hundreds, Canvas to thousands, WebGL to millions.** Let dataset size dictate the renderer, not framework fashion.
- **Bridge design tokens through `ChartThemeService`.** A single service reads CSS custom properties, exposes them as a signal, and feeds every chart's `computed()` options. Theme toggling propagates through the cascade *and* through chart re-renders without any per-chart plumbing.
- **Wrap charts in domain components.** `FinAllocationChart`, `FinPriceLineChart`, and `FinSpendingBarChart` expose inputs that speak the portfolio domain (slices, points, buckets), not chart-library vocabulary. Feature developers never touch ApexCharts APIs directly.
- **Accessibility is built in, not bolted on.** Each chart carries `role="img"` with a computed `aria-label`, a visually-hidden data table with the raw numbers, and a parallel keyboard affordance for any interactive behavior. Color never carries meaning alone.
- **Lazy-load the chart library.** `@defer (on viewport; prefetch on idle)` keeps the chart code out of the initial bundle while preserving the feel of "charts are there when you scroll to them."
- **Push heavy rollups to a worker.** Large aggregations belong on a background thread, delivered back into a signal that drives the chart -- the pattern in [Chapter 38](ch38-web-workers.md).
- **Test the options builder, the a11y fallback, and the rendered page independently.** Pure-data unit tests catch logic regressions, snapshot tests catch fallback regressions, Playwright screenshots catch theme and layout regressions.

The charts in `libs/shared/charts/` are a template more than a library. New chart types -- a heatmap of daily transaction counts, a candlestick for ticker pages, a Sankey of cash flows between accounts -- follow the same shape: a `computed()` options object fed by a domain-shaped input, a theme signal from `ChartThemeService`, a hidden data fallback, and a `@defer` wrapper at the callsite. Once the first three charts are in place, the next ten are mostly configuration.
