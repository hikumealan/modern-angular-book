# Chapter 36: Angular Elements

Sometimes Angular is the right tool for a widget, but the host page is not. FinancialApp's product team strikes a distribution deal with a partner bank: starting next quarter, the partner will embed FinancialApp's account-summary widget directly on their landing page so that signed-in customers can see their most recent balances without clicking through to a separate portal. The widget logic -- account formatting, currency conversion, pending-transaction indicators -- already exists in FinancialApp's Angular codebase, battle-tested and accessible. The partner's landing page is a server-rendered Thymeleaf template, and their app shell is React.

You could rewrite the widget in React. You could hand the partner a design spec and ask them to re-implement it. Both options discard years of engineering investment and guarantee that the two implementations will drift apart within six months. A better option is to ship the existing Angular component as a **Custom Element** -- a framework-agnostic web standard that any HTML page can consume.

This chapter shows how to package an Angular component as a distributable custom element with `@angular/elements`, how inputs and outputs map across the Angular/DOM boundary, how to choose between Shadow DOM and light DOM, how to bundle the result as a single file for CDN embedding, and how FinancialApp publishes `<fin-account-summary>` to its partner network. Angular Elements is the low-coordination sibling of the micro-frontend patterns in [Chapter 30](ch38-micro-frontends.md): where federation integrates full feature areas between Angular applications, Elements ships individual widgets to foreign hosts.

> **Companion code:** `financial-app/apps/financial-widget/` builds `<fin-account-summary>` as a single-file distributable.

---

## What a Custom Element Is

A custom element is a web standard, not an Angular concept. It has three moving parts:

1. **Registration.** A JavaScript class extending `HTMLElement` is registered with the browser via `customElements.define('fin-account-summary', class { ... })`. From that moment, any `<fin-account-summary>` tag the browser encounters -- in static HTML, in React JSX, in a server-rendered template -- is upgraded into an instance of that class.
2. **Lifecycle callbacks.** The browser invokes `connectedCallback()`, `disconnectedCallback()`, and `attributeChangedCallback()` on the class as the element enters the DOM, leaves it, or has an attribute change. These are the hooks the element uses to initialize, clean up, and react to attribute updates.
3. **Shadow DOM (optional).** The element can attach a shadow root to isolate its internal DOM and styles from the host page. Shadow DOM is not required for custom elements, but it is the mechanism that makes them feel like sealed components rather than chunks of unmanaged markup.

Angular's `@angular/elements` package takes care of all three parts automatically. You hand it an Angular component; it produces a class that registers with `customElements`, wires the component's inputs to attributes and properties, and dispatches the component's outputs as DOM `CustomEvent`s. The Angular component inside is unchanged -- it has no idea it is running as a custom element. The same template, the same services, the same signals.

Custom Elements have been baseline in every evergreen browser since 2018. Today they are a safe, standards-compliant distribution format with no runtime dependencies beyond the browser itself.

---

## Setting Up `@angular/elements`

FinancialApp already lives in an Nx monorepo (see [Chapter 14](ch14-monorepos-libraries.md)). The widget becomes a new application alongside the main app -- a separate build target with its own bundle, sharing the same domain libraries.

### Installing the Package

```bash
npm install @angular/elements
npx nx g @nx/angular:app financial-widget \
  --style=scss \
  --routing=false \
  --standalone \
  --ssr=false
```

The generated app is a standard Angular application at first. We strip out the parts that do not apply to a custom element -- the router, the top-level `AppComponent`, the `index.html` -- because the widget never owns a page. It is a guest on someone else's page.

### Creating the Widget Component

The widget itself is an ordinary Angular component with signal inputs and outputs:

