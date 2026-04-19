# Modern Testing with Vitest

Untested code is unreliable code. No matter how elegant the signal graph or how clean the architecture, an Angular application earns its developers' confidence only when automated tests prove that it behaves correctly -- and keeps behaving correctly as the codebase evolves. In Angular v21, the testing story has changed: **Vitest is the default test runner**, replacing the Karma/Jasmine combination that served the ecosystem for a decade.

This chapter walks through Vitest from first principles, shows how to integrate it with Angular's `TestBed`, and builds a testing toolkit for the FinancialApp. By the end you will be able to test components, services, stores, and HTTP interactions -- all in a fast, modern, ESM-native test runner.

> **Prerequisites:** Familiarity with signal-based components ([Chapter 2](ch02-signal-components.md)) and services with signals ([Chapter 5](ch05-state-management.md)). We will test many of the constructs introduced in those chapters.

---

## Vitest

Vitest is an ESM-native test framework built on top of Vite. It shares the same configuration, plugin pipeline, and dependency resolution as the Vite dev server that Angular v21 uses under the hood. The practical result: zero configuration drift between your application build and your test build.

### Structure of a Vitest Test

A Vitest test file follows the same `describe` / `it` / `expect` structure familiar from other frameworks, but imports everything from `vitest`:

```typescript
import { describe, it, expect } from 'vitest';
import { formatCurrency } from './format-currency.util';

describe('formatCurrency', () => {
  it('formats positive amounts with two decimals', () => {
    expect(formatCurrency(1234.5, 'USD')).toBe('$1,234.50');
  });

  it('formats negative amounts with a minus sign', () => {
    expect(formatCurrency(-42, 'EUR')).toBe('-\u20AC42.00');
  });

  it('returns an empty string for NaN', () => {
    expect(formatCurrency(NaN, 'USD')).toBe('');
  });
});
```

Each `it` block defines a single behavior. The `describe` block groups related behaviors under a common label. Nesting is allowed but rarely needed -- flat structures are easier to scan.

### Skipping Tests

During development you often need to isolate a single test or temporarily disable a failing one. Vitest provides two mechanisms:

```typescript
it.skip('handles edge case with zero balance', () => {
  // this test will not run, but it will appear as "skipped" in the report
});

it.only('the one test I am debugging right now', () => {
  // only this test (and any other .only tests) will run in this file
});
```

Use `it.skip` for known-broken tests that you will fix later. Use `it.only` for focused debugging -- but never commit it. Most teams enforce this with a lint rule.

### Turning on Browser Mode

By default, Vitest runs tests in a Node.js environment using `jsdom` or `happy-dom` for DOM simulation. For tests that require a real browser (e.g., testing scroll behavior, canvas APIs, or visual regressions), Vitest supports browser mode:

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: 'playwright',
      instances: [
        { browser: 'chromium' },
      ],
    },
  },
});
```

Browser mode runs each test in a real Chromium instance via Playwright. It is slower than `jsdom` but eliminates an entire class of false positives caused by DOM emulation gaps. For the FinancialApp, we use `jsdom` for unit tests and reserve browser mode for a small suite of integration tests.

### Running Tests

Angular v21 projects generated with `ng new` come with Vitest preconfigured. Running tests is straightforward:

```bash
# run all tests once
npx ng test

# run in watch mode (re-runs on file changes)
npx ng test --watch

# run a specific test file
npx vitest run src/app/domains/transactions/transaction-search.component.spec.ts

# run tests matching a name pattern
npx vitest run --reporter=verbose -t "TransactionSearch"
```

Watch mode is the default workflow during development. Vitest's HMR-aware file watcher re-runs only the tests affected by your change, keeping feedback loops tight even in large codebases.

---

## Angular and Vitest

Pure utility functions can be tested with Vitest alone. But Angular components, services, and stores rely on the dependency injection system, making `TestBed` essential for most Angular tests.

### Preparing the TestBed

`TestBed` configures a lightweight Angular environment for a single test suite. In Angular v21, you configure it in zoneless mode -- no `zone.js`, no automatic change detection. This matches production behavior and eliminates an entire category of test flakiness:

```typescript
import { TestBed } from '@angular/core/testing';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { describe, it, expect, beforeEach } from 'vitest';

