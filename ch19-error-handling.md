# Chapter 40: Error Handling

Errors are inevitable in a financial application. The API times out mid-transaction. The network drops while loading portfolio holdings. A validation rule rejects a transfer amount that looked correct on the client. The question is never whether errors will occur, but how each layer of the application responds when they do.

Angular's error handling philosophy follows a clear hierarchy: handle errors at the callsite whenever possible, let interceptors and resource APIs manage HTTP-level concerns, and reserve the global `ErrorHandler` as a last-resort safety net for truly unexpected failures. This chapter walks through each layer -- from the global handler down to inline error UI -- using FinancialApp as the proving ground.

> **Forward reference:** The `ErrorBoundary` component and error display patterns built here are consumed by [Chapter 32](ch22-material-design-system.md)'s `FinFormField` wrapper, which uses them for inline validation error display.

---

## Angular's ErrorHandler

Every uncaught error in an Angular application eventually reaches the `ErrorHandler` class. The default implementation logs the error to the console and stops there. For a development build this is fine -- you see the stack trace in DevTools and fix it. In production, a console log disappears the moment the user closes the tab. No one is notified, no data is captured, and the same bug reappears without context.

### Custom GlobalErrorHandler

A custom `ErrorHandler` replaces the default with structured logging and analytics reporting. The FinancialApp sends every uncaught error to a centralized analytics service with enough context to reproduce the issue:

```typescript
// libs/shared/util-error/src/lib/global-error-handler.ts
import { ErrorHandler, Injectable, inject } from '@angular/core';
import { AnalyticsService } from '@financial-app/shared/util-analytics';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private analytics = inject(AnalyticsService);

  handleError(error: unknown): void {
    const resolved = this.unwrap(error);

    console.error('[GlobalErrorHandler]', resolved);

    this.analytics.captureException(resolved, {
      timestamp: Date.now(),
      url: window?.location?.href,
      userAgent: navigator?.userAgent,
    });
  }

  private unwrap(error: unknown): Error {
    if (error instanceof Error) {
      return error.cause instanceof Error ? error.cause : error;
    }
    return new Error(String(error));
  }
}
```

The `unwrap` method handles a subtlety: Angular sometimes wraps the original error inside another `Error` via the `cause` property. Extracting the root cause gives the analytics service a cleaner stack trace.

Register the handler in the application configuration:

```typescript
// apps/financial-app/src/app/app.config.ts
import { ApplicationConfig, ErrorHandler } from '@angular/core';
import { GlobalErrorHandler } from '@financial-app/shared/util-error';

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler },
    // ...other providers
  ],
};
```

This replaces the default for the entire application. Every component, service, and pipe that throws an unhandled error will flow through `GlobalErrorHandler.handleError()`.

---

## Global Error Listeners

Angular's `ErrorHandler` catches errors that originate inside the framework's execution context -- template expressions, lifecycle hooks, event handlers. It does not catch errors that escape Angular entirely: a `setTimeout` callback that throws, a promise rejection that no one handles, or an error in third-party code that runs outside Angular's awareness.

Angular v19+ provides `provideBrowserGlobalErrorListeners()` to close this gap. It installs `window.onerror` and `window.onunhandledrejection` listeners that forward uncaught errors and unhandled promise rejections to the application's `ErrorHandler`:

```typescript
// apps/financial-app/src/app/app.config.ts
import { provideBrowserGlobalErrorListeners } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler },
    provideBrowserGlobalErrorListeners(),
    // ...other providers
  ],
};
```

With this in place, an unhandled `fetch()` rejection in a third-party charting library still reaches `GlobalErrorHandler`, gets logged, and gets sent to analytics.

On the server side, Angular's SSR runtime automatically installs `unhandledRejection` and `uncaughtException` listeners on the Node.js process. These forward server-side errors to the same `ErrorHandler`, giving you a unified error pipeline across both environments. See [Chapter 22](ch31-defer-ssr-hydration.md) for more on SSR architecture.

---

## Error Handling at the Callsite

The global handler is a safety net, not a strategy. The preferred approach is to handle errors where they happen -- in the service method, the component, or the template -- where you have enough context to display a meaningful message or attempt recovery.

### Async Methods with try/catch

When a service method performs an asynchronous operation, wrapping it in `try/catch` lets you transform opaque HTTP errors into user-friendly messages before the component sees them:

