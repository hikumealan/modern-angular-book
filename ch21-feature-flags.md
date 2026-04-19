# Chapter 38: Feature Flags

A deployment is not a launch. The moment CI pushes a new build to production, thousands of users are running your code -- but that is not the same as those users *seeing* your new feature. The gap between "deployed" and "launched" is where feature flags live. They let you ship code to production with the new behavior hidden, then reveal it gradually: to an internal QA cohort first, then to one percent of real users, then five, then twenty-five, then everyone. When something goes wrong at three percent, you flip a toggle and the new code stops executing without a rollback, a hotfix, or a pager alert at 2am.

Feature flags decouple two things that deployments used to fuse: the act of publishing code and the act of exposing behavior. Once that separation exists, other patterns become possible. A/B tests that compare two checkout flows with the same underlying codebase. Kill switches that disable a flaky integration during an outage. Dark launches of a new ledger service that runs in parallel with the old one, shadowing writes and comparing outputs, before you ever show users a single pixel of new UI. Premium-only features that unlock when a user's subscription changes, without re-deploying the app.

This chapter builds a signal-based feature flag system for FinancialApp: how flags load at startup ([Chapter 48](ch13-initialization-routes.md)), how they integrate with the router's conditional lazy loading ([Chapter 9](ch32-performance.md)) and route guards ([Chapter 4](ch04-router.md)), how exposure events feed the observability pipeline ([Chapter 17](ch20-observability.md)), and how flag-driven code paths evolve into clean migrations ([Chapter 42](ch50-migrations.md)).

> **Companion code:** `financial-app/libs/shared/feature-flags/` with `FeatureFlagsService`, a JSON flag endpoint mock, and example flags wired into the FinancialApp shell.

---

## Categories of Flags

Not every flag does the same job, and conflating them leads to confusion in code reviews and dashboards. Most teams settle on four categories, each with different lifespans, targeting rules, and cleanup expectations.

**Release flags** guard code that is not yet ready for general availability. They are short-lived by design -- you create them when starting a feature, toggle them on when the feature is ready, and delete them a sprint or two later. For FinancialApp, a release flag like `new-transaction-filters` might live for three weeks while the redesigned filter panel rolls out. If a release flag is still in the codebase six months after its feature launched, you have flag debt.

**Experiment flags** drive A/B tests. Unlike release flags, they are not boolean toggles that go from off to on -- they are multivariate assignments that stay partitioned until the experiment concludes. A `checkout-button-variant` flag might return `'control'`, `'primary-blue'`, or `'primary-green'` depending on which bucket a user lands in. These flags have a defined experiment window, and cleanup happens when the winning variant is chosen and promoted.

**Permission flags** gate features by entitlement rather than rollout progress. `premium-dashboard`, `beta-features`, `advisor-role-tools` -- these flags check a user's plan, role, or opt-in status. They are long-lived by design; they do not "finish rolling out" because their purpose is permanent segmentation. The line between a permission flag and your authorization system blurs here, and in mature codebases the two often merge into a single entitlement service.

**Operational flags** are kill switches and circuit breakers. `enable-external-market-feed`, `allow-wire-transfers`, `use-new-risk-engine` -- these flags exist so that when a downstream service misbehaves or a third-party API goes dark, you can disable the feature that depends on it without a deploy. Operational flags often have no user targeting at all; they are global toggles flipped by on-call engineers during incidents. They live forever, because the thing they protect is always a potential liability.

The category shapes how a flag is evaluated, monitored, and retired. Release flags show up on cleanup dashboards. Experiment flags emit exposure events. Permission flags get checked against user attributes. Operational flags have their own alerting. Tagging each flag with its category at creation time keeps the model coherent as the list grows past a hundred.

---

## Build vs Buy

Once you commit to flags, the next question is where the flag configuration lives -- and who has permission to change it. The options form a spectrum from fully managed SaaS to a JSON file in your own repository.

**LaunchDarkly** is the enterprise standard. It offers rich targeting rules (by user attribute, device, geo, cohort), audit logs, approval workflows, percentage rollouts with stable hashing, scheduled launches, and a mature SDK for Angular. It also bills per seat and per monthly active user, which adds up quickly for a consumer app with millions of users. Teams building regulated financial products often choose LaunchDarkly specifically for the audit trail and RBAC controls that satisfy compliance reviewers.