describe('TransactionSearchComponent', () => {
  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideExperimentalZonelessChangeDetection(),
      ],
    });
  });
});
```

With zoneless testing, you take explicit control of when change detection runs. This makes tests deterministic: you know exactly what state the DOM is in at every assertion.

### A First Component Test

Let us test the `TransactionCardComponent` -- a presentational component that displays a single transaction (see [Chapter 2](ch02-signal-components.md) for the component implementation):

```typescript
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { describe, it, expect, beforeEach } from 'vitest';
import { TransactionCardComponent } from './transaction-card.component';

describe('TransactionCardComponent', () => {
  let fixture: ComponentFixture<TransactionCardComponent>;

  beforeEach(() => {
    fixture = TestBed.configureTestingModule({
      imports: [TransactionCardComponent],
      providers: [provideExperimentalZonelessChangeDetection()],
    }).createComponent(TransactionCardComponent);
  });

  it('displays the transaction description', () => {
    fixture.componentRef.setInput('transaction', {
      id: 1,
      description: 'Grocery Store',
      amount: -45.99,
      type: 'debit',
      category: 'food',
      date: '2026-03-15',
      pending: false,
      accountId: 10,
    });
    fixture.detectChanges();

    const el = fixture.nativeElement as HTMLElement;
    expect(el.textContent).toContain('Grocery Store');
  });

  it('applies a pending class when the transaction is pending', () => {
    fixture.componentRef.setInput('transaction', {
      id: 2,
      description: 'Wire Transfer',
      amount: 500,
      type: 'credit',
      category: 'transfer',
      date: '2026-03-16',
      pending: true,
      accountId: 10,
    });
    fixture.detectChanges();

    const card = fixture.nativeElement.querySelector('.transaction-card');
    expect(card.classList.contains('pending')).toBe(true);
  });
});
```

Key points: inputs are set via `componentRef.setInput()`, which works with signal-based `input()` properties. After setting inputs, call `fixture.detectChanges()` to trigger rendering.

### Working with Locators

Querying the DOM with raw CSS selectors is brittle. If a developer changes a class name for styling purposes, tests break for the wrong reason. Angular's `HarnessLoader` provides **component harnesses** -- stable, semantic APIs for interacting with components:

```typescript
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { MatButtonHarness } from '@angular/material/button/testing';

it('disables the submit button when the form is invalid', async () => {
  const loader = TestbedHarnessEnvironment.loader(fixture);
  const button = await loader.getHarness(
    MatButtonHarness.with({ text: 'Search' })
  );

  expect(await button.isDisabled()).toBe(true);
});
```

Harnesses abstract away the internal DOM structure of Material components. When the Material team changes the button's markup, the harness is updated accordingly and your test continues to pass.

### Locating Elements via DebugElement

For custom components without harnesses, `DebugElement` provides a middle ground between raw DOM queries and full harnesses:

```typescript
import { By } from '@angular/platform-browser';

it('emits a search event when the user submits the form', () => {
  const debugEl = fixture.debugElement;
  const form = debugEl.query(By.css('form'));

  fixture.componentRef.setInput('initialQuery', 'groceries');
  fixture.detectChanges();

  let emittedQuery = '';
  fixture.componentInstance.search.subscribe((q: string) => {
    emittedQuery = q;
  });

  form.triggerEventHandler('submit', new Event('submit'));
  expect(emittedQuery).toBe('groceries');
});
```

`By.css()` returns a `DebugElement` rather than a raw DOM node, giving you access to Angular-specific metadata like component instances and injected directives. `By.directive()` is also available for querying by Angular directive type.

### Defining Default Timeouts

Some tests -- particularly those involving animations or debounced inputs -- may exceed the default 5-second timeout. Configure a global default in `vitest.config.ts`:

```typescript
export default defineConfig({
  test: {
    testTimeout: 10_000,
    hookTimeout: 10_000,
  },
});
```

Prefer increasing the timeout globally rather than scattering `{ timeout: 10000 }` across individual tests. If a test routinely needs more than a few seconds, that is a signal to use fake timers instead (covered later in this chapter).

---

## Mocking and Spies

Real applications have dependencies -- HTTP backends, browser APIs, third-party services. Tests that hit real dependencies are slow, flaky, and hard to set up. Mocking replaces those dependencies with controlled substitutes so you can test your code in isolation.

### Mocking Services

The most common mock target is a service injected into a component. For the `TransactionSearchComponent`, which depends on `AccountService`, you provide a mock implementation through `TestBed`:

```typescript
import { vi, describe, it, expect, beforeEach } from 'vitest';
import { signal } from '@angular/core';
import { AccountService } from '@financial-app/accounts';
import { TransactionSearchComponent } from './transaction-search.component';

