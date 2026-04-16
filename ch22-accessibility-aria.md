# Chapter 22: Accessibility, Angular Aria & a11y Best Practices

An application that cannot be used by everyone is an incomplete application. Accessibility is not a feature toggle you flip before launch -- it is an architectural constraint that shapes how you build components, manage focus, announce dynamic content, and structure navigation. Users who rely on screen readers, keyboard navigation, or alternative input devices need the same reliable access to their accounts, transactions, and portfolios as everyone else.

This chapter treats WCAG 2.2 Level AA as the baseline for FinancialApp. We will cover ARIA attribute binding, accessible progress bars, focus management after route changes, deferred content, headless accessible widgets with `@angular/aria`, focus utilities from `@angular/cdk/a11y`, and automated testing with `axe-core`.

> **Forward reference:** The headless Angular Aria components built in this chapter (Toolbar, Tabs, Combobox) are wrapped with design tokens and domain logic in [Chapter 27](ch27-material-design-system.md) to create production-ready FinancialApp components like `FinAccountSelector`.

---

## Why Accessibility Matters

The case rests on three pillars. **Legal:** the ADA, European Accessibility Act, and Section 508 impose enforceable requirements, especially in financial services. **Ethical:** roughly 15% of the global population lives with some form of disability. **Business:** accessible applications are better for everyone -- keyboard navigability improves power-user workflows, semantic markup helps SEO, and clear focus management reduces confusion during flows like fund transfers. WCAG 2.2 Level AA is the standard we target.

---

## ARIA Attributes in Angular Templates

Angular provides several mechanisms for applying ARIA attributes, depending on whether the value is static, dynamic, or tied to an element reference.

### Static and Dynamic ARIA Attributes

When the value never changes, use a plain HTML attribute. When it depends on component state, use `[attr.aria-*]`:

```html
<!-- financial-app/libs/shared/ui/src/lib/nav/nav.component.html -->
<nav aria-label="Main navigation">
  <a routerLink="/accounts">Accounts</a>
  <a routerLink="/transactions">Transactions</a>
</nav>
```

```html
<!-- financial-app/libs/shared/ui/src/lib/progress/fin-progress-bar.component.html -->
<div role="progressbar"
     aria-valuemin="0"
     aria-valuemax="100"
     [attr.aria-valuenow]="currentValue()"
     [attr.aria-label]="label()">
  <div class="bar-fill" [style.width.%]="currentValue()"></div>
</div>
```

The `attr.` prefix is essential. ARIA attributes are not DOM properties, so `[aria-valuenow]` without `attr.` will not work -- the binding compiles without error but the attribute never appears in the DOM.

### Element Reference Binding

Some ARIA properties accept element references rather than string IDs. Angular exposes these through property bindings:

```html
<!-- financial-app/libs/shared/ui/src/lib/form-field/form-field.component.html -->
<label #fieldLabel>Account name</label>
<input type="text"
       [ariaLabelledByElements]="[fieldLabel]"
       [ariaDescribedByElements]="[fieldHint, fieldError]" />
<span #fieldHint class="hint">Enter the display name for this account.</span>
<span #fieldError class="error" [hidden]="!hasError()">Name is required.</span>
```

This avoids the brittleness of manually synchronizing `id` and `aria-labelledby` string attributes -- Angular manages the underlying DOM relationship.

### Host Binding for ARIA in Components

When a component *is* the semantic element -- a custom button, a badge, a tab -- ARIA attributes belong on the host element. Use the `host` object in the component metadata:

```typescript
// financial-app/libs/shared/ui/src/lib/badge/fin-status-badge.component.ts
@Component({
  selector: 'fin-status-badge',
  host: {
    'role': 'status',
    '[attr.aria-label]': 'statusLabel()',
  },
  template: `<span class="badge" [class]="variant()">{{ text() }}</span>`,
})
export class FinStatusBadgeComponent {
  text = input.required<string>();
  variant = input<'success' | 'warning' | 'error'>('success');
  protected statusLabel = computed(() => `Status: ${this.text()}`);
}
```