**Unleash** is the open-source alternative. The core server is free to self-host, with a paid SaaS offering and paid tiers for advanced features like approval workflows and SSO. It has first-class Angular examples, a solid REST and SSE API for evaluating flags, and a similar targeting model to LaunchDarkly. For teams that want SaaS ergonomics without the SaaS bill, or that need to keep flag configuration inside a private network for compliance reasons, Unleash is the dominant choice.

**Flagsmith** occupies a similar niche to Unleash -- open source, self-hostable, with a free SaaS tier for smaller projects. Its targeting model emphasizes user traits and segments, and it ships a small Angular-friendly JavaScript SDK. The feature set overlaps heavily with Unleash; the choice between them often comes down to which operator experience your platform team prefers.

**Home-grown** is viable for small teams with simple needs. A JSON file served from a CDN, refreshed every few minutes, covers release flags and operational toggles for many applications. The cost is the features you give up: no per-user targeting without writing it yourself, no audit log without wiring one up, no approval flow without building it. For a pre-revenue startup with three engineers, a JSON file is exactly right. For a regulated bank with ten teams and a compliance officer, it is not.

FinancialApp, in the companion code, uses a home-grown service with a JSON endpoint. The architecture it establishes -- flags loaded at init, exposed as signals, evaluated through a single service -- applies identically to any of the managed providers. Swapping LaunchDarkly for the mock implementation is a matter of changing the initializer's fetch call, not rewriting consumers.

---

## A Signal-Based FeatureFlagsService

Flags are a natural fit for signals. Their values change (rarely, but they do), templates and components react to those changes, and `computed()` signals compose cleanly over boolean and multivariate values. The service's job is to load flag values at startup, expose each flag as a signal, and propagate updates when the backend pushes a new configuration.

The service lives in a shared library because every layer of the application consumes it -- the shell for navigation, features for conditional rendering, route guards for access control, analytics interceptors for exposure tracking.

```typescript
// libs/shared/feature-flags/src/lib/feature-flags.service.ts
import { Injectable, Signal, computed, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export type FlagValue = boolean | string | number;
export type FlagSnapshot = Record<string, FlagValue>;

@Injectable({ providedIn: 'root' })
export class FeatureFlagsService {
  private http = inject(HttpClient);
  private flags = signal<FlagSnapshot>({});

  async load(): Promise<void> {
    const snapshot = await firstValueFrom(
      this.http.get<FlagSnapshot>('/api/feature-flags'),
    );
    this.flags.set(snapshot);
  }

  isEnabled(key: string): Signal<boolean> {
    return computed(() => this.flags()[key] === true);
  }

  variant<T extends FlagValue>(key: string, fallback: T): Signal<T> {
    return computed(() => {
      const value = this.flags()[key];
      return (value ?? fallback) as T;
    });
  }

  snapshot(): FlagSnapshot {
    return this.flags();
  }

  applyOverride(key: string, value: FlagValue): void {
    this.flags.update((current) => ({ ...current, [key]: value }));
  }
}
```

The `isEnabled` method returns a `Signal<boolean>`, not a plain boolean. This is the key design decision: every consumer subscribes to flag state through the signal graph. When the backing signal updates -- because of a polling refresh, a Server-Sent Event, or a runtime override -- every template and computed signal that reads `isEnabled('some-flag')()` re-evaluates automatically. No manual subscription management, no ad-hoc event bus.

Flags load during bootstrap through `provideAppInitializer`, the same mechanism [Chapter 48](ch13-initialization-routes.md) used for configuration loading. Angular waits for the promise to settle before rendering the root component, so the first paint already has correct flag values:

```typescript
// apps/financial-app/src/app/app.config.ts
import { provideAppInitializer, inject } from '@angular/core';
import { FeatureFlagsService } from '@financial-app/shared/feature-flags';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAppInitializer(() => inject(FeatureFlagsService).load()),
  ],
};
```

