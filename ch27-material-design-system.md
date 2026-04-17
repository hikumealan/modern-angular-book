# Chapter 27: Angular Material & Design System

A financial application that looks inconsistent across its pages erodes user trust. If the button in the accounts view has a different border radius than the button in the portfolio view, users notice -- maybe not consciously, but the overall impression shifts from "polished" to "cobbled together." Hardcoded SCSS values scattered across component files create visual drift that compounds with every feature added.

A design system solves this by establishing a single source of truth for visual decisions. Instead of `border-radius: 8px` repeated in forty files, a design token called `--fin-radius-md` captures the decision once. When the design team decides medium radius should be 12px, the change propagates everywhere from a single file.

This chapter builds the FinancialApp design system: tokens, typography, iconography, theming, Angular Material integration, domain-aware wrapper components, and animations. The headless Angular Aria components from [Chapter 22](ch22-accessibility-aria.md) are wrapped with design tokens and domain logic here. The error display patterns from [Chapter 23](ch23-error-handling.md) are consumed by `FinFormField`.

> **Prerequisites:** [Chapter 10](ch10-signal-queries.md) (signal queries), [Chapter 15](ch15-internationalization.md) (locale-aware formatting), [Chapter 22](ch22-accessibility-aria.md) (Angular Aria, a11y defaults), [Chapter 23](ch23-error-handling.md) (error display patterns).

> **Companion code:** `financial-app/libs/shared/design-system/` contains tokens, themes, `ThemeService`, `FinIcon`, and reusable animations. Wrapper components (`FinButton`, `FinDataTable`, `FinFormField`, `FinAccountSelector`, `FinCurrencyInput`) live in `libs/shared/ui/`. Unit tests follow [Chapter 7](ch07-testing-vitest.md) patterns.

---

## Design Tokens Foundation

Design tokens are named values that represent the atomic visual decisions of your application. A token is not a CSS class -- it is a single value like a color, a spacing unit, or a font size. Tokens compose upward into components but are defined independently so that themes can override them without touching component code.

FinancialApp organizes tokens into six categories: color, typography, spacing, elevation, border-radius, and motion.

```scss
// libs/shared/design-system/src/styles/_tokens.scss
:root {
  --fin-color-primary: #1565c0;
  --fin-color-primary-contrast: #ffffff;
  --fin-color-secondary: #00897b;
  --fin-color-danger: #c62828;
  --fin-color-warning: #ef6c00;
  --fin-color-surface: #ffffff;
  --fin-color-surface-variant: #f5f5f5;
  --fin-color-on-surface: #1c1b1f;
  --fin-color-on-surface-variant: #49454f;
  --fin-color-outline: #79747e;
  --fin-color-outline-variant: #cac4d0;

  --fin-space-xs: 4px;
  --fin-space-sm: 8px;
  --fin-space-md: 16px;
  --fin-space-lg: 24px;
  --fin-space-xl: 32px;
  --fin-space-2xl: 48px;

  --fin-elevation-1: 0 1px 3px rgba(0, 0, 0, 0.12), 0 1px 2px rgba(0, 0, 0, 0.08);
  --fin-elevation-2: 0 3px 6px rgba(0, 0, 0, 0.12), 0 2px 4px rgba(0, 0, 0, 0.08);

  --fin-radius-sm: 4px;
  --fin-radius-md: 8px;
  --fin-radius-lg: 16px;
  --fin-radius-full: 9999px;

  --fin-motion-duration-fast: 150ms;
  --fin-motion-duration-normal: 250ms;
  --fin-motion-duration-slow: 400ms;
  --fin-motion-easing-standard: cubic-bezier(0.2, 0, 0, 1);
  --fin-motion-easing-decelerate: cubic-bezier(0, 0, 0, 1);
  --fin-motion-easing-accelerate: cubic-bezier(0.3, 0, 1, 1);
}
```

The spacing scale uses a 4px base unit. This is not arbitrary -- 4px grids align cleanly at common display densities, and the progression (4, 8, 16, 24, 32, 48) provides enough granularity without creating decision fatigue.

### Typography Scale

The type ramp defines six levels -- display, headline, title, body, label, caption -- each with a prescribed size, line-height, weight, and letter-spacing:

```scss
// libs/shared/design-system/src/styles/_typography.scss
@mixin fin-type-display {
  font-family: var(--fin-font-family-brand, 'Inter', sans-serif);
  font-size: 2.25rem; line-height: 2.75rem; font-weight: 400;
  letter-spacing: -0.015em;
}
@mixin fin-type-headline {
  font-family: var(--fin-font-family-brand, 'Inter', sans-serif);
  font-size: 1.5rem; line-height: 2rem; font-weight: 500;
}
@mixin fin-type-title {
  font-family: var(--fin-font-family-brand, 'Inter', sans-serif);
  font-size: 1.125rem; line-height: 1.625rem; font-weight: 500;
}
@mixin fin-type-body {
  font-family: var(--fin-font-family-plain, 'Inter', sans-serif);
  font-size: 0.9375rem; line-height: 1.375rem; font-weight: 400;
}
@mixin fin-type-label {
  font-family: var(--fin-font-family-plain, 'Inter', sans-serif);
  font-size: 0.8125rem; line-height: 1.125rem; font-weight: 500;
  letter-spacing: 0.031em;
}
@mixin fin-type-caption {
  font-family: var(--fin-font-family-plain, 'Inter', sans-serif);
  font-size: 0.75rem; line-height: 1rem; font-weight: 400;
}
@mixin fin-type-number {
  font-family: var(--fin-font-family-mono, 'Roboto Mono', monospace);
  font-variant-numeric: tabular-nums;
}
```

The `fin-type-number` mixin encodes `font-variant-numeric: tabular-nums` so that columns of financial figures align vertically. Mixing proportional and tabular numerals in the same table is a subtle but real readability problem -- this mixin prevents it application-wide.

### Iconography

Icons use Material Symbols, Google's variable-font icon set. The `FinIcon` component wraps the raw font with consistent sizing and correct accessibility attributes:

```typescript
// libs/shared/design-system/src/lib/icon/fin-icon.component.ts
@Component({
  selector: 'fin-icon',
  host: {
    'class': 'fin-icon',
    '[attr.aria-hidden]': '!ariaLabel()',
    '[attr.aria-label]': 'ariaLabel() || null',
    '[attr.role]': 'ariaLabel() ? "img" : null',
    '[style.font-size.px]': 'size()',
  },
  template: `{{ name() }}`,
  styles: `
    :host {
      font-family: 'Material Symbols Outlined';
      display: inline-flex; align-items: center; justify-content: center;
      font-variation-settings: 'FILL' 0, 'wght' 400, 'GRAD' 0, 'opsz' 24;
      color: currentColor; user-select: none;
    }
  `,
})
export class FinIconComponent {
  readonly name = input.required<string>();
  readonly size = input(24);
  readonly ariaLabel = input<string | undefined>(undefined);
}
```

When `ariaLabel` is undefined, `aria-hidden="true"` makes the icon decorative. When provided, `role="img"` and `aria-label` make it informative. This follows [Chapter 22](ch22-accessibility-aria.md)'s principle that the callsite decides an icon's semantic role.

### Light and Dark Modes

FinancialApp ships two themes as CSS custom property overrides scoped to a data attribute:

```scss
// libs/shared/design-system/src/styles/_light.scss
[data-theme='light'] {
  --fin-color-primary: #1565c0;
  --fin-color-surface: #ffffff;
  --fin-color-on-surface: #1c1b1f;
  --fin-color-on-surface-variant: #49454f;
  --fin-color-outline: #79747e;
}

// libs/shared/design-system/src/styles/_dark.scss
[data-theme='dark'] {
  --fin-color-primary: #90caf9;
  --fin-color-surface: #1c1b1f;
  --fin-color-on-surface: #e6e1e5;
  --fin-color-on-surface-variant: #cac4d0;
  --fin-color-outline: #938f99;
}
```

Every component referencing `--fin-color-surface` adapts automatically when the theme changes. No component-level conditionals -- the cascade does the work.

### ThemeService

The `ThemeService` manages the current theme as a signal, respects `prefers-color-scheme` on initial load, and persists the user's override to `localStorage`:

```typescript
// libs/shared/design-system/src/lib/theme/theme.service.ts
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private readonly document = inject(DOCUMENT);
  readonly theme = signal<Theme>(this.initialTheme());

  constructor() {
    effect(() => {
      const current = this.theme();
      this.document.documentElement.setAttribute('data-theme', current);
      localStorage.setItem('fin-theme', current);
    });
  }

  toggleTheme(): void {
    this.theme.update(t => t === 'light' ? 'dark' : 'light');
  }

  private initialTheme(): Theme {
    const stored = localStorage.getItem('fin-theme') as Theme | null;
    if (stored === 'light' || stored === 'dark') return stored;
    return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
  }
}
```

A stored preference wins; otherwise the OS-level media query decides. The `effect()` updates both the DOM attribute and localStorage whenever `theme` changes.

---

## Angular Material v21 Setup

Angular Material provides accessible components following Material Design 3. For FinancialApp, Material serves as the component foundation that the design system wraps with domain-specific behavior.

```bash
ng add @angular/material
```

The schematic installs `@angular/material` and `@angular/cdk`.

### Generating a Material Theme

Material Design 3 palettes -- primary, secondary, tertiary, neutral, neutral-variant, and error -- each contain 13 or more tones spanning light to dark. Hand-authoring those tones is tedious and error-prone; small deviations from the Material specification cause subtle contrast failures that only surface during accessibility audits. Angular Material ships two schematics that derive mathematically correct M3 palettes from seed colors, and every new design system should start with one of them.

**`ng generate @angular/material:m3-theme`** is the primary schematic. It reads seed colors, applies Material's tonal palette algorithm, and writes an SCSS file with complete light and dark theme objects. Options:

| Option | Description |
|---|---|
| `--primaryColor` | Hex code for the brand primary; the only required seed |
| `--secondaryColor` | Optional; derived from `primaryColor` when omitted |
| `--tertiaryColor` | Optional accent seed |
| `--neutralColor` | Optional; controls surface and background tones |
| `--neutralVariantColor` | Optional; controls subtle backgrounds and dividers |
| `--errorColor` | Optional; defaults to Material's error red |
| `--directory` | Output directory for the generated SCSS file |
| `--themeFileName` | Filename (without extension) of the generated file |
| `--includeHighContrast` | Generates a high-contrast variant for accessibility compliance |

For FinancialApp, the seeds map directly to the brand palette. Commit the command to an npm script so that regenerating the theme is reproducible across the team:

```json
// financial-app/package.json (excerpt)
{
  "scripts": {
    "generate:m3-theme": "ng generate @angular/material:m3-theme --primaryColor=\"#1a73e8\" --secondaryColor=\"#34a853\" --tertiaryColor=\"#fbbc04\" --errorColor=\"#d93025\" --directory=\"libs/shared/design-system/src/styles\" --themeFileName=\"_m3-theme\" --includeHighContrast=true"
  }
}
```

Running `npm run generate:m3-theme` produces `libs/shared/design-system/src/styles/_m3-theme.scss`. The file has a predictable structure:

```scss
// libs/shared/design-system/src/styles/_m3-theme.scss
@use '@angular/material' as mat;

$_palettes: (
  primary: (
    0: #000000, 10: #001b3d, 20: #002e69, 30: #004494, 40: #1a73e8,
    50: #4c90ff, 60: #73abff, 70: #99c6ff, 80: #bcd7ff, 90: #dae2ff,
    95: #edf0ff, 99: #fefbff, 100: #ffffff,
  ),
  secondary: ( /* tonal palette derived from #34a853 */ ),
  tertiary:  ( /* tonal palette derived from #fbbc04 */ ),
  neutral:   ( /* surface and background tones */ ),
  neutral-variant: ( /* subtle backgrounds, dividers */ ),
  error:     ( /* tonal palette derived from #d93025 */ ),
);

$light-theme: mat.define-theme((
  color: ( theme-type: light, primary: $_palettes ),
));

$dark-theme: mat.define-theme((
  color: ( theme-type: dark, primary: $_palettes ),
));
```

The tonal maps at the top are machine-generated. The two `mat.define-theme(...)` calls at the bottom are what the rest of the design system consumes. Teams typically leave the palette maps alone (regenerate them when brand colors change) and hand-edit only the `mat.define-theme` call when they need to customize typography, density, or add extra config layers.