Prefer the `host` object over `@HostBinding` decorators -- it keeps all host interactions in one place and makes the accessibility contract explicit.

---

## Case Study: Accessible Progress Bar

A progress bar is visually simple but requires careful ARIA markup. FinancialApp uses one during portfolio data loads:

```typescript
// financial-app/libs/shared/ui/src/lib/progress/fin-progress-bar.component.ts
import { Component, computed, input } from '@angular/core';

@Component({
  selector: 'fin-progress-bar',
  host: {
    'role': 'progressbar',
    '[attr.aria-valuenow]': 'value()',
    '[attr.aria-valuemin]': 'min()',
    '[attr.aria-valuemax]': 'max()',
    '[attr.aria-label]': 'label()',
  },
  template: `
    <div class="track">
      <div class="fill" [style.width.%]="percentage()"></div>
    </div>
  `,
})
export class FinProgressBarComponent {
  value = input.required<number>();
  min = input(0);
  max = input(100);
  label = input('Loading progress');

  protected percentage = computed(() => {
    const range = this.max() - this.min();
    return range > 0 ? ((this.value() - this.min()) / range) * 100 : 0;
  });
}
```

The ARIA attributes live on the host because the component *is* the progress bar in the accessibility tree. A screen reader announces "Loading progress, progressbar, 45%." Without these attributes, it sees an empty `<fin-progress-bar>` and says nothing. Usage:

```html
<!-- financial-app/apps/financial-app/src/app/features/portfolio/portfolio-overview.component.html -->
@if (loadingProgress(); as progress) {
  <fin-progress-bar [value]="progress" label="Loading portfolio data" />
}
```

---

## Focus Management After Route Navigation

Single-page applications break the browser's natural focus model. A traditional page load moves focus to the top of the document; an SPA route change swaps content silently, leaving focus on the navigation link the user just clicked. The screen reader user hears nothing about the new page.

The fix: after each successful navigation, move focus to the new page's main heading:

```typescript
// financial-app/apps/financial-app/src/app/core/a11y/route-focus.service.ts
import { Injectable, inject } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class RouteFocusService {
  private router = inject(Router);

  initialize(): void {
    this.router.events.pipe(
      filter((e): e is NavigationEnd => e instanceof NavigationEnd),
    ).subscribe(() => {
      const heading = document.querySelector<HTMLElement>('main h1');
      if (heading) {
        heading.setAttribute('tabindex', '-1');
        heading.focus();
        heading.addEventListener('blur',
          (e) => (e.target as HTMLElement).removeAttribute('tabindex'), { once: true });
      }
    });
  }
}
```

`tabindex="-1"` makes the heading programmatically focusable without adding it to the tab order; removing it on blur keeps the tab sequence clean. Initialize this service at bootstrap via `APP_INITIALIZER` as described in [Chapter 12](ch12-initialization-routes.md).

---

## Active Link Identification

Sighted users see the highlighted nav item; screen reader users need an equivalent signal. `RouterLinkActive` provides `ariaCurrentWhenActive`:

```html
<!-- financial-app/apps/financial-app/src/app/shell/sidebar-nav.component.html -->
<nav aria-label="Main navigation">
  <a routerLink="/accounts" routerLinkActive="active" ariaCurrentWhenActive="page">Accounts</a>
  <a routerLink="/transactions" routerLinkActive="active" ariaCurrentWhenActive="page">Transactions</a>
  <a routerLink="/portfolio" routerLinkActive="active" ariaCurrentWhenActive="page">Portfolio</a>
  <a routerLink="/clients" routerLinkActive="active" ariaCurrentWhenActive="page">Clients</a>
</nav>
```

When a link becomes active, Angular adds `aria-current="page"`. Screen readers announce "current page," giving the user the same orientation that the visual highlight provides. Use `"step"` for multi-step wizards or `"true"` for generic active items.

---

