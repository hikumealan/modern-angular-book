# Chapter 25: E2E Testing with Playwright

Unit tests prove that a service calculates portfolio returns correctly. Integration tests prove that the service, its HTTP interceptor, and the component that consumes it compose without error. Neither proves that a user can log in, navigate to the portfolio page, and see their holdings rendered in a table. End-to-end tests fill this gap. They drive a real browser, click real buttons, and assert against real rendered output -- the closest thing to a human walking through the application.

The cost is speed. A Vitest unit test executes in milliseconds. A Playwright E2E test launches a browser, starts a server, and waits for network responses -- seconds, sometimes tens of seconds. Write too many and your CI pipeline becomes a bottleneck. The art is in choosing *which* user flows deserve E2E coverage and keeping everything else in the faster layers.

> **Prerequisites:** [Chapter 7](ch07-testing-vitest.md) (Vitest unit/integration testing foundation).

---

## The Testing Pyramid

At the base sit **unit tests** from [Chapter 7](ch07-testing-vitest.md) -- hundreds of focused specs verifying signal stores, pipes, and service logic. In the middle sit **integration tests** -- component tests rendered with `TestBed` that verify template bindings and DI wiring. Both run in Vitest without a browser. **Storybook interaction tests** ([Chapter 28](ch28-storybook.md)) render components in isolation but drive them with real user interactions via `play` functions.

At the top sit **E2E tests**, exercising the full stack: compiled Angular app, real browser engine, routing, and authentication guards. They catch bugs that lower layers cannot -- a misconfigured route, a CSS `z-index` hiding a button, a Content Security Policy blocking an API call.

| Layer | Volume | Speed | Catches |
|---|---|---|---|
| Unit (Vitest) | Hundreds | Milliseconds | Logic errors, pure function regressions |
| Integration (TestBed) | Dozens | Seconds | Template binding bugs, DI wiring |
| Storybook interaction | Dozens | Seconds | Complex component interaction sequences |
| E2E (Playwright) | A handful | Tens of seconds | Routing, auth, API integration, visual regressions |

Write E2E tests for **critical user flows** -- the paths that, if broken, generate support tickets within minutes. For FinancialApp: logging in, viewing the portfolio, completing client onboarding, and navigating between sections.

---

## Playwright Setup in an Nx Workspace

Playwright drives Chromium, Firefox, and WebKit with a single API and installs its own browser binaries.

```bash
npm install -D @playwright/test
npx playwright install
```

Nx encourages separating E2E tests into their own project. The `project.json` registers the target so Nx can orchestrate it:

```json
// apps/financial-app-e2e/project.json
{
  "name": "financial-app-e2e",
  "projectType": "application",
  "targets": {
    "e2e": {
      "executor": "nx:run-commands",
      "options": {
        "commands": ["npx playwright test"],
        "cwd": "apps/financial-app-e2e"
      },
      "dependsOn": ["financial-app:build"]
    }
  }
}
```

The Playwright configuration defines browsers, the base URL, and a `webServer` block that launches the Angular dev server automatically:

```typescript
// apps/financial-app-e2e/playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './src/tests',
  fullyParallel: true,
  forbidOnly: !!process.env['CI'],
  retries: process.env['CI'] ? 2 : 0,
  workers: process.env['CI'] ? 1 : undefined,
  reporter: process.env['CI'] ? 'github' : 'html',

  use: {
    baseURL: 'http://localhost:4200',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],

  webServer: {
    command: 'npx nx serve financial-app',
    url: 'http://localhost:4200',
    reuseExistingServer: !process.env['CI'],
    timeout: 120_000,
  },
});
```

`reuseExistingServer` avoids starting a second dev server during local development. `retries: 2` in CI absorbs flaky failures, and `trace: 'on-first-retry'` captures DOM snapshots and network logs for diagnosis. Run via `nx e2e financial-app-e2e`, or use Playwright's interactive UI mode during development with `npx playwright test --ui`.

---

## Page Object Model

Raw Playwright tests scatter selectors across every file. When markup changes, every test that touches the changed element breaks. The Page Object Model centralizes selectors into reusable classes so tests read like user stories.

```typescript
// apps/financial-app-e2e/src/page-objects/accounts.page.ts
import { type Locator, type Page, expect } from '@playwright/test';

export class AccountsPage {
  readonly accountList: Locator;
  readonly balanceDisplay: Locator;

  constructor(private readonly page: Page) {
    this.accountList = page.getByRole('list', { name: /accounts/i });
    this.balanceDisplay = page.getByTestId('account-balance');
  }

  async goto(): Promise<void> {
    await this.page.goto('/accounts');
    await this.accountList.waitFor();
  }

  async selectAccount(name: string): Promise<void> {
    await this.accountList.getByRole('listitem').filter({ hasText: name }).click();
    await this.page.waitForURL(/\/accounts\/.+/);
  }

  async getBalance(): Promise<string> {
    return this.balanceDisplay.textContent() as Promise<string>;
  }
}
```

