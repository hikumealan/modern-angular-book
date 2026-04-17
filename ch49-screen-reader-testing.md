# Chapter 49: Screen Reader Testing & Advanced A11y

> **Extends:** [Chapter 22](ch22-accessibility-aria.md) introduced Angular Aria and axe-core testing. This chapter moves into hands-on screen reader testing, cognitive accessibility, and the intersection of accessibility with mobile and compliance.

An `axe-core` report that says "zero violations" feels reassuring. It also lies, at least by omission. The audit catches violations a parser can detect -- missing `alt` attributes, invalid ARIA values, insufficient contrast ratios, duplicate landmark roles. It cannot tell you that your login form is technically labeled but announces "edit, edit, button" because every field shares the same generic name. It cannot tell you that a fund-transfer flow, while semantically correct, requires the user to memorize an eight-step sequence of landmarks because you never gave them a heading to anchor to. That only comes out when someone actually tries to use the application with a screen reader.

This chapter is about closing that gap. We will get VoiceOver and NVDA running, learn just enough shortcuts to navigate like a real user does, walk through three representative FinancialApp flows with assistive technology active, revisit the ARIA patterns from Chapter 22 with their actual screen-reader behavior audible in the background, and cover cognitive accessibility, mobile a11y, and a Playwright fixture that automates the catchable parts across every route.

---

## Screen Reader Setup

You do not need every reader. You need one on each operating system you target, one that matches the reader your users use most, and the discipline to exercise them periodically. Do not save this for the week before release -- by then, every issue you find is an emergency.

**VoiceOver (macOS/iOS)** ships with every Mac and iPhone. Toggle it with `Cmd+F5` on macOS (or triple-press the side button on iOS). `Ctrl` silences speech without disabling the reader, which you will want often during testing. The **Rotor** -- `Ctrl+Opt+U` -- is the killer feature: a modal menu that lets you jump between headings, landmarks, links, form controls, or tables on the current page. Arrow left/right to pick a category, up/down to move through items. If your markup is good, the Rotor is fast; if it is bad, the Rotor reveals exactly why.