## Defer Block Accessibility

`@defer` blocks ([Chapter 17](ch17-defer-ssr-hydration.md)) present an accessibility challenge: content appears asynchronously, and screen readers have no way to know it arrived. Wrap deferred regions in an ARIA live region:

```html
<!-- financial-app/apps/financial-app/src/app/features/portfolio/portfolio-overview.component.html -->
<div aria-live="polite" aria-atomic="true">
  @defer (on viewport) {
    <app-holdings-table [holdings]="holdings()" />
  } @placeholder {
    <p>Holdings data will load when you scroll here.</p>
  } @loading (minimum 300ms) {
    <p aria-busy="true">Loading holdings...</p>
  }
</div>
```

`aria-live="polite"` announces changes at the next pause in speech. `aria-atomic="true"` re-reads the entire region, not just the delta. `aria-busy="true"` signals a transitional state. Without this wrapper, a screen reader user who scrolled past the placeholder would never learn that the holdings table arrived.

---

## Angular Aria Library

`@angular/aria` provides headless directive primitives that implement WAI-ARIA design patterns -- keyboard interactions, focus management, and ARIA attribute wiring without imposing visual design. You bring the styles; the library brings the behavior.

```bash
npm install @angular/aria
```

### Toolbar

The toolbar pattern groups controls and adds arrow-key navigation:

```typescript
// financial-app/libs/shared/ui/src/lib/toolbar/fin-formatting-toolbar.component.ts
import { Component } from '@angular/core';
import { NgToolbar, NgToolbarWidget, NgToolbarWidgetGroup } from '@angular/aria';

@Component({
  selector: 'fin-formatting-toolbar',
  imports: [NgToolbar, NgToolbarWidget, NgToolbarWidgetGroup],
  template: `
    <div ngToolbar aria-label="Text formatting">
      <div ngToolbarWidgetGroup aria-label="Font style">
        <button ngToolbarWidget (click)="applyFormat('bold')">Bold</button>
        <button ngToolbarWidget (click)="applyFormat('italic')">Italic</button>
        <button ngToolbarWidget (click)="applyFormat('underline')">Underline</button>
      </div>
      <div ngToolbarWidgetGroup aria-label="Alignment">
        <button ngToolbarWidget (click)="applyFormat('left')">Left</button>
        <button ngToolbarWidget (click)="applyFormat('center')">Center</button>
        <button ngToolbarWidget (click)="applyFormat('right')">Right</button>
      </div>
    </div>`,
})
export class FinFormattingToolbarComponent {
  applyFormat(format: string): void { /* delegate to editor service */ }
}
```

`ngToolbar` sets `role="toolbar"`, manages roving tabindex, and handles arrow-key navigation. `ngToolbarWidgetGroup` provides `role="group"`. Only the first widget in each group gets `tabindex="0"`; the rest get `tabindex="-1"` until arrowed to. Tab exits the toolbar entirely, matching the WAI-ARIA specification.

The same pattern adapts to FinancialApp's navigation -- replace `<button>` elements with `<a ngToolbarWidget routerLink="...">` links and the nav bar gains arrow-key navigation for free:

```typescript
// financial-app/libs/shared/ui/src/lib/nav/fin-nav-toolbar.component.ts
@Component({
  selector: 'fin-nav-toolbar',
  imports: [NgToolbar, NgToolbarWidget, RouterLink, RouterLinkActive],
  template: `
    <nav ngToolbar aria-label="Main navigation">
      <a ngToolbarWidget routerLink="/accounts"
         routerLinkActive="active" ariaCurrentWhenActive="page">Accounts</a>
      <a ngToolbarWidget routerLink="/transactions"
         routerLinkActive="active" ariaCurrentWhenActive="page">Transactions</a>
      <a ngToolbarWidget routerLink="/portfolio"
         routerLinkActive="active" ariaCurrentWhenActive="page">Portfolio</a>
    </nav>`,
})
export class FinNavToolbarComponent {}
```

### Tabs