For real-time kill switches, polling or SSE keeps flags fresh after bootstrap. Polling is simple -- a timer re-fetches the snapshot every 30 or 60 seconds -- but adds latency that can stretch to the full polling interval during an incident. SSE pushes updates immediately at the cost of a persistent connection. FinancialApp uses SSE for operational flags and polling for the rest:

```typescript
// libs/shared/feature-flags/src/lib/feature-flags.service.ts (continued)
connectLiveUpdates(): void {
  const source = new EventSource('/api/feature-flags/stream');
  source.addEventListener('update', (event) => {
    const patch = JSON.parse((event as MessageEvent).data) as FlagSnapshot;
    this.flags.update((current) => ({ ...current, ...patch }));
  });
}
```

When the on-call engineer flips `allow-wire-transfers` to `false`, every user has the new value within a second. Components reading `flags.isEnabled('allow-wire-transfers')()` re-render immediately, disabling the transfer button without waiting for the next navigation.

---

## Template Patterns

Inside templates, flags compose with `@if` and `@switch` naturally. The service returns a signal, and the template invokes it with `()`:

```html
<!-- transactions-page.component.html -->
@if (flags.isEnabled('new-transaction-filters')()) {
  <app-new-filters [accounts]="accounts()" />
} @else {
  <app-legacy-filters [accounts]="accounts()" />
}

@if (flags.isEnabled('export-to-csv')()) {
  <button class="secondary" (click)="exportCsv()">Export CSV</button>
}
```

The component that owns the template exposes the service through a public field so the template can reach it:

```typescript
@Component({
  selector: 'app-transactions-page',
  imports: [NewFiltersComponent, LegacyFiltersComponent],
  templateUrl: './transactions-page.component.html',
})
export class TransactionsPageComponent {
  protected flags = inject(FeatureFlagsService);
  accounts = input.required<Account[]>();

  exportCsv(): void { /* ... */ }
}
```

For multivariate flags, `variant()` returns a signal whose value drives a `@switch`:

```html
@switch (flags.variant('checkout-button-variant', 'control')()) {
  @case ('primary-blue') { <button class="cta-blue">Continue</button> }
  @case ('primary-green') { <button class="cta-green">Continue</button> }
  @default { <button class="cta-default">Continue</button> }
}
```

The default branch matters. If the flag service has not loaded yet, or if the user was not assigned to any bucket, the template falls back to the control variant. This is defensive programming applied to rollouts: when in doubt, show the old thing.

---

## Route-Level Flags

Template-level gating handles in-page features. Route-level gating handles entire pages and bundles. [Chapter 9](ch32-performance.md) covered conditional `loadComponent`, which runs inside Angular's injection context so it can read from services. Flags are the canonical use case.

A guard redirects away from a feature that is not enabled for the current user:

```typescript
// libs/shared/feature-flags/src/lib/flag.guard.ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';
import { FeatureFlagsService } from './feature-flags.service';

export const requireFlag = (key: string, redirectTo = '/upgrade'): CanActivateFn =>
  () => {
    const flags = inject(FeatureFlagsService);
    const router = inject(Router);
    return flags.isEnabled(key)() ? true : router.createUrlTree([redirectTo]);
  };
```

The factory `requireFlag('premium-dashboard')` returns a `CanActivateFn`. The `UrlTree` return path preserves the redirect model from [Chapter 48](ch13-initialization-routes.md), so other guards and resolvers see a consistent cancellation rather than a stale in-flight navigation. Apply it on the route:

```typescript
// apps/financial-app/src/app/app.routes.ts
{
  path: 'premium-dashboard',
  canActivate: [requireFlag('premium-dashboard')],
  loadComponent: () =>
    import('./features/premium-dashboard/premium-dashboard.component')
      .then((m) => m.PremiumDashboardComponent),
}
```

For migrations where a new version of a component is rolling out alongside the old one, the `loadComponent` itself branches. This technique appeared in [Chapter 9](ch32-performance.md) for the free-vs-premium dashboard; the same pattern applies to version-based rollouts:

```typescript
{
  path: 'dashboard',
  loadComponent: () => {
    const flags = inject(FeatureFlagsService);
    return flags.isEnabled('v2-dashboard')()
      ? import('./features/dashboard-v2/dashboard.component')
          .then((m) => m.DashboardV2Component)
      : import('./features/dashboard-v1/dashboard.component')
          .then((m) => m.DashboardV1Component);
  },
}
```