```typescript
// apps/financial-widget/src/app/account-summary.component.ts
import {
  Component,
  ChangeDetectionStrategy,
  ViewEncapsulation,
  input,
  output,
  computed,
} from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { AccountSummaryData } from '@financial-app/shared/models';

@Component({
  selector: 'fin-account-summary-inner',
  standalone: true,
  imports: [CurrencyPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  encapsulation: ViewEncapsulation.ShadowDom,
  template: `
    <section
      class="fin-summary"
      role="region"
      [attr.aria-label]="'Account ' + account().name"
    >
      <header>
        <h3>{{ account().name }}</h3>
        <span class="type">{{ account().type }}</span>
      </header>
      <p class="balance">{{ account().balance | currency: account().currency }}</p>
      @if (account().pendingCount > 0) {
        <button type="button" (click)="viewPending.emit(account().id)">
          {{ account().pendingCount }} pending
        </button>
      }
    </section>
  `,
  styleUrl: './account-summary.component.scss',
})
export class AccountSummaryComponent {
  readonly account = input.required<AccountSummaryData>();
  readonly viewPending = output<number>();

  readonly isLowBalance = computed(() => this.account().balance < 100);
}
```

Nothing about this component signals that it will ship as a custom element. It uses `input()` signals, a typed `output()`, `OnPush` change detection, and standalone imports -- the same patterns described in earlier chapters. The custom element shell will be added at bootstrap.

### Bootstrapping as a Custom Element

The `main.ts` for an Elements-only build looks different from a normal Angular app. There is no `bootstrapApplication`, no root component rendered into an `<app-root>` tag. Instead, we construct an `ApplicationRef` purely to serve as an injector context, then register the component with `customElements`:

```typescript
// apps/financial-widget/src/main.ts
import { createApplication } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { provideZonelessChangeDetection } from '@angular/core';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { AccountSummaryComponent } from './app/account-summary.component';

async function bootstrap() {
  const app = await createApplication({
    providers: [
      provideZonelessChangeDetection(),
      provideHttpClient(withFetch()),
    ],
  });

  const SummaryElement = createCustomElement(AccountSummaryComponent, {
    injector: app.injector,
  });

  customElements.define('fin-account-summary', SummaryElement);
}

bootstrap();
```

`createApplication` is the key primitive: it returns a standalone `ApplicationRef` with no root component attached. We pull its injector out and hand it to `createCustomElement`. Every instance of `<fin-account-summary>` the browser encounters will use that injector to resolve dependencies -- `HttpClient`, signal stores, anything else the widget's component tree needs.

There is no `index.html`. The output of this build is a single JavaScript file that, when loaded by a host page, registers the element with the browser. The host page never imports an Angular module.

---

## Input and Output Mapping

`createCustomElement` is where Angular semantics meet DOM semantics. The component's inputs and outputs get translated into the conventions that custom element consumers expect.

### Inputs Become Both Attributes and Properties

For every signal input on the component, the generated custom element class exposes two entry points:

1. A **DOM property** on the element instance. You can set `element.account = { ... }` from JavaScript.
2. An **HTML attribute** with the kebab-case form of the input name. You can write `<fin-account-summary account-id="42">` in markup.

Signal inputs participate in this bridge just like the older `@Input()` decorator did. `input.required<AccountSummaryData>()` becomes a setter that triggers the Angular change-detection lifecycle internally whenever the property or attribute changes.

Type coercion matters here. **HTML attributes are always strings.** If the partner writes:

```html
<fin-account-summary account-id="42"></fin-account-summary>
```

the input receives the string `"42"`, not the number `42`. For primitives this is usually fine -- a template literal coerces cleanly. For objects and arrays it is not fine at all, because an attribute cannot carry a structured value. The host page must set complex inputs as properties:

```javascript
const el = document.querySelector('fin-account-summary');
el.account = {
  id: 42,
  name: 'Primary Checking',
  type: 'checking',
  balance: 1423.55,
  currency: 'USD',
  pendingCount: 2,
};
```

React's JSX transpiles known props to property assignments automatically when the element is defined, so `<fin-account-summary account={account} />` works as expected. Vue 3 handles this through its `:account.prop` binding modifier. Server-rendered templates need a small inline script to do the property assignment after render.