**`ng generate @angular/material:theme-color`** is the narrower alternative. It generates a single tonal palette rather than a full theme. Use it when you need to add one additional semantic color -- for example, a "success" green for positive transaction amounts that is distinct from the tertiary palette -- to an existing theme:

```bash
ng generate @angular/material:theme-color --color="#2e7d32" --isDark=false
```

The output is a tonal palette map you paste into `_m3-theme.scss` or a sibling file.

### Integrating Generated Palettes with FinancialApp Tokens

The generator produces Material's theme object. The `_tokens.scss` file from earlier in this chapter provides app-level tokens (spacing, typography scale, iconography) that live *alongside* Material. Wire them together so `ThemeService` drives both:

```scss
// libs/shared/design-system/src/styles/_material-theme.scss
@use '@angular/material' as mat;
@use './m3-theme' as theme;

html {
  @include mat.all-component-themes(theme.$light-theme);
  @include mat.theme((typography: Inter, density: 0));
}

[data-theme='dark'] {
  @include mat.all-component-themes(theme.$dark-theme);
}
```

The `ThemeService` from earlier in this chapter still toggles `data-theme` on the root element; now the toggle swaps Material's theme *and* the app tokens simultaneously. Wrapper components like `FinButton` and `FinFormField` can pull from both Material's `--mat-sys-*` CSS custom properties (which `mat.all-component-themes` emits) and the `--fin-*` tokens defined in `_tokens.scss`.

When to regenerate vs hand-edit: regenerate whenever brand colors change, because hand-editing tones drifts from the M3 specification. Hand-edit the `mat.define-theme` call itself when adjusting typography, density, or adding config layers that are not color-related.

The custom theme then maps FinancialApp's tokens to Material's theming system:

```scss
// libs/shared/design-system/src/styles/_material-theme.scss
@use '@angular/material' as mat;

html {
  @include mat.theme((
    color: (primary: mat.$blue-palette, tertiary: mat.$green-palette, theme-type: light),
    typography: Inter,
    density: 0,
  ));
}

[data-theme='dark'] {
  @include mat.theme((
    color: (primary: mat.$blue-palette, tertiary: mat.$green-palette, theme-type: dark),
  ));
}
```

FinancialApp uses a focused subset imported per-feature: MatToolbar, MatSidenav, MatTable, MatCard, MatFormField, MatButton, MatDialog, MatSnackBar, MatTabs, and MatDatepicker.

Material's `MatButton` uses an attribute selector on the native `<button>` rather than a custom element. The native button retains its inherent accessibility semantics -- keyboard focusability, form submission, screen reader announcement. Material layers on styling and ripple animation without replacing the element's role. This is the same augmentation philosophy behind Angular's attribute directives from [Chapter 11](ch11-directives-templates.md).

---

## Domain-Aware Wrapper Components

Angular Material gives you a component library. A design system gives you a component *vocabulary* -- components named after your domain, styled by your tokens, and constrained to the patterns your application actually needs. Atomic Design offers a useful mental model for composition (atoms, molecules, organisms), but the file system stays organized by feature and shared library.

### FinButton

`FinButton` wraps Material's button variants with a constrained set of visual variants, consistent icon placement, and a loading state:

```typescript
// libs/shared/ui/src/lib/button/fin-button.component.ts
@Component({
  selector: 'fin-button',
  imports: [MatButtonModule, MatProgressSpinnerModule, FinIconComponent],
  template: `
    <button [attr.mat-raised-button]="variant() === 'primary' ? '' : null"
            [attr.mat-stroked-button]="variant() === 'ghost' ? '' : null"
            [attr.mat-flat-button]="variant() === 'danger' ? '' : null"
            [color]="buttonColor()"
            [disabled]="disabled() || loading()"
            [attr.aria-busy]="loading()">
      @if (loading()) { <mat-spinner diameter="18" /> }
      @if (icon(); as iconName) { <fin-icon [name]="iconName" [size]="18" /> }
      <span class="fin-button__label"><ng-content /></span>
    </button>
  `,
  styles: `
    :host { display: inline-block; }
    button { gap: var(--fin-space-sm); border-radius: var(--fin-radius-md); }
  `,
})
export class FinButtonComponent {
  readonly variant = input<'primary' | 'danger' | 'ghost'>('primary');
  readonly icon = input<string | undefined>(undefined);
  readonly disabled = input(false);
  readonly loading = input(false);

  protected readonly buttonColor = computed(() =>
    this.variant() === 'danger' ? 'warn' : this.variant() === 'ghost' ? undefined : 'primary'
  );
}
```