Users in the v2 cohort download only the v2 bundle; users in the v1 cohort download only v1. There is no shipping-both-and-hiding-one overhead, which is what makes this pattern valuable for large components.

---

## Targeting

A boolean `enabled: true` file is fine for release flags that go to everyone. Real rollouts are more nuanced: ten percent of users on the iOS app, or everyone in Germany with a Premium plan, or the internal QA team plus one percent of the wider population. The server-side evaluation engine typically handles this, but the Angular client still needs to send the right context.

The client collects a user context object at startup -- user id, plan tier, country, account age in days, device type -- and includes it when fetching flags. The server evaluates its rules against that context and returns the per-user snapshot.

```typescript
// libs/shared/feature-flags/src/lib/user-context.ts
export interface UserContext {
  userId: string;
  plan: 'free' | 'standard' | 'premium';
  country: string;
  accountAgeDays: number;
  device: 'mobile' | 'tablet' | 'desktop';
}

// libs/shared/feature-flags/src/lib/feature-flags.service.ts (updated load method)
async load(context: UserContext): Promise<void> {
  const snapshot = await firstValueFrom(
    this.http.post<FlagSnapshot>('/api/feature-flags/evaluate', context),
  );
  this.flags.set(snapshot);
}
```

Percentage rollouts deserve a note. A naive implementation -- `Math.random() < 0.1` -- would assign a different bucket on every load, so a user would see the new feature one moment and the old one the next. Stable hashing fixes this. The server hashes `userId + flagKey` with a fast hash (xxHash, FNV) and maps the result to a 0-100 range. As long as the user id is consistent, the assignment is stable across sessions and devices. A user at percentile 7 remains at percentile 7 forever, so the flag is either always on or always off for them -- until the rollout percentage passes their threshold.

Device and viewport targeting is browser-side. The `UserContext` can include screen size, connection type (`navigator.connection.effectiveType`), or PWA install state. These values are not sensitive and can live on the client; user entitlements should always be evaluated server-side.

---

## Analytics Integration

A/B testing is useless without correlating flag exposure to user behavior. The observability chapter ([Chapter 17](ch20-observability.md)) covered how FinancialApp sends events to its analytics pipeline; flag exposure is one of the most valuable event streams you can emit. Every time a user *sees* a feature -- not when the flag is evaluated, not when the component is instantiated, but when the user actually experiences the variant -- an exposure event fires.

The service emits the event the first time a flag is read per session:

```typescript
// libs/shared/feature-flags/src/lib/feature-flags.service.ts (with analytics)
private analytics = inject(AnalyticsService);
private exposed = new Set<string>();

isEnabled(key: string): Signal<boolean> {
  return computed(() => {
    const value = this.flags()[key] === true;
    this.reportExposure(key, value);
    return value;
  });
}

private reportExposure(key: string, value: FlagValue): void {
  if (this.exposed.has(key)) return;
  this.exposed.add(key);
  this.analytics.track('feature_flag_exposed', { key, value });
}
```

Downstream, your analytics warehouse joins `feature_flag_exposed` events to conversion events: transactions completed, accounts opened, sessions that lasted more than five minutes. The statistical test that determines whether variant A beats variant B runs against this joined data.

Two subtleties are worth calling out. First, `computed()` runs its body only when someone reads the signal, which means the exposure fires at the moment of render -- not at load time. This matches the semantic you want: a user is "exposed" when they see the feature, not when the flag was fetched. Second, the deduplication via `Set` prevents flooding the analytics pipeline with one event per change detection cycle. Once per session per flag is the right granularity for experiment analysis.

---

## Flag Hygiene

Every flag is a fork in your code. A hundred active flags means 2^100 theoretical branches, and while the practical number is much lower, the cognitive load on reviewers and the surface area for bugs grow with every addition. Healthy teams treat flag hygiene as an ongoing engineering responsibility, not an afterthought.

**Lifecycle.** A flag is created with a hypothesis ("reduce friction on checkout"), an owner, a category, and a planned removal date. When the rollout completes, the code path for the losing (or older) variant is deleted, and the flag is retired from the configuration service. Calendar reminders and dashboards help, but the highest-leverage intervention is a linter.