describe('TransactionSearchComponent', () => {
  const mockAccountService = {
    accounts: signal([
      { id: 1, name: 'Checking', type: 'checking', balance: 2500, currency: 'USD', ownerId: 1 },
      { id: 2, name: 'Savings', type: 'savings', balance: 10000, currency: 'USD', ownerId: 1 },
    ]),
    selectedAccount: signal(undefined),
    loadAccounts: vi.fn(),
  };

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [TransactionSearchComponent],
      providers: [
        provideExperimentalZonelessChangeDetection(),
        { provide: AccountService, useValue: mockAccountService },
      ],
    });
  });

  it('displays the account selector with available accounts', () => {
    const fixture = TestBed.createComponent(TransactionSearchComponent);
    fixture.detectChanges();

    const options = fixture.nativeElement.querySelectorAll('option');
    expect(options.length).toBe(2);
    expect(options[0].textContent).toContain('Checking');
  });
});
```

The `{ provide: AccountService, useValue: mockAccountService }` syntax replaces the real service with a plain object. Signal properties return canned data; methods are replaced with `vi.fn()` so you can assert on calls later.

### Mocking Services on the Component Level

Sometimes a service is provided directly on the component rather than at the module or route level. In that case, `TestBed`'s provider override will not reach it. Use `overrideComponent` instead:

```typescript
TestBed.configureTestingModule({
  imports: [TransactionSearchComponent],
  providers: [provideExperimentalZonelessChangeDetection()],
})
.overrideComponent(TransactionSearchComponent, {
  set: {
    providers: [
      { provide: AccountService, useValue: mockAccountService },
    ],
  },
});
```

This replaces the component's own `providers` array, ensuring your mock is injected even when the component declares its own instance.

### Mocking Child Components and Shallow Testing

A component under test may render child components with their own complex dependency trees. If you only want to test the parent's behavior, mock the children:

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-transaction-card',
  standalone: true,
  template: '<div class="mock-transaction-card"></div>',
})
class MockTransactionCardComponent {
  transaction = input.required<Transaction>();
}

beforeEach(() => {
  TestBed.configureTestingModule({
    imports: [TransactionListComponent, MockTransactionCardComponent],
    providers: [provideExperimentalZonelessChangeDetection()],
  })
  .overrideComponent(TransactionListComponent, {
    remove: { imports: [TransactionCardComponent] },
    add: { imports: [MockTransactionCardComponent] },
  });
});
```

This technique -- **shallow testing** -- isolates the parent from its children. The mock child component matches the same selector and input signature, so the parent's template compiles, but none of the child's internal logic runs.

Shallow testing is valuable when child components have heavy setup requirements (HTTP calls, store initialization) that are irrelevant to the behavior you are testing in the parent.

### Mocking HTTP Calls

Angular provides `provideHttpClientTesting()` for intercepting HTTP requests in tests. This replaces the real `HttpClient` backend with a mock that lets you flush responses manually:

```typescript
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('TransactionApiService', () => {
  let service: TransactionApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideExperimentalZonelessChangeDetection(),
        provideHttpClient(),
        provideHttpClientTesting(),
        TransactionApiService,
      ],
    });
    service = TestBed.inject(TransactionApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('fetches transactions for an account', () => {
    const mockTransactions = [
      { id: 1, accountId: 10, amount: -45, type: 'debit', category: 'food',
        date: '2026-03-15', description: 'Grocery Store', pending: false },
    ];

    service.getByAccount(10).subscribe(transactions => {
      expect(transactions).toEqual(mockTransactions);
    });

    const req = httpMock.expectOne('/api/accounts/10/transactions');
    expect(req.request.method).toBe('GET');
    req.flush(mockTransactions);
  });

  afterEach(() => {
    httpMock.verify();
  });
});
```

The `httpMock.verify()` call in `afterEach` ensures no unexpected requests were made. If your code fires an HTTP call you did not anticipate, the test fails -- catching accidental network traffic early.

### Gray-Box Testing with Spies

Spies let you observe interactions between collaborators without replacing the entire implementation. Use `vi.spyOn()` to wrap a real method with observation logic:

```typescript
import { vi, describe, it, expect } from 'vitest';

it('calls loadAccounts on initialization', () => {
  const loadSpy = vi.spyOn(mockAccountService, 'loadAccounts');

  const fixture = TestBed.createComponent(TransactionSearchComponent);
  fixture.detectChanges();

  expect(loadSpy).toHaveBeenCalledOnce();
});

it('calls searchTransactions with the correct filters', () => {
  const searchSpy = vi.spyOn(transactionStore, 'search');

  fixture.componentRef.setInput('initialQuery', 'rent');
  fixture.detectChanges();
  triggerSearch(fixture);

  expect(searchSpy).toHaveBeenCalledWith({
    query: 'rent',
    accountId: undefined,
    dateRange: undefined,
  });
});
```

Gray-box testing -- verifying *how* collaborators are called rather than just the final output -- is useful when the observable output is indirect (e.g., a side effect that updates a store or navigates to a route).

### Testing Routed Components

Components that depend on route parameters need a test harness that provides `ActivatedRoute`. Use `RouterModule.forRoot()` or Angular's `provideRouter()` with a test route configuration:

```typescript
import { provideRouter } from '@angular/router';
import { Router } from '@angular/router';

describe('TransactionDetailComponent', () => {
  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideExperimentalZonelessChangeDetection(),
        provideRouter([
          { path: 'transactions/:id', component: TransactionDetailComponent },
        ]),
        { provide: TransactionStore, useValue: mockTransactionStore },
      ],
    });
  });

  it('loads the transaction matching the route parameter', async () => {
    const router = TestBed.inject(Router);
    const fixture = TestBed.createComponent(TransactionDetailComponent);

    await router.navigate(['/transactions', '42']);
    fixture.detectChanges();

    expect(mockTransactionStore.loadById).toHaveBeenCalledWith('42');
  });
});
```

By setting up a real router with a test route, you exercise the same parameter extraction logic that runs in production. This catches bugs in `input()` bindings wired to route parameters via `withComponentInputBinding()`.

---

## Testing Timers and Debouncing

The FinancialApp's transaction search uses a debounced input: the user types a query, and the search fires only after 300ms of inactivity. Testing time-dependent behavior without fake timers leads to slow, flaky tests that use real `setTimeout` delays.

### Mocking Asynchronicity and Delays

The naive approach -- using `setTimeout` in tests -- is fragile:

```typescript
// DO NOT do this in real tests
it('searches after a delay', (done) => {
  triggerInput(fixture, 'groceries');
  setTimeout(() => {
    expect(searchSpy).toHaveBeenCalled();
    done();
  }, 350);
});
```

This test takes 350ms to run, and if the debounce value changes to 500ms, the test breaks silently. Fake timers solve both problems.

### Fake Timers

Vitest provides `vi.useFakeTimers()` to replace the global timer functions (`setTimeout`, `setInterval`, `Date.now`) with controllable fakes:

```typescript
import { vi, describe, it, expect, beforeEach, afterEach } from 'vitest';

describe('TransactionSearchComponent (debounce)', () => {
  beforeEach(() => {
    vi.useFakeTimers();
    // ... TestBed setup
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('does not search before the debounce period elapses', () => {
    triggerInput(fixture, 'groceries');
    vi.advanceTimersByTime(200);
    fixture.detectChanges();

    expect(searchSpy).not.toHaveBeenCalled();
  });

  it('searches after the debounce period elapses', () => {
    triggerInput(fixture, 'groceries');
    vi.advanceTimersByTime(300);
    fixture.detectChanges();

    expect(searchSpy).toHaveBeenCalledWith(
      expect.objectContaining({ query: 'groceries' })
    );
  });

  it('resets the debounce timer on subsequent keystrokes', () => {
    triggerInput(fixture, 'gro');
    vi.advanceTimersByTime(200);
    triggerInput(fixture, 'groceries');
    vi.advanceTimersByTime(300);
    fixture.detectChanges();

    expect(searchSpy).toHaveBeenCalledOnce();
    expect(searchSpy).toHaveBeenCalledWith(
      expect.objectContaining({ query: 'groceries' })
    );
  });
});
```

`vi.advanceTimersByTime(ms)` fast-forwards the clock by the specified duration, executing any pending timers along the way. Tests run instantly regardless of the debounce duration. `vi.useRealTimers()` in `afterEach` restores normal timer behavior so subsequent test suites are not affected.