FinancialApp's widget accepts both shapes for convenience. A small number of scalar inputs (`currency`, `locale`) can come in through attributes so that static HTML pages work. The rich account object must come in as a property.

### Outputs Become `CustomEvent`s

Angular outputs are mapped to DOM `CustomEvent` dispatches. The `viewPending` output above fires a `viewPending` event on the host element, with the emitted value tucked into `event.detail`:

```javascript
const el = document.querySelector('fin-account-summary');
el.addEventListener('viewPending', (event) => {
  const accountId = event.detail; // the number emitted from Angular
  openPendingDrawer(accountId);
});
```

Event names are case-sensitive and are **not** converted to kebab-case. An output called `viewPending` fires an event named `viewPending`, which React writes as `onViewPending` and Vue as `@view-pending`. Document this clearly in the widget's README -- partners will look for the exact event names.

Outputs should carry plain-data detail payloads. Avoid emitting live Angular entities, class instances, or anything with cyclic references. The consumer might try to serialize or log the event, and obscure DOM errors are a poor welcome to your widget.

---

## Shadow DOM vs Light DOM

Every custom element is rendered into a host element in the host page. The question is whether the element's internal DOM is isolated from that host page or not. Angular's `ViewEncapsulation` setting maps directly onto this decision.

**`ViewEncapsulation.ShadowDom`** attaches a real shadow root to the host element. Styles defined inside the component cannot leak out; styles defined on the host page cannot leak in. This is exactly what you want when embedding on a foreign page whose CSS you do not control -- the partner bank's `.button { color: red; }` rule will not accidentally turn your submit button red.

**`ViewEncapsulation.None`** renders into the light DOM and applies styles globally. The widget inherits the host page's CSS, which is sometimes a feature (the widget blends with the host's typography) and sometimes a bug (the host's `* { box-sizing: content-box; }` breaks the widget's layout).

**`ViewEncapsulation.Emulated`** (the default) does not work the way you might expect inside a custom element. The scoped attribute selectors still apply, but without a shadow root the host page's styles still reach into the component tree. You get the downsides of both modes with few of the benefits.

For FinancialApp's embeddable widget, Shadow DOM is the right choice. The partner bank's design language should stay in the partner bank's UI; the widget is a self-contained guest that renders its own view with its own styles. For widgets that live inside a company's own pages -- a marketing dashboard, an internal admin tool -- light DOM may be preferable so that the widget inherits the site-wide typography and color scheme.

The cost of Shadow DOM is that theme inheritance becomes explicit. CSS custom properties (`--fin-color-primary`) cross the shadow boundary; regular CSS selectors do not. That makes tokens, not classes, the contract between the host page and the widget -- which is the next section's topic.

---

## Styling Strategies

The widget has to look coherent on a partner page you have never seen. FinancialApp chose a two-layer approach.

**Layer 1: a complete default theme shipped inside the bundle.** The widget's SCSS compiles into the shadow root with full default values for every token. If the partner does nothing, the widget renders in FinancialApp's brand colors, typography, and spacing -- the same design tokens defined in [Chapter 32](ch22-material-design-system.md). This is the guarantee: the widget is always visually complete on its own.

**Layer 2: overridable CSS custom properties pierce the shadow boundary.** Partners who want the widget to match their own theme override a small set of documented tokens on the host element:

```html
<style>
  fin-account-summary {
    --fin-color-primary: #0b3d91;
    --fin-color-surface: #ffffff;
    --fin-font-family-brand: 'PartnerSans', sans-serif;
    --fin-radius-md: 4px;
  }
</style>

<fin-account-summary id="summary"></fin-account-summary>
```

Inside the widget, every style references the tokens rather than hardcoded values:

```scss
.fin-summary {
  background: var(--fin-color-surface);
  color: var(--fin-color-on-surface);
  border-radius: var(--fin-radius-md);
  font-family: var(--fin-font-family-brand, system-ui, sans-serif);
  padding: var(--fin-space-md);
}
```

The fallback value in `var(..., system-ui)` is important: if a partner accidentally removes a token, the widget still renders with something reasonable rather than a blank page. Every documented token has a fallback.

The full token set is intentionally small -- eight color tokens, four spacing tokens, two radius tokens, one font family. A large token contract means the partner has to think hard to theme the widget; a small, well-chosen one means they can usually ship a branded widget by overriding three or four values. Resist the urge to expose every design decision as a token.

---

## Bundling as a Single File

For CDN distribution, a single-file bundle is the right deliverable. The partner embeds one `<script>` tag and has a new HTML tag available.

The Nx application's production build config for the widget is tailored for this:

```json
// apps/financial-widget/project.json (excerpt)
{
  "targets": {
    "build": {
      "executor": "@angular-devkit/build-angular:application",
      "options": {
        "outputPath": "dist/fin-widgets",
        "index": false,
        "browser": "apps/financial-widget/src/main.ts",
        "tsConfig": "apps/financial-widget/tsconfig.app.json",
        "inlineStyleLanguage": "scss",
        "outputHashing": "none"
      },
      "configurations": {
        "production": {
          "optimization": true,
          "extractLicenses": true,
          "sourceMap": false
        }
      }
    }
  }
}
```

`"index": false` tells the builder not to emit an `index.html` -- there is no page to render, only a bundle to ship. `"outputHashing": "none"` keeps the filename stable at `main.js` so that the publishing pipeline can rename it to `fin-widgets.v1.js` predictably.

**Tree-shaking considerations.** The widget bundle pulls in Angular's runtime, the parts of `@angular/common` the component uses, and any shared libraries the component imports. Audit carefully: a stray import of a heavy library from `libs/shared/ui` can add hundreds of kilobytes to a widget that should be lean. The `@financial-app/shared/models` library is fine -- it is interface-only and tree-shakes to nothing. `@financial-app/shared/ui` components are fine if the widget actually uses them. An accidental import from `@financial-app/shared/charts` is a problem.

**Polyfills.** Custom Elements are baseline in every evergreen browser, so the widget ships without a Custom Elements polyfill. `ElementInternals` -- the API that lets custom elements participate in form submission and accessibility trees -- is more recent and still worth checking against the widget's browser matrix. If you need `ElementInternals` on a browser that does not support it, the `element-internals-polyfill` package provides coverage.

**Target output.** FinancialApp's widget bundle is around 180 KB gzipped, which is the cost of embedding Angular's runtime. For a single widget on a page, this is fine. For multiple Angular-based widgets on the same page, each shipping its own runtime, consider Native Federation (see [Chapter 30](ch38-micro-frontends.md)) instead of independent Elements bundles.

---

## Bootstrapping for Element-Only Builds

The `main.ts` shown earlier is worth dwelling on. Unlike a normal Angular application, an Elements-only build has no `AppComponent` -- the whole notion of a single root component does not apply when the output is a library of registered tags rather than a page.

`createApplication` is purpose-built for this:

```typescript
import { createApplication } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { AccountSummaryComponent } from './app/account-summary.component';
import { TransactionListComponent } from './app/transaction-list.component';
import { widgetProviders } from './app/widget.providers';

async function registerWidgets() {
  const app = await createApplication({ providers: widgetProviders });

  const widgets: Array<[string, any]> = [
    ['fin-account-summary', AccountSummaryComponent],
    ['fin-transaction-list', TransactionListComponent],
  ];

  for (const [tag, component] of widgets) {
    if (customElements.get(tag)) continue;
    const element = createCustomElement(component, { injector: app.injector });
    customElements.define(tag, element);
  }
}

registerWidgets().catch((err) => console.error('[fin-widgets] bootstrap failed', err));
```