```typescript
// libs/features/transactions/src/lib/transaction.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class TransactionService {
  private http = inject(HttpClient);

  async submitTransaction(tx: TransactionRequest): Promise<TransactionResult> {
    try {
      return await firstValueFrom(
        this.http.post<TransactionResult>('/api/transactions', tx)
      );
    } catch (err) {
      if (err instanceof HttpErrorResponse) {
        if (err.status === 422) {
          throw new Error(
            err.error?.message ?? 'Transaction failed validation.'
          );
        }
        if (err.status === 409) {
          throw new Error('Duplicate transaction detected. Please review your recent activity.');
        }
      }
      throw new Error('Unable to submit transaction. Please try again.');
    }
  }
}
```

The component calling `submitTransaction()` receives a descriptive error message it can display directly, rather than a raw `HttpErrorResponse` with a status code the user cannot interpret.

### RxJS catchError

For service methods that return observables, the `catchError` operator serves the same purpose. The FinancialApp's account service normalizes API errors before they reach subscribers:

```typescript
// libs/features/accounts/src/lib/account.service.ts
getAccountDetails(id: string): Observable<Account> {
  return this.http.get<Account>(`/api/accounts/${id}`).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 404) {
        return throwError(() => new Error('Account not found.'));
      }
      return throwError(() => new Error('Failed to load account details.'));
    })
  );
}
```

The pattern is the same in both styles: intercept the error at the boundary between the HTTP layer and the business logic, and replace it with something the UI can present.

---

## Resource API Error States

Angular's `resource()` and `httpResource()` APIs provide built-in error state management that eliminates most of the boilerplate shown in the previous section. A resource exposes reactive signals for its status, value, and error, making template-level error handling declarative.

The FinancialApp's portfolio detail page loads holdings through an `httpResource`:

```typescript
// libs/features/portfolios/src/lib/portfolio-detail.component.ts
@Component({
  selector: 'app-portfolio-detail',
  template: `
    @if (holdingsResource.isLoading()) {
      <mat-spinner diameter="48" />
    } @else if (holdingsResource.error()) {
      <div class="error-panel" role="alert">
        <p>Could not load holdings for this portfolio.</p>
        <button mat-stroked-button (click)="holdingsResource.reload()">
          Retry
        </button>
      </div>
    } @else if (holdingsResource.hasValue()) {
      <app-holdings-table [holdings]="holdingsResource.value()!" />
    }
  `,
})
export class PortfolioDetailComponent {
  portfolioId = input.required<string>();

  holdingsResource = httpResource<Holding[]>(() => ({
    url: `/api/portfolios/${this.portfolioId()}/holdings`,
  }));
}
```

The `httpResource` handles the HTTP call, tracks loading and error states, and re-executes when `portfolioId` changes. The template checks `isLoading()`, `error()`, and `hasValue()` -- no subscription management, no imperative state tracking. The `reload()` method lets the user retry after a failure without navigating away.

This pattern scales across the application. Every data-fetching component that uses `httpResource` gets consistent error handling for free, and the error UI stays co-located with the data display rather than buried in a service layer. For more on the resource API, see [Chapter 3](ch03-reactive-signals.md).

---

## HTTP Error Interceptors

While callsite handling covers known error scenarios, some HTTP concerns are better handled centrally. Retry logic is the clearest example: every API call in FinancialApp should retry on transient server errors, and duplicating that logic across dozens of service methods would be maintenance poison.

### Retry Interceptor with Exponential Backoff

An `HttpInterceptorFn` can inspect the error and decide whether to retry. Server errors (5xx) and network failures are generally safe to retry. Client errors (4xx) are not -- retrying a `400 Bad Request` or `403 Forbidden` will produce the same result.

```typescript
// libs/shared/util-http/src/lib/retry.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { retry, timer } from 'rxjs';

const MAX_RETRIES = 3;
const INITIAL_DELAY_MS = 500;

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.method !== 'GET') {
    return next(req);
  }

  return next(req).pipe(
    retry({
      count: MAX_RETRIES,
      delay: (error, retryCount) => {
        if (error instanceof HttpErrorResponse && error.status < 500) {
          throw error;
        }
        const delayMs = INITIAL_DELAY_MS * Math.pow(2, retryCount - 1);
        return timer(delayMs);
      },
    })
  );
};
```

