# Chapter 8: Accessibility Full-Stack

An application that cannot be used by everyone is an incomplete application. Accessibility is not a feature toggle you flip before launch -- it is an architectural constraint that shapes how you build components, manage focus, announce dynamic content, and structure navigation. Users who rely on screen readers, keyboard navigation, or alternative input devices need the same reliable access to their accounts, transactions, and portfolios as everyone else.

This chapter treats WCAG 2.2 Level AA as the baseline for FinancialApp and covers accessibility full-stack: the patterns you ship (ARIA attribute binding, accessible progress bars, focus management, `@angular/aria` widgets, `@angular/cdk/a11y` utilities, automated testing with `axe-core`), the verification you perform to prove those patterns work (screen reader setup, end-to-end walkthroughs with VoiceOver and NVDA, cognitive and mobile accessibility, a Playwright regression fixture), and the governance that keeps accessibility from regressing between releases (VPAT production, linking automated and manual findings, retesting cadence).

> **Prerequisites:** Familiarity with Vitest and component harnesses from [Chapter 7](ch07-testing-vitest.md) for the unit-level accessibility tests, and the Playwright scaffolding from [Chapter 19](ch37-e2e-playwright.md) for the automated a11y regression fixture at the end of the chapter.

> **Forward reference:** The headless Angular Aria components built in this chapter (Toolbar, Tabs, Combobox) are wrapped with design tokens and domain logic in [Chapter 32](ch22-material-design-system.md) to create production-ready FinancialApp components like `FinAccountSelector`. Storybook stories that catalogue each component with axe-core hooks active are set up in [Chapter 28](ch26-storybook.md).

---

## Why Accessibility Matters

The case rests on three pillars. **Legal:** the ADA, European Accessibility Act, and Section 508 impose enforceable requirements, especially in financial services. **Ethical:** roughly 15% of the global population lives with some form of disability. **Business:** accessible applications are better for everyone -- keyboard navigability improves power-user workflows, semantic markup helps SEO, and clear focus management reduces confusion during flows like fund transfers.

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

`tabindex="-1"` makes the heading programmatically focusable without adding it to the tab order; removing it on blur keeps the tab sequence clean. Initialize this service at bootstrap via `APP_INITIALIZER` as described in [Chapter 48](ch13-initialization-routes.md).

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

`@defer` blocks ([Chapter 22](ch31-defer-ssr-hydration.md)) present an accessibility challenge: content appears asynchronously, and screen readers have no way to know it arrived. Wrap deferred regions in an ARIA live region:

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

When a modal opens, focus must be trapped inside it. The CDK's `cdkTrapFocus` directive handles Tab/Shift+Tab cycling. Building on the confirm dialog from [Chapter 47](ch12-directives-templates.md):

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

## Screen Reader Verification

An `axe-core` report that says "zero violations" feels reassuring. It also lies, at least by omission. The audit catches violations a parser can detect -- missing `alt` attributes, invalid ARIA values, insufficient contrast ratios, duplicate landmark roles. It cannot tell you that your login form is technically labeled but announces "edit, edit, button" because every field shares the same generic name. It cannot tell you that a fund-transfer flow, while semantically correct, requires the user to memorize an eight-step sequence of landmarks because you never gave them a heading to anchor to. That only comes out when someone actually tries to use the application with a screen reader.

The rest of this chapter is about closing that gap. We will get VoiceOver and NVDA running, learn just enough shortcuts to navigate like a real user does, walk through three representative FinancialApp flows with assistive technology active, revisit the ARIA patterns from earlier in the chapter with their actual screen-reader behavior audible in the background, and cover cognitive accessibility, mobile a11y, and a Playwright fixture that automates the catchable parts across every route.

### Screen Reader Setup

You do not need every reader. You need one on each operating system you target, one that matches the reader your users use most, and the discipline to exercise them periodically. Do not save this for the week before release -- by then, every issue you find is an emergency.

**VoiceOver (macOS/iOS)** ships with every Mac and iPhone. Toggle it with `Cmd+F5` on macOS (or triple-press the side button on iOS). `Ctrl` silences speech without disabling the reader, which you will want often during testing. The **Rotor** -- `Ctrl+Opt+U` -- is the killer feature: a modal menu that lets you jump between headings, landmarks, links, form controls, or tables on the current page. Arrow left/right to pick a category, up/down to move through items. If your markup is good, the Rotor is fast; if it is bad, the Rotor reveals exactly why.