A single `ApplicationRef` shared across multiple widgets keeps the injector consistent: all widgets share the same `HttpClient`, the same auth context, the same configuration. If the same bundle might be loaded twice -- for example, by two independent scripts on a CMS page -- the `customElements.get(tag)` guard prevents a redefinition error. The browser throws if you call `define` twice with the same tag.

A shared `widgetProviders` array bundles the common providers -- HTTP, zoneless change detection (see [Chapter 2](ch02-signal-components.md)), error handlers, an API base URL token -- into one place so that every widget in the bundle gets the same configuration.

---

## FinancialApp Use Cases

Three distribution scenarios justify the widget approach for FinancialApp.

**Partner portal embedding.** The driving use case. A bank's landing page includes:

```html
<head>
  <script src="https://cdn.financialapp.com/widgets/v1/fin-widgets.js" defer></script>
</head>
<body>
  <!-- partner's React app renders below -->
  <section class="account-sidebar">
    <fin-account-summary id="primary-account"></fin-account-summary>
  </section>
</body>
```

followed by a small bridge script that sets the account data from whatever the partner's React state provides. From the partner's perspective, FinancialApp is a single script tag and a custom HTML element. From FinancialApp's perspective, the widget is an ordinary Angular component that happens to build into a bundle instead of a page.

**Legacy AngularJS pages during migration.** Some FinancialApp pages are still running on AngularJS (1.x) and are slowly being rewritten. Rather than block a feature on finishing the migration, the team ships the new feature as a custom element and embeds it in the AngularJS template. The AngularJS controller passes data via attributes; the new widget dispatches `CustomEvent`s that the AngularJS digest cycle picks up through event listeners. The widget carries the migrated code into production years before the surrounding page catches up.

**Server-rendered templates with one interactive island.** Marketing pages are statically generated from Markdown. Most of the page is just HTML, but the "Try our portfolio calculator" section needs real interactivity. A custom element is the simplest integration: the static generator emits `<fin-portfolio-calculator>` in the HTML, and the page loads the widget bundle lazily when the element scrolls into view. The rest of the page stays static and cacheable; only the interactive island pays the Angular tax.

---

## Accessibility

Elements retain every accessibility property of the Angular component inside them. The semantic HTML rendered by the template -- the `<section role="region">`, the `aria-label`, the focusable `<button>` -- survives the trip into the shadow root unchanged. Screen readers announce the widget's contents the same way they would announce it on a native Angular page. See [Chapter 8](ch24-accessibility.md) for the patterns the widget uses internally.

Shadow DOM introduces one subtlety: **focus crosses shadow boundaries by default, but focus delegation does not.** If the host page applies `tabindex="0"` to the widget's host element and the user tabs into it, focus enters the shadow root and lands on the first focusable descendant -- exactly what you want. But if the user clicks on the host element's padding, focus lands on the host element itself rather than the first focusable child, which is rarely what you want.

The fix is to enable focus delegation when attaching the shadow root. Angular's `@Component` decorator does not expose this directly, but the class returned by `createCustomElement` is a regular `HTMLElement` subclass -- you can extend it and override `attachShadow` before registering the tag:

```typescript
const SummaryElement = createCustomElement(AccountSummaryComponent, {
  injector: app.injector,
});

class SummaryElementWithDelegatedFocus extends SummaryElement {
  attachShadow(init: ShadowRootInit): ShadowRoot {
    return super.attachShadow({ ...init, delegatesFocus: true });
  }
}

customElements.define('fin-account-summary', SummaryElementWithDelegatedFocus);
```

With `delegatesFocus: true`, click and programmatic `.focus()` calls on the host element are automatically forwarded to the first focusable element inside the shadow root. The host element also matches the `:focus` and `:focus-within` pseudo-classes when any descendant is focused, which matters for partner-supplied focus styles.