Feature developers never touch `mat-raised-button` directly. Three variants is all FinancialApp needs.

### FinDataTable

`FinDataTable` wraps Material's table with sorting, pagination, an empty-state template, and row selection exposed as a signal:

```typescript
// libs/shared/ui/src/lib/data-table/fin-data-table.component.ts
@Component({
  selector: 'fin-data-table',
  imports: [MatTableModule, MatSortModule, MatPaginatorModule],
  template: `
    @if (data().length === 0) {
      <ng-container *ngTemplateOutlet="emptyTemplate() ?? defaultEmpty" />
      <ng-template #defaultEmpty>
        <div class="fin-table-empty"><p>No records found.</p></div>
      </ng-template>
    } @else {
      <div class="fin-table-container">
        <table mat-table [dataSource]="dataSource" matSort>
          <ng-content />
          <tr mat-header-row *matHeaderRowDef="columns()" />
          <tr mat-row *matRowDef="let row; columns: columns()"
              [class.selected]="selection.isSelected(row)"
              (click)="toggleRow(row)" />
        </table>
        @if (paginate()) {
          <mat-paginator [pageSizeOptions]="[10, 25, 50]" showFirstLastButtons />
        }
      </div>
    }
  `,
  styles: `
    .fin-table-container { overflow-x: auto; border-radius: var(--fin-radius-md); }
    .fin-table-empty { padding: var(--fin-space-2xl); text-align: center; }
    .selected { background: color-mix(in srgb, var(--fin-color-primary) 8%, transparent); }
  `,
})
export class FinDataTableComponent<T> implements AfterViewInit {
  readonly data = input.required<T[]>();
  readonly columns = input.required<string[]>();
  readonly paginate = input(true);
  readonly rowSelected = output<T[]>();
  readonly emptyTemplate = contentChild<TemplateRef<unknown>>('emptyState');
  private readonly sort = viewChild(MatSort);
  private readonly paginator = viewChild(MatPaginator);
  readonly selection = new SelectionModel<T>(true, []);
  protected dataSource = new MatTableDataSource<T>();

  ngAfterViewInit(): void {
    if (this.sort()) this.dataSource.sort = this.sort()!;
    if (this.paginator()) this.dataSource.paginator = this.paginator()!;
  }

  toggleRow(row: T): void {
    this.selection.toggle(row);
    this.rowSelected.emit(this.selection.selected);
  }
}
```

The empty-state slot uses `contentChild()` to detect a template reference named `emptyState`, following the signal query patterns from [Chapter 10](ch10-signal-queries.md). Consumers define columns using Material's `matColumnDef` directives inside the projected content.

### FinFormField

`FinFormField` wraps `MatFormField` and integrates the error display patterns from [Chapter 23](ch23-error-handling.md). The `contentChild(NgControl)` query finds the projected form control, reads its validation state reactively, and maps error keys to human-readable messages:

```typescript
// libs/shared/ui/src/lib/form-field/fin-form-field.component.ts
@Component({
  selector: 'fin-form-field',
  imports: [MatFormFieldModule],
  template: `
    <mat-form-field appearance="outline" subscriptSizing="dynamic">
      @if (label()) { <mat-label>{{ label() }}</mat-label> }
      <ng-content select="input, textarea, mat-select" />
      @if (hint()) { <mat-hint>{{ hint() }}</mat-hint> }
      @if (showError()) { <mat-error>{{ errorMessage() }}</mat-error> }
    </mat-form-field>
  `,
  styles: `mat-form-field { width: 100%; }`,
})
export class FinFormFieldComponent {
  readonly label = input<string>();
  readonly hint = input<string>();
  readonly errorMap = input<Record<string, string>>({});
  private readonly control = contentChild(NgControl);

  protected readonly showError = computed(() => {
    const ctrl = this.control();
    return ctrl?.touched && ctrl?.invalid;
  });
  protected readonly errorMessage = computed(() => {
    const errors = this.control()?.errors;
    if (!errors) return '';
    const firstKey = Object.keys(errors)[0];
    return this.errorMap()[firstKey] ?? `Invalid value (${firstKey})`;
  });
}
```