Playwright's locator API encourages **accessible selectors** -- `getByRole`, `getByLabel`, `getByText` -- that mirror how assistive technology finds elements. `getByTestId` is a fallback for elements lacking semantic roles. This aligns with [Chapter 22](ch22-accessibility-aria.md): if your application is properly labeled, your tests find elements the same way a screen reader does.

```typescript
// apps/financial-app-e2e/src/page-objects/navigation.page.ts
import { type Locator, type Page, expect } from '@playwright/test';

export class NavigationBar {
  readonly nav: Locator;

  constructor(private readonly page: Page) {
    this.nav = page.getByRole('navigation', { name: /main/i });
  }

  async navigateTo(section: string): Promise<void> {
    await this.nav.getByRole('link', { name: section }).click();
  }

  async expectActiveLink(section: string): Promise<void> {
    await expect(
      this.nav.getByRole('link', { name: section })
    ).toHaveAttribute('aria-current', 'page');
  }
}
```

---

## Smoke Tests for FinancialApp

Smoke tests verify that critical paths work end to end -- the happy path and a few key error scenarios.

### Navigation Flows

```typescript
// apps/financial-app-e2e/src/tests/smoke.spec.ts
import { test, expect } from '@playwright/test';
import { NavigationBar } from '../page-objects/navigation.page';
import { AccountsPage } from '../page-objects/accounts.page';

test.describe('Navigation', () => {
  test('navigates between major sections', async ({ page }) => {
    const nav = new NavigationBar(page);
    await new AccountsPage(page).goto();
    await nav.expectActiveLink('Accounts');

    await nav.navigateTo('Portfolio');
    await expect(page).toHaveURL(/\/portfolio/);

    await page.goBack();
    await expect(page).toHaveURL(/\/accounts/);
  });
});
```

This catches misconfigurations in the router setup from [Chapter 12](ch12-initialization-routes.md) -- a lazy-loaded route that fails to resolve, a redirect loop, or a guard that blocks navigation.

### API Round-Trips

E2E tests should not depend on a live backend. Playwright's `page.route()` intercepts requests and returns mock responses:

```typescript
test('renders transactions from API response', async ({ page }) => {
  await page.route('**/api/transactions*', (route) =>
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, description: 'Wire Transfer', amount: -5000, date: '2026-03-15' },
        { id: 2, description: 'Deposit', amount: 12000, date: '2026-03-16' },
      ]),
    })
  );

  await page.goto('/accounts/acct-001/transactions');
  const rows = page.getByRole('row');
  await expect(rows).toHaveCount(3);
  await expect(rows.nth(1)).toContainText('Wire Transfer');
});
```

### Multi-Page Workflows

The client onboarding flow from [Chapter 6](ch06-signal-forms.md) spans multiple form steps and a redirect -- exactly the kind of workflow that unit tests cannot cover:

```typescript
test('completes onboarding and redirects to client detail', async ({ page }) => {
  await page.route('**/api/clients', (route) =>
    route.fulfill({
      status: 201,
      contentType: 'application/json',
      body: JSON.stringify({ id: 'client-42', name: 'Acme Corp' }),
    })
  );

  await page.goto('/onboarding');
  await page.getByLabel('Company Name').fill('Acme Corp');
  await page.getByLabel('Tax ID').fill('12-3456789');
  await page.getByRole('button', { name: 'Next' }).click();

  await page.getByLabel('Primary Contact Email').fill('contact@acme.com');
  await page.getByRole('button', { name: 'Next' }).click();

  await page.getByRole('button', { name: 'Submit' }).click();
  await expect(page).toHaveURL(/\/clients\/client-42/);
  await expect(page.getByRole('heading')).toContainText('Acme Corp');
});
```

### Auth-Gated Routes

Authentication guards from [Chapter 16](ch16-auth-patterns.md) redirect unauthenticated users to login. E2E tests verify this works as an integrated system:

```typescript
test('redirects unauthenticated users to login', async ({ page }) => {
  await page.goto('/portfolio');
  await expect(page).toHaveURL(/\/login\?returnUrl=/);
});

test('authenticated users access portfolio', async ({ page }) => {
  await page.route('**/api/auth/session', (route) =>
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ user: 'analyst@acme.com', roles: ['viewer'] }),
    })
  );

  await page.goto('/portfolio');
  await expect(page).toHaveURL(/\/portfolio/);
  await expect(page.getByRole('heading')).toContainText('Portfolio');
});
```

---

## Visual Regression Testing

Functional assertions verify behavior. Visual regression testing verifies appearance -- a CSS change that shifts a button off-screen, a theme update that makes text unreadable, a layout break at a specific viewport width. Playwright includes built-in screenshot comparison:

```typescript
// apps/financial-app-e2e/src/tests/visual.spec.ts
import { test, expect } from '@playwright/test';

test('portfolio dashboard matches baseline', async ({ page }) => {
  await page.route('**/api/portfolio*', (route) =>
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        holdings: [
          { symbol: 'AAPL', shares: 150, value: 28500 },
          { symbol: 'GOOGL', shares: 50, value: 8750 },
        ],
      }),
    })
  );

  await page.clock.setFixedTime(new Date('2026-04-01T10:00:00Z'));
  await page.goto('/portfolio');
  await page.getByTestId('portfolio-summary').waitFor();

  await expect(page).toHaveScreenshot('portfolio-dashboard.png', {
    maxDiffPixelRatio: 0.01,
  });
});
```

Three techniques keep visual tests deterministic: **mock API responses** so the same content renders every time, **freeze time** with `page.clock.setFixedTime()` to prevent date-dependent diffs, and **set a pixel tolerance** to absorb sub-pixel rendering variations across platforms. Update baselines with `npx playwright test --update-snapshots` and review the diffs in your pull request.

---

## Accessibility Audits in E2E

[Chapter 22](ch22-accessibility-aria.md) showed how to run axe-core against individual components. E2E audits complement that by scanning the *composed* application -- a dialog trapping focus incorrectly, a duplicated landmark role, or a contrast violation introduced by a parent's background are invisible to per-component tests.

```bash
npm install -D @axe-core/playwright
```

```typescript
// apps/financial-app-e2e/src/tests/a11y.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('accounts page has no critical a11y violations', async ({ page }) => {
  await page.goto('/accounts');
  await page.getByRole('list', { name: /accounts/i }).waitFor();

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

The `withTags` filter scopes to WCAG conformance levels. The `include` method focuses on a DOM subtree when third-party widgets produce violations you cannot fix. Each audit adds less than a second to the test.

---

## Performance Auditing in CI

Performance budgets prevent gradual regression. Lighthouse CI automates audits and fails builds that exceed budgets.

```bash
npm install -D @lhci/cli
```

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:4200/', 'http://localhost:4200/portfolio'],
      startServerCommand: 'npx nx serve financial-app --configuration=production',
      startServerReadyPattern: 'Local:',
      numberOfRuns: 3,
      settings: { preset: 'desktop' },
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['warn', { maxNumericValue: 300 }],
      },
    },
    upload: { target: 'temporary-public-storage' },
  },
};
```

Run with `npx lhci autorun`. Three runs per URL smooth out measurement variance. If LCP exceeds 2.5 seconds or the accessibility score drops below 95%, the step fails. Start with budgets your application currently meets, then tighten over time.

---

## CI Integration

A GitHub Actions workflow ties everything together:

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on:
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1/3, 2/3, 3/3]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium firefox webkit
      - run: npx nx build financial-app --configuration=production
      - name: Run Playwright tests
        run: npx playwright test --shard=${{ matrix.shard }}
        working-directory: apps/financial-app-e2e
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report-${{ strategy.job-index }}
          path: apps/financial-app-e2e/playwright-report/
          retention-days: 14
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: test-traces-${{ strategy.job-index }}
          path: apps/financial-app-e2e/test-results/
          retention-days: 7

  lighthouse:
    runs-on: ubuntu-latest
    needs: e2e
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npx lhci autorun
```

Playwright's `--shard` flag splits the suite across parallel CI jobs. Reports upload unconditionally; traces (large files with DOM snapshots) upload only on failure. The Lighthouse job depends on E2E (`needs: e2e`) so performance is not audited if the application is functionally broken.

---

## Summary

E2E tests are the final layer of confidence before code reaches users. They are expensive, so spend them wisely.

- **The testing pyramid** guides allocation: hundreds of unit tests, dozens of integration tests, a handful of E2E tests. Focus E2E on critical user flows and let Vitest ([Chapter 7](ch07-testing-vitest.md)) and Storybook ([Chapter 28](ch28-storybook.md)) handle the rest.
- **Playwright** provides a single API across Chromium, Firefox, and WebKit. Its locator API encourages accessible selectors aligned with [Chapter 22](ch22-accessibility-aria.md).
- **Page objects** encapsulate selectors so tests read like user stories and markup changes update one class, not every test file.
- **Mock API responses** with `page.route()` for deterministic, backend-independent tests.
- **Visual regression testing** with `toHaveScreenshot()` catches CSS and layout bugs that functional assertions miss.
- **Accessibility audits** with `@axe-core/playwright` scan composed pages for WCAG violations that per-component tests cannot detect.
- **Lighthouse CI** enforces performance budgets and fails builds that regress.
- **CI integration** with GitHub Actions shards tests, uploads artifacts, and gates Lighthouse behind passing E2E tests.

The unit tests from [Chapter 7](ch07-testing-vitest.md) verify logic. The integration tests verify composition. The E2E tests in this chapter verify that the application works the way a user expects it to -- from login to logout, across browsers, with accessible markup and acceptable performance.

> **Companion code:** `financial-app/apps/financial-app-e2e/` contains Playwright config, page objects, smoke tests, and Lighthouse CI config.