The interceptor only retries `GET` requests. Retrying a `POST` that creates a transaction could produce duplicates -- a catastrophic outcome for a financial application. The `delay` callback uses exponential backoff: 500ms, 1000ms, 2000ms. If the error is a 4xx, it re-throws immediately to skip retries.

Register the interceptor alongside any existing ones in [Chapter 48](ch13-initialization-routes.md)'s configuration:

```typescript
// apps/financial-app/src/app/app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { retryInterceptor } from '@financial-app/shared/util-http';
import { authTokenInterceptor } from '@financial-app/shared/util-auth';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authTokenInterceptor, retryInterceptor])
    ),
    // ...other providers
  ],
};
```

Interceptors run in array order. Place `authTokenInterceptor` first so retried requests carry a valid token.

---

## TestBed Error Behavior

Angular's `TestBed` rethrows application errors by default. If a component under test throws during change detection, the test fails immediately with the original error. This is the right default -- it surfaces bugs early and produces clear test output.

Occasionally you need to test that a component *handles* an error gracefully rather than crashing. The `rethrowApplicationErrors` option controls this behavior:

```typescript
// libs/features/portfolios/src/lib/portfolio-detail.component.spec.ts
import { TestBed } from '@angular/core/testing';

describe('PortfolioDetailComponent error resilience', () => {
  beforeEach(() => {
    TestBed.configureTestingModule({
      rethrowApplicationErrors: false,
    });
  });

  it('should render error panel when holdings fail to load', async () => {
    // Arrange: configure httpResource to fail
    // Act: trigger change detection
    // Assert: error panel is visible, no unhandled exception
  });
});
```

Setting `rethrowApplicationErrors: false` tells TestBed to swallow errors through the `ErrorHandler` instead of rethrowing them. Use this sparingly -- most tests should let errors propagate so broken code does not hide behind a passing test suite. See [Chapter 7](ch07-testing-vitest.md) for the broader testing strategy.

---

## User-Facing Error UI Patterns

Error handling does not end when the error is caught. The user needs to know what happened and what they can do about it. FinancialApp uses three patterns, each suited to a different failure mode.

### Inline Errors

Inline errors display directly next to the component that failed. The holdings table shows an error panel instead of an empty table. The transaction form shows validation feedback beneath the offending field. This is the most informative pattern because it preserves the user's spatial context -- they can see exactly what broke without losing their place.

The `httpResource` template pattern shown earlier is a textbook inline error. The error message and retry button appear where the data would have been, and the rest of the page remains functional.

### Toast Notifications

Transient errors that do not block the primary workflow belong in toast notifications. A brief network hiccup during a background data refresh should not replace the user's entire view with an error panel. A toast appears, informs, and disappears.

```typescript
// libs/shared/ui/src/lib/error-toast.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface ErrorToast {
  id: number;
  message: string;
  severity: 'error' | 'warning';
}

@Injectable({ providedIn: 'root' })
export class ErrorToastService {
  private queue = signal<ErrorToast[]>([]);
  private nextId = 0;

  readonly toasts = computed(() => this.queue());

  show(message: string, severity: 'error' | 'warning' = 'error'): void {
    const id = this.nextId++;
    this.queue.update((q) => [...q, { id, message, severity }]);

    setTimeout(() => this.dismiss(id), 5000);
  }

  dismiss(id: number): void {
    this.queue.update((q) => q.filter((t) => t.id !== id));
  }
}
```

The service uses a signal-based queue. Components that need to report transient errors inject `ErrorToastService` and call `show()`. A root-level toast container component reads the `toasts` signal and renders them:

```typescript
// libs/shared/ui/src/lib/error-toast-container.component.ts
@Component({
  selector: 'app-error-toast-container',
  template: `
    <div class="toast-stack" aria-live="polite">
      @for (toast of toastService.toasts(); track toast.id) {
        <div class="toast" [class]="toast.severity" role="alert">
          <span>{{ toast.message }}</span>
          <button (click)="toastService.dismiss(toast.id)" aria-label="Dismiss">
            &times;
          </button>
        </div>
      }
    </div>
  `,
})
export class ErrorToastContainerComponent {
  protected toastService = inject(ErrorToastService);
}
```

Place `<app-error-toast-container />` in the root `AppComponent` template. The `aria-live="polite"` attribute ensures screen readers announce new toasts without interrupting the user's current task.

### Error Boundaries