**NVDA (Windows)** is free and open-source, downloadable from [nvaccess.org](https://www.nvaccess.org). Install it on a Windows VM or Boot Camp partition if you develop on macOS -- Windows Server-side rendering differs enough from WebKit that headless audits miss real bugs. `Insert` is the NVDA modifier key (often called "NVDA key"); `Insert+N` opens the menu, `Insert+Down` starts reading from the cursor, `Ctrl` silences. NVDA's web-browse mode auto-activates on pages and gives you single-letter navigation: `H` for next heading, `K` for next link, `F` for next form control, `D` for next landmark.

**JAWS (Windows)** is the commercial reader with roughly forty-percent market share, particularly strong in enterprise and government deployments. If you ship into financial services or the public sector, you need to exercise JAWS at least occasionally -- its heuristics differ enough from NVDA that fully-compliant markup still produces different announcements. A license is a few hundred dollars per year; Freedom Scientific offers a forty-minute "demo mode" that resets on reboot, which is enough for periodic smoke tests.

**Orca (Linux)** is the GNOME default, mostly used by Linux-first users. If your user research shows any non-trivial Linux audience, exercise it; otherwise, NVDA coverage is usually sufficient.

**TalkBack (Android)** and **VoiceOver (iOS)** on mobile are covered in more detail in [Chapter 40](ch40-capacitor-mobile.md), where we discuss the Capacitor wrapper around FinancialApp. Mobile readers have their own gesture vocabulary (one-finger swipe to move, two-finger swipe to scroll, double-tap to activate) that differs from desktop keyboard navigation.

**Chrome DevTools Accessibility tree** is your quick-and-dirty first-pass inspector. Open DevTools, click the **Accessibility** panel, and inspect any element to see its computed name, role, and state. If the name is empty or the role says "generic," no amount of CSS will fix the experience -- the element is invisible to assistive technology. Keep this panel open while writing components; it catches most regressions before they reach a real screen reader.

One more tip: turn on **captions for speech**. macOS calls this "VoiceOver caption panel" (enable in VoiceOver Utility); NVDA has a "Speech viewer" under the Tools menu. Both display what the reader is speaking as scrolling text, which makes bug reports repeatable. "The save button announces 'button, button' twice" is vastly more useful than "it sounds weird."

### Browser Pairings

Screen readers and browsers are not interchangeable. Each reader is most mature with a specific browser, and bugs in either half of the pair can masquerade as application bugs. The practical pairings:

- **VoiceOver + Safari** on macOS and iOS. This is the only pairing Apple tests against. Chrome on macOS with VoiceOver is acceptable but buggier; Firefox on macOS with VoiceOver is actively painful.
- **NVDA + Firefox** as the primary Windows combination. NVDA + Chrome is the secondary -- Chrome's accessibility tree matured significantly after v100 but still trails Firefox in edge cases.
- **JAWS + Chrome** is what enterprise Windows users typically run. JAWS + Edge is equivalent since both use Chromium.
- **TalkBack + Chrome** on Android; **VoiceOver + Safari** on iOS. Mobile readers do not support alternative browsers in any meaningful way.

Test against your primary pairing on every feature. Test against the secondary before each release. A bug that only appears on Chrome + VoiceOver is often an actual Chrome bug and not yours -- but if your users are on that combination, they still cannot use the app, so you still have to work around it.

---

## Learning the Essential Shortcuts

You do not need to learn every command. You need enough to navigate, read, and silence. The rest comes with practice.

**Navigate by semantic element.** Both VoiceOver (via Rotor) and NVDA (via single-letter keys) let you jump between headings, landmarks, form controls, links, and tables without tabbing through every focusable element. In NVDA: `H` next heading, `Shift+H` previous heading, `1`-`6` specific heading level, `D` next landmark, `F` next form field, `K` next link, `T` next table. In VoiceOver: open Rotor with `Ctrl+Opt+U`, arrow to a category, arrow to an item. **If your page has clear headings and landmarks, users never tab.** That is the point -- tabbing through every element is the fallback, not the default.

**Read from the cursor.** VoiceOver: `Ctrl+Opt+A` reads from the current cursor to the end. NVDA: `Insert+Down` does the same, `Insert+Up` reads the current line. This is how users consume long content: they point at a paragraph and tell the reader to continue.

**Stop speech.** VoiceOver: `Ctrl` silences. NVDA: `Ctrl` silences. Use it liberally while testing -- listening to a ten-minute page read-through is rarely necessary.

**Tab vs arrow-key navigation.** This is the part most sighted developers misunderstand. `Tab` cycles through *focusable* elements: buttons, links, form controls, `tabindex="0"` elements. Arrow keys move the **virtual cursor** through *readable* content: every paragraph, heading, list item, and text run. A paragraph of explanatory text is reachable by arrow but not by Tab, because there is no reason to focus on a paragraph. Users mix both: Tab to jump to the form, arrows to read the hints above it, Tab again to move between fields.

One consequence: **never hide informational content behind focus.** A tooltip that appears only on focus is invisible to a user who is arrow-reading the description above a field. Put the description in the DOM, associate it with `aria-describedby`, and the reader announces it automatically when focus reaches the input.

---

## Walkthrough: Login Flow with VoiceOver

Turn VoiceOver on (`Cmd+F5`), open the login page in Safari, and listen. Here is what the flow should sound like on FinancialApp, and what it often sounds like when things go wrong.

**Step 1 -- Page arrival.** A well-built login page announces: "FinancialApp, web content, Log in, heading level 1, main landmark." The page title (`<title>` in `index.html`), the `<main>` wrapper, and an `<h1>` at the top give the user an instant mental map. A broken version announces "FinancialApp, web content" and then silence, because there is no `<main>`, no heading, and focus landed on the body.

**Step 2 -- Finding the form.** The user presses `Ctrl+Opt+U`, arrows to "Form controls," and hears "Email address, edit text, required." On a broken page they hear "edit text" -- unlabeled. The fix is unambiguous: every `<input>` needs an associated `<label>` or an `aria-label`. The `FinFormField` wrapper from Chapter 22 handles this automatically via `[ariaLabelledByElements]`; raw `<input>` tags do not.

**Step 3 -- Typing and submitting.** The user types a password, tabs to "Log in, button," and presses Return. On a successful login the app navigates to `/dashboard`. Here is where most SPAs silently break: focus stays on the login button, which no longer exists in the DOM, so focus falls back to `<body>` and the user hears nothing. They have no idea what happened. The fix is the `RouteFocusService` from Chapter 22 -- after every navigation, move focus to the new page's `<h1>`. The user now hears "Dashboard, heading level 1" and knows the login succeeded.

**Step 4 -- Handling a failure.** The user enters a wrong password. The app displays an error banner below the form. If the banner is a plain `<div>`, the reader says nothing -- it is on a page the user is no longer actively reading. The fix is `role="alert"` (or the `LiveAnnouncer` service from Chapter 22) so the banner announces immediately: "Alert. Incorrect email or password. Please try again." After the announcement, focus should move back to the email field so the user can correct their input without hunting.

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

---

## Walkthrough: Transaction Submission with NVDA

Boot Windows, launch NVDA, open FinancialApp, and navigate to an account detail page. We are going to submit a new transaction and observe what the user hears.

**Step 1 -- Opening the form.** The user presses `B` (next button) until they reach "Add transaction, button, opens dialog." Good: `aria-haspopup="dialog"` tells the reader the activation opens something. They press Return; the dialog opens. The reader announces: "Dialog, New transaction, heading level 2." The dialog has `role="dialog"`, `aria-modal="true"`, and an `aria-labelledby` pointing to the heading.

**Step 2 -- Focus trap.** The user presses Tab. Focus moves to the first field. They keep tabbing; focus cycles through the dialog's fields and buttons, never escaping to the page behind it. This is `cdkTrapFocus` from [Chapter 22](ch22-accessibility-aria.md) doing its job. They press `Escape`; the dialog closes and focus returns to the "Add transaction" button. Without `cdkTrapFocusAutoCapture`, the user would be left on `<body>` and disoriented.

**Step 3 -- Inline validation errors.** The user enters "-500" into the Amount field (negative is invalid for this form) and tabs to Category. Chapter 23 showed the `FinFormField` wrapper linking the error message via `aria-describedby`. The reader announces: "Amount, edit, minus five hundred, invalid entry. Amount must be positive." The trailing error text arrives because `aria-invalid="true"` combined with `aria-describedby` causes NVDA to read the associated message automatically.

What happens if the error text is not associated? The reader says "Amount, edit, minus five hundred" and then silence. The user tabs forward thinking the value was accepted. When they eventually hit Submit, a banner fires "Form has errors" -- with no indication of *which* field. They are now hunting blindly.

**Step 4 -- The confirm dialog.** The user fills the form, presses Submit, and a confirm dialog from [Chapter 11](ch11-directives-templates.md) appears: "Are you sure you want to debit $500 from Checking?" Test two behaviors:

1. **Focus trap.** Tab cycles between "Cancel" and "Confirm" without escaping.
2. **Escape closes and restores.** Pressing `Escape` dismisses the dialog and restores focus to the Submit button, so the user can try again or correct the form. Both are handled by `cdkTrapFocus` and the dialog's own key handler.

**Step 5 -- Success.** After confirming, a toast announces "Transaction saved successfully" via `LiveAnnouncer.announce(..., 'polite')`. Focus does not move -- the user stays in the transaction form, ready to add another. A polite announcement waits for the reader to finish its current utterance; an assertive one interrupts, which is the right choice for errors but wrong for successes.

Run this flow once a sprint. Each of these steps corresponds to a specific ARIA or focus-management decision. Breaking any one of them turns the flow from ten seconds into two minutes of confusion.

---

## Walkthrough: Data-Heavy Pages (Portfolio with Charts)

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

**Text alternatives for charts.** A donut chart from [Chapter 35](ch35-charts.md) is purely visual. The SVG has no intrinsic meaning to a reader. Two fixes, used together: add `role="img"` and `aria-label` to the chart host for a summary sentence, and provide the raw data as a visually-hidden table that the reader can explore:

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

---

## Common Patterns Revisited with Real Screen-Reader Impact

The ARIA attributes from Chapter 22 have audible consequences. Knowing *what* each sounds like tells you when to reach for which.

**`aria-live="polite"`** queues an announcement until the reader finishes speaking. Use it for transient updates that the user does not need to know about *right now*: a background sync completing, a search result count updating, a toast confirming a save. The reader finishes its current utterance, then says the new one. The user is not interrupted.

**`aria-live="assertive"`** interrupts immediately. Use it sparingly -- error messages, critical state changes, session timeouts. An assertive announcement that fires during the user's own typing will cut them off mid-word, which is worse than useless. The `LiveAnnouncer.announce(msg, 'assertive')` from Chapter 22 wraps this safely.

**`aria-live="off"`** is the default and means "do not announce changes." This is what most of your DOM should be.

**`aria-busy="true"`** during loading suppresses repeat announcements of the region's contents while it is being updated. A deferred holdings table that re-renders three times during load should be wrapped in `aria-busy`, which suppresses the intermediate announcements. When `aria-busy` flips to `false`, the final content announces once.

**`aria-expanded` and `aria-controls`** power disclosure patterns. A "Show transaction details" button with `aria-expanded="false"` `aria-controls="txn-detail-42"` announces as "Show transaction details, collapsed, button." After activation: "expanded, button." The user knows the state changed without scrolling to find the newly-revealed content.

**`role="status"`** is a less-disruptive form of `role="alert"`. An inline "Saved" indicator next to a field gets `role="status"`; it announces politely. A form-wide "Payment failed" banner gets `role="alert"`; it announces assertively. Use the role that matches the urgency.

---

## Focus Management in SPA Navigation (Continued)

Chapter 22 introduced `RouteFocusService`. Two corner cases deserve attention.

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

Mobile assistive technology differs from desktop in ways that matter. [Chapter 40](ch40-capacitor-mobile.md) covers Capacitor integration; the accessibility implications are worth naming here.

**Touch targets.** WCAG 2.5.5 (Level AAA) requires touch targets to be at least 44x44 CSS pixels. WCAG 2.5.8 (Level AA in 2.2) relaxes this to 24x24 with spacing requirements. Either way, the 32x32 icon buttons that look fine on desktop are too small on a phone, especially for users with motor impairments. The minimum across FinancialApp is 44x44, with `padding` making the hit area larger than the visual icon where necessary.

**Gestures.** Mobile screen readers (TalkBack, VoiceOver on iOS) remap standard gestures. A one-finger swipe moves to the next element; a two-finger swipe scrolls. Double-tap activates the focused element; a three-finger swipe changes pages. The upshot: never rely on custom gestures that conflict with the reader's vocabulary. A "swipe to dismiss" toast should always have an equivalent close button, because swipe-to-dismiss means something else under TalkBack.

**Zoom and reflow.** WCAG 1.4.10 requires content to reflow at 320 CSS pixel width with no horizontal scrolling, except for content that genuinely requires two-dimensional layout (data tables, maps). Test this explicitly: open DevTools device emulation at 320px and verify that every page reads top-to-bottom without horizontal scroll. A fixed-width dashboard that demands 1200px is unusable at mobile sizes even with zoom. The responsive design techniques from Chapter 27 address this, but only if you test at the boundary.

---

## Accessibility Review Checklist for Pull Requests

A short, blunt checklist belongs in your pull-request template. Every reviewer asks these questions about every change that touches UI:

1. **Can I use the feature with keyboard alone?** Tab to it, activate it with Enter/Space, navigate its contents with arrows, dismiss it with Escape where applicable. If mouse is the only path, the feature is incomplete.
2. **Does every interactive element have a visible focus indicator?** Open DevTools, remove the default outline, and verify that your replacement is still obvious. A focus ring you cannot see from two feet away is a failed focus ring.
3. **Does my form error get announced?** `aria-describedby` on the input, `role="alert"` or `LiveAnnouncer` on form-wide errors, focus restored to the first invalid field on submit.
4. **Do I pass `axe-core`?** The helper from Chapter 22 or the Playwright fixture below.
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

Wire the sweep into the existing GitHub Actions pipeline from [Chapter 25](ch25-e2e-playwright.md):

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

## Summary

- **`axe-core` catches what a parser can see.** Everything else -- whether a workflow is actually usable -- requires real assistive technology. Both matter.
- **Get at least two readers running.** VoiceOver on macOS, NVDA on Windows. JAWS if you ship to enterprise. Chrome DevTools Accessibility panel for quick inspection while writing components.
- **Learn the minimum shortcuts.** Rotor, heading navigation, read-from-cursor, silence. Enough to move like a user, not a tutorial.
- **Walk through flows end-to-end.** Login, transaction submission, portfolio dashboard. Each surfaces different patterns: route focus, inline errors, table semantics, chart alternatives.
- **ARIA attributes have audible consequences.** `polite` vs `assertive`, `aria-busy`, `aria-expanded`, `role="status"` -- pick the one that matches what you want the user to hear, not the one that silences the lint rule.
- **Cognitive accessibility is a separate discipline.** Plain language, actionable errors, consistent placement, short sentences. Benefits everyone.
- **Mobile a11y differs.** 44x44 touch targets, no custom gestures, reflow at 320px. Tested in conjunction with the Capacitor build from [Chapter 40](ch40-capacitor-mobile.md).
- **A pull-request checklist of five questions** catches the majority of regressions before they ship.
- **A Playwright fixture running `@axe-core/playwright` across every route** gives you continuous regression coverage at CI-acceptable cost. Built on the E2E foundation from [Chapter 25](ch25-e2e-playwright.md).

WCAG AA is the floor. The ceiling is an application that disabled users reach for by choice, not because it is the only one their employer approves. Screen-reader testing, cognitive review, and mobile verification are how you get there.

> **Companion code:** `financial-app/libs/shared/testing/a11y-fixture.ts` with a Playwright fixture that audits every feature route.