Consumers declare validation messages declaratively with the `errorMap` input instead of scattering `@if (form.get('amount')?.hasError('required'))` across templates.

### FinAccountSelector

`FinAccountSelector` composes the Angular Aria Combobox from [Chapter 22](ch22-accessibility-aria.md) with account search, type-ahead filtering, and account-type icons:

```typescript
// libs/shared/ui/src/lib/account-selector/fin-account-selector.component.ts
export interface FinAccount {
  id: number;
  name: string;
  type: 'checking' | 'savings' | 'investment' | 'credit';
  maskedNumber: string;
}

const ACCOUNT_TYPE_ICONS: Record<FinAccount['type'], string> = {
  checking: 'account_balance', savings: 'savings',
  investment: 'trending_up', credit: 'credit_card',
};

@Component({
  selector: 'fin-account-selector',
  imports: [NgCombobox, NgComboboxInput, NgComboboxListbox, NgComboboxOption, FinIconComponent],
  template: `
    <div ngCombobox [ngComboboxDisplayWith]="displayWith"
         (ngComboboxValueChange)="accountSelected.emit($event)">
      <label [id]="labelId()">{{ label() }}</label>
      <input ngComboboxInput [attr.aria-labelledby]="labelId()"
             [placeholder]="placeholder()" (input)="onFilter($event)" />
      <ul ngComboboxListbox>
        @for (account of filteredAccounts(); track account.id) {
          <li [ngComboboxOption]="account">
            <fin-icon [name]="ACCOUNT_TYPE_ICONS[account.type]" [size]="20" />
            <span>{{ account.name }}</span>
            <span class="muted">{{ account.maskedNumber }}</span>
          </li>
        }
      </ul>
    </div>
  `,
  styles: `
    ul { list-style: none; padding: 0; }
    li { display: flex; align-items: center; gap: var(--fin-space-sm);
         padding: var(--fin-space-sm) var(--fin-space-md); cursor: pointer; }
    li:hover { background: var(--fin-color-surface-variant); }
    .muted { color: var(--fin-color-on-surface-variant); }
  `,
})
export class FinAccountSelectorComponent {
  readonly accounts = input.required<FinAccount[]>();
  readonly label = input('Select account');
  readonly placeholder = input('Search accounts...');
  readonly labelId = input('fin-account-label');
  readonly accountSelected = output<FinAccount>();
  protected readonly ACCOUNT_TYPE_ICONS = ACCOUNT_TYPE_ICONS;
  private readonly filterText = signal('');

  protected readonly filteredAccounts = computed(() => {
    const text = this.filterText().toLowerCase();
    if (!text) return this.accounts();
    return this.accounts().filter(a =>
      a.name.toLowerCase().includes(text) || a.maskedNumber.includes(text));
  });
  protected readonly displayWith = (a: FinAccount) => `${a.name} (${a.maskedNumber})`;
  protected onFilter(e: Event): void {
    this.filterText.set((e.target as HTMLInputElement).value);
  }
}
```

Angular Aria handles all the ARIA plumbing: `role="combobox"`, `aria-expanded`, `aria-activedescendant`, and keyboard navigation. This component layers domain logic on top.

### FinCurrencyInput

Currency inputs are deceptively complex. The display must use locale-appropriate formatting from [Chapter 15](ch15-internationalization.md), but the form control must hold the raw numeric value:

```typescript
// libs/shared/ui/src/lib/currency-input/fin-currency-input.component.ts
@Component({
  selector: 'fin-currency-input',
  imports: [MatInputModule],
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => FinCurrencyInputComponent),
    multi: true,
  }],
  template: `
    <input matInput [value]="displayValue()" inputmode="decimal"
           (focus)="onFocus()" (blur)="onBlur()" (input)="onInput($event)"
           [attr.aria-label]="ariaLabel()" class="fin-currency-input" />
  `,
  styles: `.fin-currency-input {
    font-variant-numeric: tabular-nums;
    font-family: var(--fin-font-family-mono, 'Roboto Mono', monospace);
    text-align: right;
  }`,
})
export class FinCurrencyInputComponent implements ControlValueAccessor {
  readonly currencyCode = input('USD');
  readonly ariaLabel = input('Currency amount');
  private readonly locale = inject(LOCALE_ID);
  private readonly currencyPipe = new CurrencyPipe(this.locale);
  private readonly rawValue = signal<number | null>(null);
  private readonly focused = signal(false);

  protected readonly displayValue = computed(() => {
    const value = this.rawValue();
    if (value === null) return '';
    if (this.focused()) return String(value);
    return this.currencyPipe.transform(value, this.currencyCode(), 'symbol', '1.2-2') ?? '';
  });

  private onChange: (v: number | null) => void = () => {};
  private onTouched: () => void = () => {};
  writeValue(v: number | null): void { this.rawValue.set(v); }
  registerOnChange(fn: (v: number | null) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  protected onFocus(): void { this.focused.set(true); }
  protected onBlur(): void { this.focused.set(false); this.onTouched(); }
  protected onInput(e: Event): void {
    const parsed = parseFloat((e.target as HTMLInputElement).value);
    const value = isNaN(parsed) ? null : parsed;
    this.rawValue.set(value);
    this.onChange(value);
  }
}
```

When focused, the user sees the raw number for easy editing. On blur, the number is formatted with the locale-aware `CurrencyPipe`. The `inputmode="decimal"` attribute triggers a numeric keyboard on mobile.

---

## Angular Animations

Angular's animation system provides a declarative API for transitions tied to component state. While CSS transitions handle simple hover effects, Angular animations integrate with the component lifecycle -- they trigger on route changes, respond to signal state, and coordinate staggered sequences across dynamic lists.

### API Fundamentals

The API in `@angular/animations` consists of composable functions: `trigger()` names an animation, `state()` defines styles for a state value, `transition()` describes how to animate between states, `animate()` specifies duration and easing, `keyframes()` defines multi-step animations, `stagger()` offsets timing for list elements, `query()` selects child elements, and `group()` runs animations in parallel.

### Reusable Animation File

FinancialApp centralizes animation definitions in a single exportable file. The easing curves follow Material Design 3's motion specification: the standard curve for entering elements, the decelerate curve for arriving elements, and the accelerate curve for leaving elements:

```typescript
// libs/shared/design-system/src/lib/animations/_animations.ts
import {
  trigger, transition, style, animate, query,
  stagger, group, state, AnimationTriggerMetadata,
} from '@angular/animations';

export const fadeIn = trigger('fadeIn', [
  transition(':enter', [
    style({ opacity: 0 }),
    animate('250ms ease-out', style({ opacity: 1 })),
  ]),
]);

export const listStagger = trigger('listStagger', [
  transition('* => *', [
    query(':enter', [
      style({ opacity: 0, transform: 'translateY(12px)' }),
      stagger('40ms', [
        animate('250ms cubic-bezier(0.2, 0, 0, 1)',
          style({ opacity: 1, transform: 'translateY(0)' })),
      ]),
    ], { optional: true }),
  ]),
]);

export const dialogEnterLeave = trigger('dialogEnterLeave', [
  transition(':enter', [
    style({ opacity: 0, transform: 'scale(0.92)' }),
    animate('200ms cubic-bezier(0, 0, 0, 1)', style({ opacity: 1, transform: 'scale(1)' })),
  ]),
  transition(':leave', [
    animate('150ms cubic-bezier(0.3, 0, 1, 1)', style({ opacity: 0, transform: 'scale(0.92)' })),
  ]),
]);

export const toastSlideIn = trigger('toastSlideIn', [
  transition(':enter', [
    style({ transform: 'translateX(100%)', opacity: 0 }),
    animate('300ms cubic-bezier(0.2, 0, 0, 1)',
      style({ transform: 'translateX(0)', opacity: 1 })),
  ]),
  transition(':leave', [
    animate('200ms cubic-bezier(0.3, 0, 1, 1)',
      style({ transform: 'translateX(100%)', opacity: 0 })),
  ]),
]);

export const themeCrossfade = trigger('themeCrossfade', [
  state('light', style({ opacity: 1 })),
  state('dark', style({ opacity: 1 })),
  transition('light <=> dark', [animate('400ms cubic-bezier(0.2, 0, 0, 1)')]),
]);

export const routeFadeSlide = trigger('routeFadeSlide', [
  transition('* <=> *', [
    query(':enter', [style({ opacity: 0, transform: 'translateY(8px)' })], { optional: true }),
    group([
      query(':leave', [
        animate('200ms cubic-bezier(0.3, 0, 1, 1)',
          style({ opacity: 0, transform: 'translateY(-8px)' })),
      ], { optional: true }),
      query(':enter', [
        animate('300ms 100ms cubic-bezier(0.2, 0, 0, 1)',
          style({ opacity: 1, transform: 'translateY(0)' })),
      ], { optional: true }),
    ]),
  ]),
]);
```