Account-type switching maps naturally to the tab pattern:

```typescript
// financial-app/libs/shared/ui/src/lib/tabs/fin-account-tabs.component.ts
import { Component, signal } from '@angular/core';
import { NgTabList, NgTab, NgTabPanel } from '@angular/aria';

@Component({
  selector: 'fin-account-tabs',
  imports: [NgTabList, NgTab, NgTabPanel],
  template: `
    <div ngTabList aria-label="Account types">
      <button ngTab [ngTabPanelId]="'panel-checking'"
              [ngTabSelected]="activeTab() === 'checking'"
              (click)="activeTab.set('checking')">Checking</button>
      <button ngTab [ngTabPanelId]="'panel-savings'"
              [ngTabSelected]="activeTab() === 'savings'"
              (click)="activeTab.set('savings')">Savings</button>
      <button ngTab [ngTabPanelId]="'panel-investment'"
              [ngTabSelected]="activeTab() === 'investment'"
              (click)="activeTab.set('investment')">Investment</button>
    </div>
    <!-- One panel per tab; each uses ng-content with a named slot -->
    <div ngTabPanel id="panel-checking" [hidden]="activeTab() !== 'checking'">
      <ng-content select="[checking]" /></div>
    <div ngTabPanel id="panel-savings" [hidden]="activeTab() !== 'savings'">
      <ng-content select="[savings]" /></div>
    <div ngTabPanel id="panel-investment" [hidden]="activeTab() !== 'investment'">
      <ng-content select="[investment]" /></div>
  `,
})
export class FinAccountTabsComponent {
  activeTab = signal<'checking' | 'savings' | 'investment'>('checking');
}
```

The directives handle the full ARIA contract: `ngTabList` sets `role="tablist"` and manages arrow-key navigation. `ngTab` applies `role="tab"`, `aria-selected`, and `aria-controls`. `ngTabPanel` sets `role="tabpanel"` and `aria-labelledby`. Left/Right arrows move between tabs, Home/End jump to boundaries, and only the selected tab has `tabindex="0"`.

### Combobox / Autocomplete

Transaction categorization uses an autocomplete -- one of the most complex ARIA widgets:

```typescript
// financial-app/libs/shared/ui/src/lib/combobox/fin-category-combobox.component.ts
import { Component, computed, signal, input } from '@angular/core';
import { NgCombobox, NgComboboxInput, NgComboboxListbox, NgComboboxOption } from '@angular/aria';

@Component({
  selector: 'fin-category-combobox',
  imports: [NgCombobox, NgComboboxInput, NgComboboxListbox, NgComboboxOption],
  template: `
    <div ngCombobox>
      <label id="category-label">Transaction category</label>
      <input ngComboboxInput aria-labelledby="category-label"
             [placeholder]="'Search categories...'" (input)="onFilter($event)" />
      @if (filteredCategories().length > 0) {
        <ul ngComboboxListbox>
          @for (cat of filteredCategories(); track cat) {
            <li ngComboboxOption [ngComboboxOptionValue]="cat"
                (ngComboboxOptionSelected)="selectCategory(cat)">{{ cat }}</li>
          }
        </ul>
      }
    </div>
  `,
})
export class FinCategoryComboboxComponent {
  categories = input.required<string[]>();
  private filterText = signal('');
  protected filteredCategories = computed(() => {
    const text = this.filterText().toLowerCase();
    return text ? this.categories().filter(c => c.toLowerCase().includes(text)) : this.categories();
  });

  onFilter(event: Event): void {
    this.filterText.set((event.target as HTMLInputElement).value);
  }
  selectCategory(category: string): void { /* emit selection to parent form */ }
}
```

`ngCombobox` manages `aria-expanded` and `aria-haspopup`. `ngComboboxInput` binds `aria-autocomplete="list"` and `aria-activedescendant`. `ngComboboxOption` sets `role="option"` and `aria-selected`. Down/Up arrows navigate, Enter selects, Escape closes. Implementing this from scratch requires over a dozen coordinated attributes; `@angular/aria` handles all of it.