Announce dynamic changes with `aria-live` regions exactly as the main FinancialApp does. The widget's internal focus trap, if any, should stay inside the shadow root -- trapping focus across the shadow boundary into the partner's page is both rude and technically messy.

---

## Distribution and Versioning

The widget bundle gets published in two channels. Partners can choose the one that fits their deployment model.

**npm package.** For partners who build their own frontend bundles, the widget is published as `@financial-app/widgets` on the registry:

```bash
npm install @financial-app/widgets
```

```javascript
import '@financial-app/widgets';
// <fin-account-summary> is now registered
```

The package side-effect is the `customElements.define` call; importing the package is what activates the widgets. Document this clearly -- a developer expecting a named export will be confused by an import whose only effect is a side effect.

**CDN.** For partners who embed a script tag, the same bundle is deployed to `cdn.financialapp.com/widgets/v{major}/fin-widgets.js`. Major versions live at stable URLs; minor and patch releases replace the content of a major-version URL. A partner pinning to `v1` receives bug fixes and non-breaking improvements automatically and never needs to change their script tag until FinancialApp ships a v2.

**Versioning strategy.** The widget follows semantic versioning with a conservative reading:

- **Patch.** Bug fixes, visual tweaks, internal refactors. Published without coordination.
- **Minor.** New optional inputs, new outputs, new widget tags. Published with a changelog entry; partners read the changelog if they want the new capability but are not required to change anything.
- **Major.** Breaking changes to inputs, outputs, or event payloads. Published to a new `v{N}` CDN path and a new npm major version. The previous major stays available indefinitely.

Breaking changes are expensive, because partners do not upgrade on your schedule -- some will pin to `v1` for years. FinancialApp's engineering guideline is that a major version bump for the widget requires the same sign-off as a breaking API change: it costs partner integration time, and partner integration time is what this whole distribution strategy exists to save.

The widget's public contract -- its tag names, input names, event names, theme tokens, and emitted event shapes -- is documented in a single `CONTRACT.md` file in the widget app's root. That file is the source of truth for partner integrators and for the team's own versioning decisions. When someone asks "is this a breaking change?", the answer is "open CONTRACT.md and check whether this change modifies anything documented there."

---

## Summary

Angular Elements is the right tool when an Angular component needs to live on a page Angular does not own. A component gets wrapped by `createCustomElement`, registered with the browser's `customElements` registry, and distributed as a single-file bundle. The host page -- React, Vue, server-rendered HTML, AngularJS in its twilight -- loads the bundle once and gets a new HTML tag to consume.

**The integration contract is the DOM itself.** Signal inputs become element properties and attributes; outputs become `CustomEvent`s; theming flows through CSS custom properties that pierce the shadow boundary. No shared JavaScript module, no framework coordination, no Angular knowledge required on the host side.

**Shadow DOM is almost always the right choice for embeddable widgets.** It isolates the widget's styles from the unpredictable CSS of the host page. The cost -- explicit theming through CSS custom properties -- is a feature rather than a bug: it turns the widget's visual API into a small, documented contract instead of "whatever classes happen to match."

**Bootstrap with `createApplication`, not `bootstrapApplication`.** An Elements-only build has no root component and no `index.html`. The `ApplicationRef` exists purely to provide an injector context for the registered elements; everything else is driven by the browser's custom-element lifecycle.

**Bundle size is the real cost.** A single Angular widget ships around 180 KB gzipped of framework runtime. This is acceptable for one widget on a partner page. For many widgets on the same page -- or many Angular-based micro frontends on the same page -- Native Federation from [Chapter 30](ch38-micro-frontends.md) shares the runtime across modules and is the better architecture.

Custom Elements and Native Federation solve adjacent problems. Federation integrates Angular applications that share a design language and a runtime. Elements distributes individual widgets to foreign hosts that have no runtime to share. Most organizations end up with both: federation inside the owned product surface, Elements at the edges where the product meets partners, legacy systems, and the wider web.