### Applying Animations

**Dialog enter/leave.** The `dialogEnterLeave` trigger gives transfer confirmation dialogs a scale-and-fade effect. Apply `@dialogEnterLeave` to the dialog's content wrapper; Angular runs the enter animation when the dialog opens and the leave animation on close.

**Toast slide-in.** The error toast container from [Chapter 23](ch23-error-handling.md) applies `@toastSlideIn` on each toast element. Because the `@for` block tracks by `toast.id`, Angular correctly identifies which toasts are entering and leaving.

**List stagger.** Binding `[@listStagger]="transactions().length"` on a container triggers the stagger whenever the list changes. Each row fades in and slides up, offset by 40ms. The `{ optional: true }` flag prevents errors when the list is empty.

**Route transitions.** The `routeFadeSlide` animation binds to `outlet.activatedRouteData` on the router outlet. The outgoing page fades up while the incoming page fades in from below with a 100ms overlap:

```typescript
// apps/financial-app/src/app/app.component.ts
@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  animations: [routeFadeSlide],
  template: `
    <fin-nav-toolbar />
    <main>
      <div [@routeFadeSlide]="outlet.activatedRouteData">
        <router-outlet #outlet="outlet" />
      </div>
    </main>
  `,
})
export class AppComponent {}
```

**Theme crossfade.** The `themeCrossfade` trigger wraps the application shell, smoothing the transition between light and dark modes with a 400ms fade instead of an abrupt flash.

### Why provideAnimationsAsync()

In [Chapter 1](ch01-getting-started.md), the bootstrap uses `provideAnimationsAsync()` rather than `provideAnimations()`. The async variant defers the animation engine (~60KB) to a separate chunk that loads after the initial bundle. The application renders immediately; animations load transparently on first interaction. This keeps the critical rendering path lean without any behavioral change:

```typescript
// apps/financial-app/src/app/app.config.ts
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';

export const appConfig: ApplicationConfig = {
  providers: [provideAnimationsAsync()],
};
```

---

## Summary

A design system is not a component library -- it is a set of decisions encoded in code. Tokens define the visual vocabulary. Typography mixins enforce consistent type treatment. The `ThemeService` manages light and dark modes through CSS custom properties and a single reactive signal. Angular Material provides the accessible component foundation, and domain-aware wrappers constrain it to the patterns FinancialApp actually needs.

- **Tokens over hardcoded values.** Every color, spacing unit, radius, and motion duration is a CSS custom property in `_tokens.scss`. Themes override tokens, not components.
- **Wrappers over direct Material usage.** Feature developers use `FinButton`, not `mat-raised-button`. This centralizes visual decisions, reduces API surface, and isolates upgrade paths.
- **Headless + styled composition.** The Angular Aria Combobox from [Chapter 22](ch22-accessibility-aria.md) handles ARIA semantics. `FinAccountSelector` adds domain logic and token-based styling on top.
- **Centralized animations.** All motion definitions live in `_animations.ts` with consistent Material Design 3 easing curves. Components import triggers rather than defining animation logic inline.
- **Deferred animation loading.** `provideAnimationsAsync()` keeps the animation engine out of the critical rendering path with no behavioral change.

The design system lives in `libs/shared/design-system/` (tokens, themes, `ThemeService`, `FinIcon`, animations) and `libs/shared/ui/` (wrapper components). This separation keeps the visual foundation independent of the domain components that consume it -- a library boundary that pays dividends as the application scales across teams.