**Dead-flag detection.** A custom ESLint rule scans for `isEnabled('known-flag-name')` calls whose flag is either permanently `true` or missing from the flag registry. When a release flag has been at 100% for 30 days, the rule starts failing the build, forcing the team to delete the dead branch. Storing flag metadata (category, creation date, rollout percentage) in a JSON file committed alongside the code makes this linter trivial to implement.

**Flag debt.** The typical anti-pattern is the flag that ships as a release flag, rolls out successfully, and never gets cleaned up. A year later, someone changes the adjacent code and introduces a subtle regression in the flag-off branch that no user has hit in 11 months. Or worse: the flag stays at 100% until a developer who does not know its history flips it back to `false` for a debugging session and breaks production. Treat flag debt like technical debt -- make it visible on a dashboard, budget time to pay it down each sprint, and resist the urge to let "temporary" flags calcify into permanent architecture.

**Naming conventions.** Prefix flags with their category: `release-new-transaction-filters`, `experiment-checkout-button-variant`, `perm-premium-dashboard`, `ops-allow-wire-transfers`. The prefix makes dashboards easier to filter and makes the flag's expected lifespan obvious at the call site.

---

## Testing Feature-Flagged Code

Flag-conditional code has to be tested in both states. A feature that works perfectly with the flag on but crashes with it off is a production outage waiting for the kill switch to fire. [Chapter 7](ch07-testing-vitest.md) covered Vitest for Angular; mocking the flag service is a three-line operation:

```typescript
// libs/shared/feature-flags/src/lib/testing/provide-mock-flags.ts
import { Provider, signal } from '@angular/core';
import { FeatureFlagsService, FlagSnapshot } from '../feature-flags.service';

export function provideMockFlags(flags: FlagSnapshot): Provider {
  const mockService = {
    isEnabled: (key: string) => signal(flags[key] === true).asReadonly(),
    variant: <T>(key: string, fallback: T) =>
      signal((flags[key] ?? fallback) as T).asReadonly(),
    snapshot: () => flags,
  };
  return { provide: FeatureFlagsService, useValue: mockService };
}
```

Tests provide the mock in the TestBed configuration, setting up the exact flag state each case needs:

```typescript
// apps/financial-app/src/app/features/transactions/transactions-page.spec.ts
import { describe, it, expect } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { TransactionsPageComponent } from './transactions-page.component';
import { provideMockFlags } from '@financial-app/shared/feature-flags/testing';

describe('TransactionsPageComponent', () => {
  it.each([
    ['new-transaction-filters on', { 'new-transaction-filters': true }, 'app-new-filters'],
    ['new-transaction-filters off', { 'new-transaction-filters': false }, 'app-legacy-filters'],
  ])('renders correct filters when %s', (_, flags, expectedSelector) => {
    TestBed.configureTestingModule({
      providers: [provideMockFlags(flags)],
    });
    const fixture = TestBed.createComponent(TransactionsPageComponent);
    fixture.detectChanges();
    expect(fixture.nativeElement.querySelector(expectedSelector)).not.toBeNull();
  });
});
```

Vitest's `it.each` parameterizes the test across flag states, so both branches are covered in a single, declarative block. The same mock provider works in Storybook ([Chapter 28](ch26-storybook.md)): register a decorator that injects flag overrides per story, and a single component gets separate stories for each flag state it responds to.

```typescript
// libs/shared/ui/src/lib/transactions-page.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { moduleMetadata } from '@storybook/angular';
import { provideMockFlags } from '@financial-app/shared/feature-flags/testing';
import { TransactionsPageComponent } from './transactions-page.component';

const meta: Meta<TransactionsPageComponent> = {
  component: TransactionsPageComponent,
};
export default meta;

export const LegacyFilters: StoryObj<TransactionsPageComponent> = {
  decorators: [moduleMetadata({ providers: [provideMockFlags({ 'new-transaction-filters': false })] })],
};

export const NewFilters: StoryObj<TransactionsPageComponent> = {
  decorators: [moduleMetadata({ providers: [provideMockFlags({ 'new-transaction-filters': true })] })],
};
```