**NVDA (Windows)** is free and open-source, downloadable from [nvaccess.org](https://www.nvaccess.org). Install it on a Windows VM or Boot Camp partition if you develop on macOS -- Windows Server-side rendering differs enough from WebKit that headless audits miss real bugs. `Insert` is the NVDA modifier key (often called "NVDA key"); `Insert+N` opens the menu, `Insert+Down` starts reading from the cursor, `Ctrl` silences. NVDA's web-browse mode auto-activates on pages and gives you single-letter navigation: `H` for next heading, `K` for next link, `F` for next form control, `D` for next landmark.

**JAWS (Windows)** is the commercial reader with roughly forty-percent market share, particularly strong in enterprise and government deployments. If you ship into financial services or the public sector, you need to exercise JAWS at least occasionally -- its heuristics differ enough from NVDA that fully-compliant markup still produces different announcements. A license is a few hundred dollars per year; Freedom Scientific offers a forty-minute "demo mode" that resets on reboot, which is enough for periodic smoke tests.

**Orca (Linux)** is the GNOME default, mostly used by Linux-first users. If your user research shows any non-trivial Linux audience, exercise it; otherwise, NVDA coverage is usually sufficient.

**TalkBack (Android)** and **VoiceOver (iOS)** on mobile are covered in more detail in [Chapter 44](ch41-capacitor-mobile.md), where we discuss the Capacitor wrapper around FinancialApp. Mobile readers have their own gesture vocabulary (one-finger swipe to move, two-finger swipe to scroll, double-tap to activate) that differs from desktop keyboard navigation.

**Chrome DevTools Accessibility tree** is your quick-and-dirty first-pass inspector. Open DevTools, click the **Accessibility** panel, and inspect any element to see its computed name, role, and state. If the name is empty or the role says "generic," no amount of CSS will fix the experience -- the element is invisible to assistive technology. Keep this panel open while writing components; it catches most regressions before they reach a real screen reader.

One more tip: turn on **captions for speech**. macOS calls this "VoiceOver caption panel" (enable in VoiceOver Utility); NVDA has a "Speech viewer" under the Tools menu. Both display what the reader is speaking as scrolling text, which makes bug reports repeatable. "The save button announces 'button, button' twice" is vastly more useful than "it sounds weird."

#### Browser Pairings

Screen readers and browsers are not interchangeable. Each reader is most mature with a specific browser, and bugs in either half of the pair can masquerade as application bugs. The practical pairings:

- **VoiceOver + Safari** on macOS and iOS. This is the only pairing Apple tests against. Chrome on macOS with VoiceOver is acceptable but buggier; Firefox on macOS with VoiceOver is actively painful.
- **NVDA + Firefox** as the primary Windows combination. NVDA + Chrome is the secondary -- Chrome's accessibility tree matured significantly after v100 but still trails Firefox in edge cases.
- **JAWS + Chrome** is what enterprise Windows users typically run. JAWS + Edge is equivalent since both use Chromium.
- **TalkBack + Chrome** on Android; **VoiceOver + Safari** on iOS. Mobile readers do not support alternative browsers in any meaningful way.

Test against your primary pairing on every feature. Test against the secondary before each release. A bug that only appears on Chrome + VoiceOver is often an actual Chrome bug and not yours -- but if your users are on that combination, they still cannot use the app, so you still have to work around it.

### Learning the Essential Shortcuts

You do not need to learn every command. You need enough to navigate, read, and silence. The rest comes with practice.

**Navigate by semantic element.** Both VoiceOver (via Rotor) and NVDA (via single-letter keys) let you jump between headings, landmarks, form controls, links, and tables without tabbing through every focusable element. In NVDA: `H` next heading, `Shift+H` previous heading, `1`-`6` specific heading level, `D` next landmark, `F` next form field, `K` next link, `T` next table. In VoiceOver: open Rotor with `Ctrl+Opt+U`, arrow to a category, arrow to an item. **If your page has clear headings and landmarks, users never tab.** That is the point -- tabbing through every element is the fallback, not the default.

**Read from the cursor.** VoiceOver: `Ctrl+Opt+A` reads from the current cursor to the end. NVDA: `Insert+Down` does the same, `Insert+Up` reads the current line. This is how users consume long content: they point at a paragraph and tell the reader to continue.

**Stop speech.** VoiceOver: `Ctrl` silences. NVDA: `Ctrl` silences. Use it liberally while testing -- listening to a ten-minute page read-through is rarely necessary.

**Tab vs arrow-key navigation.** This is the part most sighted developers misunderstand. `Tab` cycles through *focusable* elements: buttons, links, form controls, `tabindex="0"` elements. Arrow keys move the **virtual cursor** through *readable* content: every paragraph, heading, list item, and text run. A paragraph of explanatory text is reachable by arrow but not by Tab, because there is no reason to focus on a paragraph. Users mix both: Tab to jump to the form, arrows to read the hints above it, Tab again to move between fields.

One consequence: **never hide informational content behind focus.** A tooltip that appears only on focus is invisible to a user who is arrow-reading the description above a field. Put the description in the DOM, associate it with `aria-describedby`, and the reader announces it automatically when focus reaches the input.

### Walkthrough: Login Flow with VoiceOver

Turn VoiceOver on (`Cmd+F5`), open the login page in Safari, and listen. Here is what the flow should sound like on FinancialApp, and what it often sounds like when things go wrong.

**Step 1 -- Page arrival.** A well-built login page announces: "FinancialApp, web content, Log in, heading level 1, main landmark." The page title (`<title>` in `index.html`), the `<main>` wrapper, and an `<h1>` at the top give the user an instant mental map. A broken version announces "FinancialApp, web content" and then silence, because there is no `<main>`, no heading, and focus landed on the body.

**Step 2 -- Finding the form.** The user presses `Ctrl+Opt+U`, arrows to "Form controls," and hears "Email address, edit text, required." On a broken page they hear "edit text" -- unlabeled. The fix is unambiguous: every `<input>` needs an associated `<label>` or an `aria-label`. The `FinFormField` wrapper covered earlier handles this automatically via `[ariaLabelledByElements]`; raw `<input>` tags do not.

**Step 3 -- Typing and submitting.** The user types a password, tabs to "Log in, button," and presses Return. On a successful login the app navigates to `/dashboard`. Here is where most SPAs silently break: focus stays on the login button, which no longer exists in the DOM, so focus falls back to `<body>` and the user hears nothing. They have no idea what happened. The fix is the `RouteFocusService` covered earlier -- after every navigation, move focus to the new page's `<h1>`. The user now hears "Dashboard, heading level 1" and knows the login succeeded.

**Step 4 -- Handling a failure.** The user enters a wrong password. The app displays an error banner below the form. If the banner is a plain `<div>`, the reader says nothing -- it is on a page the user is no longer actively reading. The fix is `role="alert"` (or the `LiveAnnouncer` service covered earlier) so the banner announces immediately: "Alert. Incorrect email or password. Please try again." After the announcement, focus should move back to the email field so the user can correct their input without hunting.

```typescript
// financial-app/apps/financial-app/src/app/features/auth/login.component.ts
import { Component, inject, viewChild, ElementRef } from '@angular/core';
import { LiveAnnouncer } from '@angular/cdk/a11y';
import { AuthService } from '@financial-app/shared/auth';

@Component({
  selector: 'fin-login',
  templateUrl: './login.component.html',
})
export class LoginComponent {
  private auth = inject(AuthService);
  private announcer = inject(LiveAnnouncer);
  private emailInput = viewChild<ElementRef<HTMLInputElement>>('emailInput');

  async submit(email: string, password: string): Promise<void> {
    try {
      await this.auth.logIn(email, password);
      // RouteFocusService handles focus after navigation.
    } catch (error) {
      this.announcer.announce(
        'Incorrect email or password. Please try again.',
        'assertive',
      );
      this.emailInput()?.nativeElement.focus();
    }
  }
}
```

The three fixes -- label every input, announce the error, restore focus on failure -- are independent and small. Together they turn "logging in with VoiceOver is impossible" into "logging in with VoiceOver feels normal."

### Walkthrough: Transaction Submission with NVDA

Boot Windows, launch NVDA, open FinancialApp, and navigate to an account detail page. We are going to submit a new transaction and observe what the user hears.

**Step 1 -- Opening the form.** The user presses `B` (next button) until they reach "Add transaction, button, opens dialog." Good: `aria-haspopup="dialog"` tells the reader the activation opens something. They press Return; the dialog opens. The reader announces: "Dialog, New transaction, heading level 2." The dialog has `role="dialog"`, `aria-modal="true"`, and an `aria-labelledby` pointing to the heading.

**Step 2 -- Focus trap.** The user presses Tab. Focus moves to the first field. They keep tabbing; focus cycles through the dialog's fields and buttons, never escaping to the page behind it. This is `cdkTrapFocus` from earlier in the chapter doing its job. They press `Escape`; the dialog closes and focus returns to the "Add transaction" button. Without `cdkTrapFocusAutoCapture`, the user would be left on `<body>` and disoriented.

**Step 3 -- Inline validation errors.** The user enters "-500" into the Amount field (negative is invalid for this form) and tabs to Category. Chapter 40 showed the `FinFormField` wrapper linking the error message via `aria-describedby`. The reader announces: "Amount, edit, minus five hundred, invalid entry. Amount must be positive." The trailing error text arrives because `aria-invalid="true"` combined with `aria-describedby` causes NVDA to read the associated message automatically.

What happens if the error text is not associated? The reader says "Amount, edit, minus five hundred" and then silence. The user tabs forward thinking the value was accepted. When they eventually hit Submit, a banner fires "Form has errors" -- with no indication of *which* field. They are now hunting blindly.

**Step 4 -- The confirm dialog.** The user fills the form, presses Submit, and a confirm dialog from [Chapter 47](ch12-directives-templates.md) appears: "Are you sure you want to debit $500 from Checking?" Test two behaviors:

1. **Focus trap.** Tab cycles between "Cancel" and "Confirm" without escaping.
2. **Escape closes and restores.** Pressing `Escape` dismisses the dialog and restores focus to the Submit button, so the user can try again or correct the form. Both are handled by `cdkTrapFocus` and the dialog's own key handler.

**Step 5 -- Success.** After confirming, a toast announces "Transaction saved successfully" via `LiveAnnouncer.announce(..., 'polite')`. Focus does not move -- the user stays in the transaction form, ready to add another. A polite announcement waits for the reader to finish its current utterance; an assertive one interrupts, which is the right choice for errors but wrong for successes.

Run this flow once a sprint. Each of these steps corresponds to a specific ARIA or focus-management decision. Breaking any one of them turns the flow from ten seconds into two minutes of confusion.

### Walkthrough: Data-Heavy Pages (Portfolio with Charts)

The portfolio dashboard is the hardest part of FinancialApp to make accessible. Charts, dense tables, and summary cards stack on a single page. Here is how to make them usable without oversimplifying.

**Tables with real table semantics.** Screen readers have mature support for `<table>` with proper `<thead>`, `<tbody>`, `<th scope="col">`, and `<th scope="row">`. The reader announces "column Ticker, row Apple" when the user is on a data cell, so they always know where they are. `role="grid"` layers keyboard navigation on top of the same structure -- useful for editable grids, but for a read-only holdings list a plain `<table>` is clearer. Avoid `<div>` soup styled to look like a table; readers cannot infer column-row relationships from CSS.

```html
<!-- financial-app/apps/financial-app/src/app/features/portfolio/holdings-table.component.html -->
<table>
  <caption>Current holdings</caption>
  <thead>
    <tr>
      <th scope="col">Ticker</th>
      <th scope="col">Shares</th>
      <th scope="col">Value</th>
    </tr>
  </thead>
  <tbody>
    @for (h of holdings(); track h.id) {
      <tr>
        <th scope="row">{{ h.ticker }}</th>
        <td>{{ h.shares }}</td>
        <td>{{ h.currentValue | currency }}</td>
      </tr>
    }
  </tbody>
</table>
```

The `<caption>` gives the table a name; `scope="row"` on the ticker column anchors each row. A user who arrow-navigates into a random cell hears "Apple, Value, 28500 dollars" -- full context from any cell.

**Text alternatives for charts.** A donut chart from [Chapter 25](ch30-charts.md) is purely visual. The SVG has no intrinsic meaning to a reader. Two fixes, used together: add `role="img"` and `aria-label` to the chart host for a summary sentence, and provide the raw data as a visually-hidden table that the reader can explore:

```html
<div role="img"
     [attr.aria-label]="'Portfolio allocation: ' + summaryText()">
  <apx-chart [options]="allocationOptions()" />
</div>
<table class="visually-hidden">
  <caption>Portfolio allocation details</caption>
  <thead>
    <tr><th scope="col">Asset class</th><th scope="col">Percentage</th></tr>
  </thead>
  <tbody>
    @for (slice of allocation(); track slice.label) {
      <tr><th scope="row">{{ slice.label }}</th><td>{{ slice.percent }}%</td></tr>
    }
  </tbody>
</table>
```

The `.visually-hidden` utility class uses the standard clip-rect technique -- off-screen positioning plus `clip` -- so the content is in the DOM for screen readers but invisible to sighted users:

```scss
// financial-app/libs/shared/ui/src/styles/_a11y.scss
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

Never use `display: none` or `visibility: hidden` for this -- both remove the element from the accessibility tree. A chart that says "Portfolio allocation: 45% equities, 30% bonds, 25% cash" plus an explorable table is more informative than the chart ever was visually.

**Skip links.** Dense pages have many landmarks; tab order is long. A skip link at the top of the page -- visible only when focused -- lets keyboard users jump past the navigation into the main content in one keystroke:

```html
<a class="skip-link" href="#main-content">Skip to main content</a>
<nav>...</nav>
<main id="main-content" tabindex="-1">
  <!-- ... -->
</main>
```

Style `.skip-link` to be off-screen until focused, then brought on-screen at top-left with high contrast. A sighted keyboard user sees it when they Tab; screen-reader users hear "Skip to main content, link" as the first interactive element. Either way, they bypass thirty nav items in one action.

### Common Patterns Revisited with Real Screen-Reader Impact

The ARIA attributes covered earlier have audible consequences. Knowing *what* each sounds like tells you when to reach for which.

**`aria-live="polite"`** queues an announcement until the reader finishes speaking. Use it for transient updates that the user does not need to know about *right now*: a background sync completing, a search result count updating, a toast confirming a save. The reader finishes its current utterance, then says the new one. The user is not interrupted.

**`aria-live="assertive"`** interrupts immediately. Use it sparingly -- error messages, critical state changes, session timeouts. An assertive announcement that fires during the user's own typing will cut them off mid-word, which is worse than useless. The `LiveAnnouncer.announce(msg, 'assertive')` covered earlier wraps this safely.

**`aria-live="off"`** is the default and means "do not announce changes." This is what most of your DOM should be.

**`aria-busy="true"`** during loading suppresses repeat announcements of the region's contents while it is being updated. A deferred holdings table that re-renders three times during load should be wrapped in `aria-busy`, which suppresses the intermediate announcements. When `aria-busy` flips to `false`, the final content announces once.

**`aria-expanded` and `aria-controls`** power disclosure patterns. A "Show transaction details" button with `aria-expanded="false"` `aria-controls="txn-detail-42"` announces as "Show transaction details, collapsed, button." After activation: "expanded, button." The user knows the state changed without scrolling to find the newly-revealed content.

**`role="status"`** is a less-disruptive form of `role="alert"`. An inline "Saved" indicator next to a field gets `role="status"`; it announces politely. A form-wide "Payment failed" banner gets `role="alert"`; it announces assertively. Use the role that matches the urgency.

### Focus Management in SPA Navigation (Continued)

The `RouteFocusService` introduced earlier handles the simple case; two corner cases deserve attention.

**Sub-route transitions.** A user on `/accounts/acct-001` navigates to `/accounts/acct-001/transactions`. Does focus move? If the sub-route renders inside a `<router-outlet>` that sits below an unchanged heading, moving focus to the same `<h1>` is jarring -- the user hears the same announcement twice. The refinement: move focus to the `<h2>` of the newly-rendered sub-route, or if there is none, to the outlet itself with `tabindex="-1"`. The rule is "focus the most specific new heading on the page."

**Focus restoration after modal close.** When a modal dialog closes, focus should return to the element that opened it. `cdkTrapFocusAutoCapture` does this automatically for the common case where the opener is a button. But if the opener was inside a list that re-rendered while the modal was open -- the opener no longer exists in the DOM -- restoration falls back to `<body>`. The mitigation: capture the opener's semantic identity (its text content or data-id) when opening, and after close, locate an equivalent element with a matching identity and focus it. In practice, stable element IDs and `track` functions on `@for` loops solve most cases before they manifest.

An enhanced `RouteFocusService` that handles both cases:

```typescript
// financial-app/apps/financial-app/src/app/core/a11y/route-focus.service.ts
import { Injectable, inject } from '@angular/core';
import { Router, NavigationEnd, ActivationEnd } from '@angular/router';
import { filter, pairwise, startWith } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class RouteFocusService {
  private router = inject(Router);

  initialize(): void {
    this.router.events
      .pipe(
        filter((e): e is NavigationEnd => e instanceof NavigationEnd),
        startWith(null),
        pairwise(),
      )
      .subscribe(([prev, curr]) => {
        if (!curr) return;
        const isSubRoute = this.isSubRouteTransition(prev?.url, curr.url);
        const target = this.findFocusTarget(isSubRoute);
        if (target) this.focusProgrammatically(target);
      });
  }

  private isSubRouteTransition(from: string | undefined, to: string): boolean {
    if (!from) return false;
    const fromBase = from.split('/').slice(0, 3).join('/');
    const toBase = to.split('/').slice(0, 3).join('/');
    return fromBase === toBase;
  }

  private findFocusTarget(isSubRoute: boolean): HTMLElement | null {
    if (isSubRoute) {
      return (
        document.querySelector<HTMLElement>('main h2') ??
        document.querySelector<HTMLElement>('main')
      );
    }
    return document.querySelector<HTMLElement>('main h1');
  }

  private focusProgrammatically(el: HTMLElement): void {
    el.setAttribute('tabindex', '-1');
    el.focus();
    el.addEventListener(
      'blur',
      () => el.removeAttribute('tabindex'),
      { once: true },
    );
  }
}
```

The `pairwise` operator gives us both the previous and current URL, so we can distinguish "full navigation" from "sub-route change." The former moves focus to `<h1>`; the latter to `<h2>` if one exists, or the `<main>` element as a fallback. Screen reader users now hear the most specific orienting heading on every navigation, never the same announcement twice.

---

## Cognitive Accessibility

Screen readers address perceptual barriers. Cognitive accessibility addresses comprehension. A user with dyslexia, a user whose first language is not English, a user who is tired or stressed -- all benefit from the same things.

**Plain language.** Financial applications drown in jargon. "Available ledger position" is a phrase an operations analyst understands; "balance available to spend" is what a customer understands. Replace each term with its concrete equivalent. Reserve the technical term for advanced contexts where it is genuinely more precise. A tooltip can offer both: primary label "Balance," hover text "Technical term: available ledger position."

**Error recovery.** Never show "Error" with no context. Every error message must tell the user: (1) what went wrong, (2) what they can do about it, (3) where to go for help if they cannot. "Transfer failed" is a bad message. "Transfer failed: insufficient funds in checking account. Try a smaller amount or transfer from savings" is a good one. This applies equally to sighted users and screen reader users -- the screen reader just makes vague errors more painful, because there is no visual affordance to fall back on.

**Consistent placement.** Navigation belongs in the same place on every page. The primary action belongs in the same place in every form. The "Cancel" button goes on the same side of every dialog. Users build muscle memory around layout; relocating the logout link between pages is a cognitive tax on everyone, and a navigation emergency for someone with memory or attention difficulties.

**Low-literacy and second-language users.** Keep sentences short. Use the active voice. Prefer nouns over gerunds ("Transfer" over "Transferring"). Avoid negatives wherever possible -- "You have 0 unread messages" is clearer than "You do not have any unread messages." Reading ease is a skill; the Hemingway score is a useful proxy, but the simplest heuristic is "would a tired person understand this on the first read?"

---

## Mobile Accessibility

Mobile assistive technology differs from desktop in ways that matter. [Chapter 44](ch41-capacitor-mobile.md) covers Capacitor integration; the accessibility implications are worth naming here.

**Touch targets.** WCAG 2.5.5 (Level AAA) requires touch targets to be at least 44x44 CSS pixels. WCAG 2.5.8 (Level AA in 2.2) relaxes this to 24x24 with spacing requirements. Either way, the 32x32 icon buttons that look fine on desktop are too small on a phone, especially for users with motor impairments. The minimum across FinancialApp is 44x44, with `padding` making the hit area larger than the visual icon where necessary.

**Gestures.** Mobile screen readers (TalkBack, VoiceOver on iOS) remap standard gestures. A one-finger swipe moves to the next element; a two-finger swipe scrolls. Double-tap activates the focused element; a three-finger swipe changes pages. The upshot: never rely on custom gestures that conflict with the reader's vocabulary. A "swipe to dismiss" toast should always have an equivalent close button, because swipe-to-dismiss means something else under TalkBack.

**Zoom and reflow.** WCAG 1.4.10 requires content to reflow at 320 CSS pixel width with no horizontal scrolling, except for content that genuinely requires two-dimensional layout (data tables, maps). Test this explicitly: open DevTools device emulation at 320px and verify that every page reads top-to-bottom without horizontal scroll. A fixed-width dashboard that demands 1200px is unusable at mobile sizes even with zoom. The responsive design techniques from Chapter 32 address this, but only if you test at the boundary.

---

## Accessibility Review Checklist for Pull Requests

A short, blunt checklist belongs in your pull-request template. Every reviewer asks these questions about every change that touches UI:

1. **Can I use the feature with keyboard alone?** Tab to it, activate it with Enter/Space, navigate its contents with arrows, dismiss it with Escape where applicable. If mouse is the only path, the feature is incomplete.
2. **Does every interactive element have a visible focus indicator?** Open DevTools, remove the default outline, and verify that your replacement is still obvious. A focus ring you cannot see from two feet away is a failed focus ring.
3. **Does my form error get announced?** `aria-describedby` on the input, `role="alert"` or `LiveAnnouncer` on form-wide errors, focus restored to the first invalid field on submit.
4. **Do I pass `axe-core`?** The helper covered earlier or the Playwright fixture below.
5. **Did I test with at least one screen reader end-to-end?** VoiceOver on Safari or NVDA on Firefox, all the way through the happy path and at least one error path. Attach a short description of what you heard to the PR.

These five questions will not catch every issue, but they catch the ones that make applications unusable. Treat them with the same seriousness as test coverage and bundle size.

---

## Playwright a11y Fixture for Regression

Manual screen-reader testing is slow and irreplaceable. Automated audits catch regressions on everything else. A Playwright fixture that runs `@axe-core/playwright` against every feature route is the backbone of continuous a11y coverage.

```typescript
// financial-app/libs/shared/testing/a11y-fixture.ts
import { test as base, expect, type Page } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

export interface A11yFixtures {
  auditPage: (options?: AuditOptions) => Promise<void>;
}

interface AuditOptions {
  /** Exclude subtrees (e.g., third-party widgets you cannot fix). */
  exclude?: string[];
  /** WCAG tags to enforce. Defaults to WCAG 2.1 AA. */
  tags?: string[];
  /** Minimum impact level that fails the test. */
  failOn?: Array<'minor' | 'moderate' | 'serious' | 'critical'>;
}

export const test = base.extend<A11yFixtures>({
  auditPage: async ({ page }, use) => {
    await use(async (options = {}) => {
      const {
        exclude = [],
        tags = ['wcag2a', 'wcag2aa', 'wcag21aa'],
        failOn = ['serious', 'critical'],
      } = options;

      let builder = new AxeBuilder({ page }).withTags(tags);
      for (const selector of exclude) {
        builder = builder.exclude(selector);
      }

      const { violations } = await builder.analyze();
      const blocking = violations.filter((v) =>
        failOn.includes(v.impact as (typeof failOn)[number]),
      );

      expect(
        blocking,
        `A11y violations on ${page.url()}:\n` +
          blocking
            .map((v) => `  [${v.impact}] ${v.id}: ${v.help}`)
            .join('\n'),
      ).toEqual([]);
    });
  },
});

export { expect };
```

Two deliberate choices here. **Only serious and critical violations fail the test.** `axe-core` reports many "moderate" and "minor" violations that represent nice-to-haves rather than blockers. Starting strict and relaxing is harder than starting lenient and tightening; begin with critical-only, add serious once you are clean there, then tackle moderate. **The failure message names the page URL and every blocking violation.** Generic "expected [] to equal [Array]" output is useless during triage; listing the impact, rule ID, and help text lets a reviewer fix issues without reading the raw `axe-core` report.

Use the fixture in a sweep that visits every major route:

```typescript
// apps/financial-app-e2e/src/tests/a11y-sweep.spec.ts
import { test, expect } from '@financial-app/shared/testing/a11y-fixture';

const ROUTES = [
  '/login',
  '/dashboard',
  '/accounts',
  '/accounts/acct-001',
  '/accounts/acct-001/transactions',
  '/portfolio',
  '/clients',
  '/clients/new',
  '/settings',
];

for (const route of ROUTES) {
  test(`no critical a11y violations on ${route}`, async ({
    page,
    auditPage,
  }) => {
    await page.goto(route);
    await page.waitForLoadState('networkidle');
    await auditPage({
      exclude: ['.third-party-widget'],
    });
  });
}
```

Nine routes, one test each, three seconds apiece. Thirty seconds of CI time buys regression coverage on every rule `axe-core` can detect across the entire application. Extend the list when you ship a new feature route; the test is so cheap that there is no reason not to.

Pair the sweep with focus-state audits. A separate spec opens each dialog, each autocomplete, each tooltip, and asserts that no new violations appear when the interactive element is active -- those states are often the source of contrast and focus-indicator regressions that a static sweep misses.

Wire the sweep into the existing GitHub Actions pipeline from [Chapter 19](ch37-e2e-playwright.md):

```yaml
# .github/workflows/e2e.yml (added job)
  a11y-sweep:
    runs-on: ubuntu-latest
    needs: e2e
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npx nx build financial-app --configuration=production
      - name: Run a11y sweep
        run: npx playwright test a11y-sweep --project=chromium
        working-directory: apps/financial-app-e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: a11y-report
          path: apps/financial-app-e2e/playwright-report/
```

Only Chromium is needed for the sweep -- `axe-core` rules are browser-agnostic, so running on three engines triples the cost without catching additional issues. The job runs after the main E2E job succeeds; if the app is broken, the a11y sweep would fail for unrelated reasons and obscure the root cause.

---

## Accessibility Governance & VPAT

Automated audits and manual walkthroughs surface problems; governance keeps them from returning once fixed, and proves to regulators and procurement officers that accessibility is a discipline rather than a one-time cleanup.

A **VPAT** -- Voluntary Product Accessibility Template -- is the industry-standard document declaring how a product conforms to WCAG, Section 508, and EN 301 549. For a public-sector or enterprise sale it is often required as part of procurement; for a regulated financial application it is evidence for compliance audits. Produce one before each major release. The template walks through every WCAG criterion with "Supports," "Partially Supports," "Does Not Support," or "Not Applicable," plus a short remark. The remark is where the manual walkthrough matters -- a criterion can pass `axe-core` and still earn "Partially Supports" because NVDA announces an error badly or because the chart text alternative is a terse sentence rather than the explorable table it should be.

Tie the automated and manual findings together in one log. Export the `@axe-core/playwright` JSON from CI, import the audit-tool report from the latest manual pass, and track each violation as a ticket in the team's backlog with severity, WCAG criterion, and owning team. The "Partially Supports" notes in the VPAT cite those ticket IDs, so the document stays honest and the team knows what to fix next. Published financial-services compliance obligations are covered in [Chapter 16](ch42-compliance.md); the VPAT is the accessibility-shaped artifact that slots into that broader regime.

Retest on a cadence. Every feature revisits its walkthroughs before release. Every chapter-covered pattern (ARIA binding, focus management, live regions) gets a regression sweep quarterly. The Playwright fixture runs on every PR. Accessibility decays the moment nobody is watching; ritualized retesting is how you catch the decay before users do.

---

## Summary

- **ARIA binding** uses `[attr.aria-*]` for dynamic values, plain attributes for static ones, `[ariaLabelledByElements]` for element references, and the `host` object for component-level semantics.
- **Progress bars, `@defer` blocks, and live regions** require `role="progressbar"` value attributes on the host and `aria-live="polite"` wrappers so screen readers announce loaded content.
- **Focus management** after route navigation prevents screen reader context loss; `ariaCurrentWhenActive="page"` supplies the non-visual equivalent of a highlighted nav link.
- **`@angular/aria`** implements WAI-ARIA keyboard and attribute contracts (Toolbar, Tabs, Combobox) without imposing visual design; **`@angular/cdk/a11y`** provides `LiveAnnouncer`, `cdkTrapFocus`, and `FocusMonitor`.
- **`axe-core` catches what a parser can see.** Everything else -- whether a workflow is actually usable -- requires real assistive technology. Both matter.
- **Get at least two screen readers running.** VoiceOver on macOS, NVDA on Windows. JAWS if you ship to enterprise. Chrome DevTools Accessibility panel for quick inspection while writing components.
- **Learn the minimum shortcuts.** Rotor, heading navigation, read-from-cursor, silence. Enough to move like a user, not a tutorial.
- **Walk through flows end-to-end.** Login, transaction submission, portfolio dashboard. Each surfaces different patterns: route focus, inline errors, table semantics, chart alternatives.
- **ARIA attributes have audible consequences.** `polite` vs `assertive`, `aria-busy`, `aria-expanded`, `role="status"` -- pick the one that matches what you want the user to hear, not the one that silences the lint rule.
- **Cognitive accessibility is a separate discipline.** Plain language, actionable errors, consistent placement, short sentences. Benefits everyone.
- **Mobile a11y differs.** 44x44 touch targets, no custom gestures, reflow at 320px. Tested in conjunction with the Capacitor build from [Chapter 44](ch41-capacitor-mobile.md).
- **A pull-request checklist of five questions** catches the majority of regressions before they ship.
- **A Playwright fixture running `@axe-core/playwright` across every route** gives you continuous regression coverage at CI-acceptable cost. Built on the E2E foundation from [Chapter 19](ch37-e2e-playwright.md).
- **Governance keeps accessibility from decaying.** Publish a VPAT per major release, link automated and manual findings in a shared backlog, and retest on a cadence.

WCAG AA is the floor. The ceiling is an application that disabled users reach for by choice, not because it is the only one their employer approves. Accessible patterns, end-to-end verification with real assistive technology, and ongoing governance are how you get there.

> **Companion code:** `@angular/aria` and `@angular/cdk` are installed in `financial-app/`. Accessible nav toolbar and account-type tabs live in `libs/shared/ui/`. The `axe-core` test helper is in `libs/shared/ui/src/testing/`, and the Playwright a11y fixture that audits every feature route is in `libs/shared/testing/a11y-fixture.ts`.