---

## Angular CDK a11y

`@angular/cdk/a11y` provides lower-level accessibility utilities that complement `@angular/aria`. Where Angular Aria gives you complete widget patterns, CDK a11y gives you building blocks.

### LiveAnnouncer

Dynamic content changes -- a transaction saved, a validation error, a sync completing -- are invisible to screen readers unless explicitly announced:

```typescript
// financial-app/apps/financial-app/src/app/features/transactions/transaction-form.component.ts
import { Component, inject } from '@angular/core';
import { LiveAnnouncer } from '@angular/cdk/a11y';

@Component({ selector: 'fin-transaction-form', templateUrl: './transaction-form.component.html' })
export class TransactionFormComponent {
  private announcer = inject(LiveAnnouncer);

  async saveTransaction(): Promise<void> {
    try {
      await this.transactionService.save(this.formData());
      this.announcer.announce('Transaction saved successfully.', 'polite');
    } catch {
      this.announcer.announce('Failed to save transaction. Please try again.', 'assertive');
    }
  }
}
```

`'polite'` waits for the screen reader to finish speaking; `'assertive'` interrupts. Use assertive sparingly -- errors and critical state changes only.

### Focus Trapping in Modal Dialogs

When a modal opens, focus must be trapped inside it. The CDK's `cdkTrapFocus` directive handles Tab/Shift+Tab cycling. Building on the confirm dialog from [Chapter 11](ch11-directives-templates.md):

```typescript
// financial-app/libs/shared/ui/src/lib/dialog/fin-confirm-dialog.component.ts
import { Component, input, output } from '@angular/core';
import { A11yModule } from '@angular/cdk/a11y';

@Component({
  selector: 'fin-confirm-dialog',
  imports: [A11yModule],
  template: `
    <div class="dialog-backdrop" (click)="cancelled.emit()">
      <div class="dialog-panel" role="alertdialog" [attr.aria-label]="title()"
           aria-modal="true" cdkTrapFocus cdkTrapFocusAutoCapture
           (click)="$event.stopPropagation()">
        <h2>{{ title() }}</h2>
        <p>{{ message() }}</p>
        <div class="dialog-actions">
          <button class="btn-secondary" (click)="cancelled.emit()">Cancel</button>
          <button class="btn-primary" (click)="confirmed.emit()">Confirm</button>
        </div>
      </div>
    </div>`,
})
export class FinConfirmDialogComponent {
  title = input('Confirm Action');
  message = input('Are you sure you want to proceed?');
  confirmed = output<void>();
  cancelled = output<void>();
}
```

`cdkTrapFocusAutoCapture` moves focus into the trap on activation and restores it when the trap is destroyed.

### FocusMonitor

A button focused by keyboard should show a visible focus ring; one clicked with a mouse should not. `FocusMonitor` distinguishes the origin:

```typescript
// financial-app/libs/shared/ui/src/lib/button/fin-button.component.ts
import { Component, ElementRef, inject, OnDestroy } from '@angular/core';
import { FocusMonitor } from '@angular/cdk/a11y';

@Component({
  selector: 'fin-button',
  host: { '[class.keyboard-focused]': 'focusedByKeyboard' },
  template: `<ng-content />`,
})
export class FinButtonComponent implements OnDestroy {
  private focusMonitor = inject(FocusMonitor);
  private el = inject(ElementRef);
  protected focusedByKeyboard = false;

  constructor() {
    this.focusMonitor.monitor(this.el, true)
      .subscribe(origin => { this.focusedByKeyboard = origin === 'keyboard'; });
  }
  ngOnDestroy(): void { this.focusMonitor.stopMonitoring(this.el); }
}
```

Style `.keyboard-focused` with a prominent focus ring. The `:focus-visible` CSS pseudo-class achieves similar results natively, but `FocusMonitor` gives you programmatic control for complex scenarios like composite widgets.