Designers and QA can click through both variants side-by-side without touching a database or editing a JSON file. When the rollout completes and the legacy branch is deleted, the `LegacyFilters` story disappears with it, keeping the gallery honest.

---

## Flag-Driven Migrations

Feature flags shine brightest during migrations. [Chapter 42](ch50-migrations.md) covers the strategic view of large framework and architecture migrations, but the tactical pattern at the code level is almost always the same: new code and old code live side by side, a flag routes traffic between them, and the flag gradually moves toward 100% on the new path.

Consider migrating FinancialApp's transaction list from a classic reactive-forms-backed filter to the new signal forms from [Chapter 6](ch06-signal-forms.md). A big-bang rewrite would ship on one deploy and succeed or fail catastrophically. A flag-driven migration ships the new implementation alongside the old one, routes an increasing share of traffic to the new path, and finishes when both paths have demonstrated parity on live data.

The steps are:

1. Implement the new component alongside the old. Both expose the same input and output contract, so the parent page treats them as interchangeable.
2. Introduce `release-signal-forms-filters` as a release flag starting at 0%.
3. Ship to production. Nothing changes for users.
4. Enable the flag for the internal QA cohort (via user id targeting). The QA team tests the new path in production.
5. Enable for 1% of real users. Monitor the exposure events joined with error rates from [Chapter 17](ch20-observability.md). Compare metrics between the two cohorts.
6. Progress to 10%, 50%, 100% as confidence grows. At any percentage, flipping the flag back to 0% reverts the change instantly.
7. Once 100% has been stable for a sprint, delete the old component and the flag check. The dead-flag linter will catch if either is forgotten.

At no point between step 2 and step 7 is a rollback a deploy. This property -- the ability to reverse a change without a code change -- is the core value proposition of feature flags during migrations. The parallel code paths add transient complexity; the flag-driven process removes the cost of being wrong.

---

## Summary

Feature flags turn a deploy into a catalog of toggles. You ship code continuously, you reveal behavior deliberately, and you reverse mistakes without redeploying.

- **Four categories** cover the span of use cases: release flags are short-lived rollout toggles, experiment flags drive A/B tests, permission flags gate entitlements, and operational flags are kill switches. Tag each flag at creation time.
- **Build vs buy** depends on scale and compliance. LaunchDarkly, Unleash, and Flagsmith handle the heavy lifting; a home-grown JSON endpoint works for small teams and is the starting point in FinancialApp's companion code.
- **A `FeatureFlagsService` built on signals** makes every flag a `Signal<boolean>` or `Signal<T>`, letting templates and `computed()` react to flag changes the same way they react to any other state.
- **Application initializers** ([Chapter 48](ch13-initialization-routes.md)) load flags before first paint. SSE or polling keeps the snapshot fresh after bootstrap so operational kill switches take effect in seconds.
- **Template patterns** use `@if` and `@switch` to branch on flag values -- no structural directive gymnastics required.
- **Route-level flags** integrate with guards (redirect if a flag is off) and conditional `loadComponent` ([Chapter 9](ch32-performance.md)) to split bundles along flag boundaries.
- **Targeting** sends a `UserContext` with each evaluation. Percentage rollouts rely on stable hashing of user id plus flag key so assignments persist across sessions.
- **Exposure events** emitted from `isEnabled()` feed the observability pipeline ([Chapter 17](ch20-observability.md)), making A/B test analysis a matter of joining exposure events to outcomes.
- **Flag hygiene** is a discipline: lifecycle dates, dead-flag linting, and naming prefixes prevent flags from calcifying into permanent debt.
- **Testing** flag-conditional code through a mock `FeatureFlagsService` works naturally with Vitest parameterized tests and Storybook stories per flag state ([Chapter 28](ch26-storybook.md)).
- **Flag-driven migrations** ([Chapter 42](ch50-migrations.md)) let new and old code paths live side by side, with the flag percentage as a dial that moves safely toward 100% and a kill switch that reverts mistakes without a deploy.

A feature flag is not the launch. A feature flag is the *capability* to launch -- on your schedule, to the audience you choose, with an undo button attached. The code to evaluate one is ten lines. The organizational discipline to use them well is what separates teams that ship confidently from teams that ship and pray.