Some errors are unrecoverable at the component level. A third-party charting library throws during initialization. A child component enters an invalid state. In these cases, you want to catch the error, display a fallback, and prevent the failure from cascading to the rest of the page.

An error boundary component wraps its children and catches errors during rendering:

```typescript
// libs/shared/ui/src/lib/error-boundary.component.ts
import { Component, input, signal, ErrorHandler, inject } from '@angular/core';

@Component({
  selector: 'app-error-boundary',
  template: `
    @if (hasError()) {
      <div class="error-boundary" role="alert">
        <h3>Something went wrong</h3>
        <p>{{ fallbackMessage() }}</p>
        <button mat-stroked-button (click)="reset()">Try Again</button>
      </div>
    } @else {
      <ng-content />
    }
  `,
})
export class ErrorBoundaryComponent {
  fallbackMessage = input('This section could not be loaded.');

  hasError = signal(false);
  private errorHandler = inject(ErrorHandler);

  handleChildError(error: unknown): void {
    this.hasError.set(true);
    this.errorHandler.handleError(error);
  }

  reset(): void {
    this.hasError.set(false);
  }
}
```

Parent components wrap unreliable sections in the boundary:

```html
<!-- portfolio-overview.component.html -->
<app-error-boundary fallbackMessage="Could not render the performance chart.">
  <app-performance-chart [data]="performanceData()" />
</app-error-boundary>
```

If `app-performance-chart` throws, the error boundary replaces it with a descriptive message and a retry button. The rest of the portfolio overview -- account summary, holdings table, transaction history -- remains fully functional. The error still flows to `GlobalErrorHandler` for analytics capture, but the user sees a graceful degradation instead of a blank screen.

### Choosing the Right Pattern

The three patterns are not mutually exclusive. A single page might use all three:

| Pattern          | When to use                                           | Example                                     |
| ---------------- | ----------------------------------------------------- | ------------------------------------------- |
| Inline error     | The failed data is the primary content of the section | Holdings table fails to load                |
| Toast            | The failure is transient or secondary to the workflow | Background refresh fails, auto-retrying     |
| Error boundary   | A child component crashes unpredictably               | Third-party chart library throws at runtime |

Inline errors are the default choice. They provide the most context and the clearest recovery path. Toasts work for non-blocking notifications. Error boundaries are the last line of defense before the global handler.

---

## Summary

Error handling is a spectrum, not a single solution. Each layer serves a distinct purpose, and the layers compose into a resilient whole:

- **`ErrorHandler`** is the global safety net. Replace the default with a custom `GlobalErrorHandler` that captures errors, logs structured data, and reports to analytics. Provide it via `{ provide: ErrorHandler, useClass: GlobalErrorHandler }`.
- **`provideBrowserGlobalErrorListeners()`** closes the gap between Angular's execution context and the raw browser environment, forwarding uncaught errors and unhandled promise rejections to the `ErrorHandler`.
- **Callsite handling** is the preferred strategy. Use `try/catch` in async methods and `catchError` in RxJS pipelines to transform raw HTTP errors into user-friendly messages at the boundary between the HTTP layer and the business logic.
- **Resource APIs** (`resource()` and `httpResource()`) provide built-in error state management via `status()`, `error()`, `hasValue()`, and `reload()`, making template-level error handling declarative.
- **HTTP interceptors** handle cross-cutting concerns like retry logic. The `retryInterceptor` retries idempotent requests with exponential backoff, distinguishing transient server errors from permanent client errors.
- **TestBed** rethrows errors by default. Set `rethrowApplicationErrors: false` only when testing error-resilient behavior explicitly.
- **User-facing UI** presents errors through inline panels (co-located with the failed content), toast notifications (transient, non-blocking), and error boundaries (fallback UI for unpredictable child failures).

The goal is not to prevent errors -- that is impossible in a distributed system. The goal is to ensure that every error is caught at the right level, reported with enough context to fix, and presented to the user in a way that lets them continue working. Start at the callsite, let each layer above handle what the layer below cannot, and treat the global handler as the last resort it was designed to be.

> **Companion code:** `GlobalErrorHandler`, `retryInterceptor`, `ErrorBoundaryComponent`, and `ErrorToastService` live in `financial-app/libs/shared/ui/` and `financial-app/apps/financial-app/src/app/shared/`. Unit tests follow [Chapter 7](ch07-testing-vitest.md) patterns.