---

## Testing Accessibility with axe-core

Automated testing catches missing labels, invalid ARIA attributes, and broken role hierarchies. `axe-core` is the industry-standard engine.

```bash
npm install -D axe-core
```

```typescript
// financial-app/libs/shared/ui/src/testing/expect-accessible.ts
import axe from 'axe-core';
import { ComponentFixture } from '@angular/core/testing';

export async function expectAccessible(fixture: ComponentFixture<unknown>): Promise<void> {
  fixture.detectChanges();
  const { violations } = await axe.run(fixture.nativeElement);
  expect(violations.map(v => ({
    id: v.id, impact: v.impact, description: v.description, nodes: v.nodes.map(n => n.html),
  }))).withContext('axe-core accessibility violations').toEqual([]);
}
```

### Structural and Keyboard Tests

The helper validates ARIA structure; keyboard behavior needs explicit tests alongside it:

```typescript
// financial-app/libs/shared/ui/src/lib/toolbar/fin-formatting-toolbar.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { FinFormattingToolbarComponent } from './fin-formatting-toolbar.component';
import { expectAccessible } from '../../testing/expect-accessible';

describe('FinFormattingToolbarComponent', () => {
  it('should have no axe-core violations', async () => {
    const fixture = TestBed.createComponent(FinFormattingToolbarComponent);
    await expectAccessible(fixture);
  });

  it('should move focus with ArrowRight and wrap at boundary', () => {
    const fixture = TestBed.createComponent(FinFormattingToolbarComponent);
    fixture.detectChanges();
    const buttons: HTMLButtonElement[] = fixture.nativeElement.querySelectorAll('button');

    buttons[0].focus();
    buttons[0].dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowRight' }));
    fixture.detectChanges();
    expect(document.activeElement).toBe(buttons[1]);

    buttons[buttons.length - 1].focus();
    buttons[buttons.length - 1].dispatchEvent(
      new KeyboardEvent('keydown', { key: 'ArrowRight' }));
    fixture.detectChanges();
    expect(document.activeElement).toBe(buttons[0]);
  });

  it('should activate widget on Enter', () => {
    const fixture = TestBed.createComponent(FinFormattingToolbarComponent);
    fixture.detectChanges();
    const spy = spyOn(fixture.componentInstance, 'applyFormat');

    const btn = fixture.nativeElement.querySelector('button');
    btn.focus();
    btn.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter' }));
    fixture.detectChanges();
    expect(spy).toHaveBeenCalledWith('bold');
  });
});
```

The `axe-core` test catches structural ARIA issues; the keyboard tests catch behavioral ones. Both build on the Vitest setup from [Chapter 7](ch07-testing-vitest.md).

---

## Summary

- **ARIA binding** uses `[attr.aria-*]` for dynamic values, plain attributes for static ones, `[ariaLabelledByElements]` for element references, and the `host` object for component-level semantics.
- **Progress bars** require `role="progressbar"` with value attributes on the host.
- **Focus management** after route navigation prevents screen reader context loss.
- **`ariaCurrentWhenActive="page"`** provides the non-visual equivalent of a highlighted nav link.
- **`@defer` blocks** need `aria-live="polite"` wrappers so screen readers announce loaded content.
- **`@angular/aria`** implements WAI-ARIA keyboard and attribute contracts (Toolbar, Tabs, Combobox) without imposing visual design.
- **`@angular/cdk/a11y`** provides `LiveAnnouncer`, `cdkTrapFocus`, and `FocusMonitor`.
- **`axe-core`** paired with keyboard tests provides comprehensive automated coverage.

WCAG AA is the floor, not the ceiling. Real accessibility requires testing with actual assistive technology and involving users with disabilities in your process.

> **Companion code:** `@angular/aria` and `@angular/cdk` are installed in `financial-app/`. Accessible nav toolbar and account-type tabs live in `libs/shared/ui/`. The `axe-core` test helper is in `libs/shared/ui/src/testing/`.