---

## Testing Services

Services that manage state through signals can be tested without `TestBed` when they have no Angular-specific dependencies. For services that use `inject()`, `TestBed` is required.

The `AccountService` from [Chapter 5](ch05-state-management.md) manages a list of accounts and exposes them as signals:

```typescript
import { TestBed } from '@angular/core/testing';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';
import { describe, it, expect, beforeEach } from 'vitest';
import { AccountService } from './account.service';

describe('AccountService', () => {
  let service: AccountService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideExperimentalZonelessChangeDetection(),
        provideHttpClient(),
        provideHttpClientTesting(),
        AccountService,
      ],
    });
    service = TestBed.inject(AccountService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('starts with an empty account list', () => {
    expect(service.accounts()).toEqual([]);
  });

  it('populates accounts after loading', () => {
    const mockAccounts = [
      { id: 1, name: 'Checking', type: 'checking', balance: 2500, currency: 'USD', ownerId: 1 },
    ];

    service.loadAccounts();

    const req = httpMock.expectOne('/api/accounts');
    req.flush(mockAccounts);

    expect(service.accounts()).toEqual(mockAccounts);
  });

  it('selects an account by id', () => {
    service.loadAccounts();
    httpMock.expectOne('/api/accounts').flush([
      { id: 1, name: 'Checking', type: 'checking', balance: 2500, currency: 'USD', ownerId: 1 },
      { id: 2, name: 'Savings', type: 'savings', balance: 10000, currency: 'USD', ownerId: 1 },
    ]);

    service.selectAccount(2);
    expect(service.selectedAccount()?.name).toBe('Savings');
  });
});
```

Signal-based services make assertions natural: read the signal, assert on its value. There is no need to subscribe to observables or manage subscriptions in your tests.

---

## Determining Test Coverage

Code coverage measures how much of your source code is exercised by tests. Vitest integrates with `v8` (default) or `istanbul` coverage providers:

```bash
# generate a coverage report
npx ng test --coverage

# or, using Vitest directly
npx vitest run --coverage
```

Configure coverage thresholds in `vitest.config.ts` to enforce minimum standards:

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      include: ['src/app/**/*.ts'],
      exclude: ['**/*.spec.ts', '**/*.stories.ts', '**/index.ts'],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      },
    },
  },
});
```

The `text` reporter prints a summary table to the terminal. The `html` reporter generates a browsable report you can open locally. The `lcov` reporter produces output compatible with CI tools like SonarQube.

A few guidelines on coverage:

- **Coverage measures execution, not correctness.** A line can be "covered" by a test that never asserts anything meaningful. Aim for meaningful assertions, not just line counts.
- **100% coverage is not the goal.** Diminishing returns set in around 80-90%. Focus coverage effort on business-critical paths -- in the FinancialApp, that means transaction processing and account balance calculations, not layout components.
- **Use thresholds to prevent regression.** Once your project reaches 80% coverage, set the threshold to 80%. This prevents gradual erosion as new code is added without tests.

---

## Summary

Angular v21 makes Vitest the standard test runner, bringing ESM-native speed and a modern API to Angular testing. The key ideas from this chapter:

- **Vitest tests** use `describe`, `it`, and `expect` from the `vitest` package. Mocking uses `vi.fn()` and `vi.spyOn()` -- not Jasmine equivalents.
- **TestBed** configures a minimal Angular environment for each test suite. In zoneless mode, change detection is explicit and deterministic.
- **Signal inputs** are set via `componentRef.setInput()`, and signal-based services make assertions straightforward -- read the signal, assert the value.
- **Mocking** replaces real dependencies with controlled substitutes. Mock at the `TestBed` level for injected services, use `overrideComponent` for component-level providers, and use `provideHttpClientTesting()` for HTTP calls.
- **Shallow testing** with mock child components isolates the unit under test from its children's dependency trees.
- **Fake timers** (`vi.useFakeTimers()`, `vi.advanceTimersByTime()`) eliminate real delays from tests, making time-dependent behavior deterministic and fast.
- **Coverage** should be measured, enforced with thresholds, and focused on business-critical paths rather than vanity metrics.

Spec files for every component, service, and store in the FinancialApp companion codebase demonstrate these techniques in context. In [Chapter 11](ch09-architecture.md), we will explore how to structure your application so that the boundaries you test against remain stable as the codebase scales.
