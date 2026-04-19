# Chapter 17: Observability & Production Monitoring

If an error happens in production and no one saw it, it happened twice -- once to the user who hit it, and again to the next user who walks into the same bug tomorrow. Observability closes that gap. It is the difference between "a customer called to say the transfer form is broken" and "error rate on `/transfers` jumped 4x at 14:02 UTC, 87% of affected users are on Safari 17.2, here is the stack trace." The former is a support ticket. The latter is a bug you can fix before lunch.

For a financial application, the stakes are higher than user satisfaction. A silent regression in portfolio valuation, a slow degradation in login success rate, a memory leak that only surfaces after fifteen minutes on the transactions page -- these turn into regulatory headaches long before anyone complains. [Chapter 40](ch19-error-handling.md) built the error-handling scaffolding inside FinancialApp: a `GlobalErrorHandler`, global browser error listeners, interceptors that normalize HTTP failures. This chapter takes the next step. We ship those errors, performance data, and business telemetry to a system that can alert a human.

> **Companion code:** `financial-app/libs/shared/observability/` with `TelemetryService`, Sentry integration, correlation-id interceptor, and web-vitals reporter.

---

## The Three Pillars in a Browser

Backend observability is described in terms of three pillars: logs, metrics, and traces. Each has a browser-side counterpart.

**Logs** are discrete events with a message and structured context -- a click, a failed form submission, a route transition. On the server they write to stdout and get shipped to Splunk or Loki; in the browser they batch into `sendBeacon` or `fetch` calls. Every log needs at minimum a timestamp, a session identifier, the application release, and the route -- a log line without context is a whodunit with no evidence.

**Metrics** are numeric measurements aggregated over time: API latency percentiles, error rate per route, transactions submitted per hour. Metrics are cheaper than logs because they compress -- a million `transaction_submit` events become a single counter. Core Web Vitals ([Chapter 9](ch32-performance.md)) are metrics reported from the browser about how the page *feels*.

**Traces** connect related operations across service boundaries. A user clicks *Submit Transfer*; the browser calls `/api/transfers`, which calls the ledger service, which calls fraud detection. A trace stitches all of it into a single timeline so you can see where the 3.2 seconds went. The browser is the origin of most user-facing traces -- if the trace does not start there, you are debugging distributed systems blindfolded.

Sentry is strongest for logs and traces; Datadog RUM excels at metrics and session replay; OpenTelemetry is a vendor-neutral wire format for all three. Most teams end up with some combination, and `libs/shared/observability/` slots any of them behind a single `TelemetryService` facade.

---

## Sentry Integration with Angular

Sentry is the default for Angular error monitoring -- `@sentry/angular` ships a first-party integration and its stack-trace grouping is unusually good.

```bash
npm install @sentry/angular
```

Initialize Sentry before `bootstrapApplication()` so the SDK is in place to capture errors thrown during bootstrap itself:

```typescript
// apps/financial-app/src/main.ts
import * as Sentry from '@sentry/angular';
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';
import { environment } from './environments/environment';

Sentry.init({
  dsn: environment.sentryDsn,
  environment: environment.name,
  release: environment.release,
  tracesSampleRate: environment.name === 'production' ? 0.1 : 1.0,
  replaysSessionSampleRate: 0.01,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration({ maskAllText: true, blockAllMedia: true }),
  ],
});

bootstrapApplication(AppComponent, appConfig);
```

`tracesSampleRate: 0.1` captures 10% of transactions for performance tracing -- enough for statistical accuracy, few enough to keep quota costs predictable. Replays are sampled at 1% normally and 100% on error. `maskAllText` and `blockAllMedia` are non-negotiable for a financial application; more on PII below.

### Providers and Composition

The Angular integration wires Sentry into the router and the framework's error-handling pipeline. [Chapter 40](ch19-error-handling.md) already built a `GlobalErrorHandler` that does structured logging and dispatches to an `AnalyticsService`. We do not want to discard it because we added Sentry -- the clean pattern is composition: the application handler runs first and then hands the error to Sentry.

```typescript
// libs/shared/observability/src/lib/composed-error-handler.ts
import { ErrorHandler, Injectable, inject } from '@angular/core';
import * as Sentry from '@sentry/angular';
import { GlobalErrorHandler } from '@financial-app/shared/util-error';

@Injectable()
export class ComposedErrorHandler implements ErrorHandler {
  private appHandler = inject(GlobalErrorHandler);
  private sentryHandler = Sentry.createErrorHandler({ showDialog: false });

  handleError(error: unknown): void {
    try {
      this.appHandler.handleError(error);
    } catch (inner) {
      console.error('[ComposedErrorHandler] app handler threw', inner);
    }
    this.sentryHandler.handleError(error);
  }
}
```

Register `ComposedErrorHandler` and `Sentry.TraceService` together:

```typescript
// apps/financial-app/src/app/app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: ErrorHandler, useClass: ComposedErrorHandler },
    { provide: Sentry.TraceService, deps: [Router] },
    {
      provide: APP_INITIALIZER,
      useFactory: () => () => {},
      deps: [Sentry.TraceService],
      multi: true,
    },
  ],
};
```

The `try/catch` around `appHandler` is deliberate -- if the application handler throws because the analytics endpoint is down, Sentry still sees the original error. The no-op `APP_INITIALIZER` is the documented way to force Angular to eagerly instantiate `TraceService`; without it, the service never constructs and no route transactions report.

### Performance Tracing

With `browserTracingIntegration()` enabled, Sentry starts a transaction for each route change and each `HttpClient` request. A route transition produces a span tree: `router.navigate` → `http.client GET /api/portfolios/12345` → `http.client GET /portfolios/12345/holdings` → component render. Sequential, serial HTTP calls are the most common finding. When the portfolio page takes four seconds and the flame chart shows two 800ms spans running back-to-back instead of in parallel, that is a bug worth fixing.

Add custom spans for operations the framework does not instrument:

```typescript
async loadPortfolioSummary(id: string) {
  return Sentry.startSpan({ name: 'portfolio.summary.compute' }, async () => {
    const accounts = await this.accountService.list(id);
    const holdings = await this.holdingService.list(id);
    return this.aggregator.compute(accounts, holdings);
  });
}
```

### Releases and Source Maps

A stack trace from a minified bundle is useless -- `a.b.c is not a function at r.prototype.ng (chunk-4c2f.js:1:23412)` tells you nothing. Source maps translate minified frames back into TypeScript, and Sentry applies the translation automatically if you upload maps tagged with the matching release. The release identifier must be the same string in three places: `Sentry.init()`'s `release` property, `sentry-cli releases new`, and `sentry-cli sourcemaps upload`. The natural candidate is the Git commit SHA.

```bash
# .github/workflows/deploy.yml (excerpt)
- run: |
    sentry-cli releases new "${GITHUB_SHA}"
    sentry-cli sourcemaps upload \
      --release "${GITHUB_SHA}" --url-prefix "~/assets/" \
      dist/financial-app/browser
    sentry-cli releases finalize "${GITHUB_SHA}"
```

CI details -- environment promotion, artifact signing, staged rollouts -- belong in [Chapter 34](ch41-ci-cd.md). The essential point: no release is complete until its maps have been uploaded. Configure the build to emit maps as hidden files (`"sourceMap": { "scripts": true, "hidden": true }` in `angular.json`) so Sentry gets them at upload time but end users never do.

---

## Datadog RUM as an Alternative

If the team already lives in Datadog dashboards for backend services, keeping frontend data in the same pane of glass is a strong argument for using RUM instead of Sentry. The feature overlap is significant -- errors, Core Web Vitals, session replays -- but Sentry is error-centric while Datadog RUM is session-centric.

Datadog offers two installation modes. The **script tag** loads the SDK from Datadog's CDN at runtime:

```html
<script src="https://www.datadoghq-browser-agent.com/us1/v5/datadog-rum.js" defer></script>
<script>
  window.DD_RUM && window.DD_RUM.init({
    clientToken: 'pub_xxx',
    applicationId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
    site: 'datadoghq.com',
    service: 'financial-app-web',
    env: 'production',
    version: '__RELEASE__',
    sessionSampleRate: 100,
    sessionReplaySampleRate: 1,
    defaultPrivacyLevel: 'mask',
  });
</script>
```

The script tag decouples SDK updates from application releases -- Datadog can push bug fixes without a redeploy. The downside is a runtime request to a third-party domain, which adds latency to first paint and requires a `script-src` entry in your CSP ([Chapter 35](ch18-security-owasp.md)). The **NPM package** (`@datadog/browser-rum`) bundles the SDK with your app -- a few kilobytes of first-party JavaScript -- but removes the third-party request and keeps the SDK version locked to a release you audit explicitly. For a regulated application, the audit story matters.

RUM is billed per session, where a session is "user activity ending after 15 minutes of inactivity." Replays are the most expensive part of the bill -- `sessionReplaySampleRate: 1` means 1% of RUM-sampled sessions record replay. Most teams start at 100% RUM / 1-5% replay and raise replay only if coverage proves insufficient.

---

## OpenTelemetry for the Browser

OpenTelemetry (OTEL) is the CNCF's vendor-neutral observability specification. The browser SDK produces the same trace format your backend Go or Java services produce, so a single trace can span from a click in Angular, through your API gateway, through three downstream services, and back -- all correlated by one trace ID. Pick OTEL when you already run an OTEL collector, you want to avoid commercial lock-in, or you specifically need distributed traces that cross the browser/server boundary cleanly.

```bash
npm install @opentelemetry/sdk-trace-web \
  @opentelemetry/instrumentation-fetch \
  @opentelemetry/exporter-trace-otlp-http
```

```typescript
// apps/financial-app/src/telemetry/otel.ts
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { Resource } from '@opentelemetry/resources';

export function initOpenTelemetry(): void {
  const provider = new WebTracerProvider({
    resource: new Resource({
      'service.name': 'financial-app-web',
      'service.version': __RELEASE__,
    }),
  });

  provider.addSpanProcessor(
    new BatchSpanProcessor(
      new OTLPTraceExporter({ url: 'https://otel.financialapp.com/v1/traces' })
    )
  );
  provider.register();

  registerInstrumentations({
    instrumentations: [
      new FetchInstrumentation({
        propagateTraceHeaderCorsUrls: [/^https:\/\/api\.financialapp\.com/],
      }),
    ],
  });
}
```

`FetchInstrumentation` wraps the global `fetch` so every request emits a span and carries `traceparent`/`tracestate` headers. The `propagateTraceHeaderCorsUrls` setting limits propagation to your own origins -- you do not want trace IDs leaking to third-party analytics or ad networks. Angular 18+ uses fetch under `HttpClient`, so this covers the whole app; add `XMLHttpRequestInstrumentation` if you still run the legacy XHR backend. The OTLP endpoint points at an OpenTelemetry Collector in your infrastructure that forwards to wherever backend traces live -- Tempo, Jaeger, Honeycomb, Datadog APM.

---

## Structured Logging with Correlation IDs

A production incident looks like this: a user reports "I tried to submit a transfer at 2:47 PM and got an error." The operator has Sentry, the API gateway's access log, and the transfer service's logs -- three separate systems. Without a shared identifier, they spend fifteen minutes with grep and guesswork. With a correlation ID, they query for one string and get the entire request chain.

The FinancialApp generates a correlation ID once per browser session and keeps it for the tab's lifetime. The ID is opaque, high-entropy, and carries no user identity -- a join key, not an identifier.

```typescript
// libs/shared/observability/src/lib/correlation-id.ts
@Injectable({ providedIn: 'root' })
export class CorrelationIdService {
  private readonly id = crypto.randomUUID();
  get(): string { return this.id; }
}
```

The interceptor pattern from [Chapter 48](ch13-initialization-routes.md) attaches an `X-Correlation-Id` header to every outbound request:

```typescript
// libs/shared/observability/src/lib/correlation.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { CorrelationIdService } from './correlation-id';

export const correlationInterceptor: HttpInterceptorFn = (req, next) => {
  const correlationId = inject(CorrelationIdService).get();
  return next(req.clone({ setHeaders: { 'X-Correlation-Id': correlationId } }));
};
```

Register it near the start of the interceptor chain so retried or re-authenticated requests still carry the same header. The order places `correlationInterceptor` before the auth and retry interceptors from [Chapter 27](ch17-auth-patterns.md):

```typescript
provideHttpClient(
  withInterceptors([correlationInterceptor, authTokenInterceptor, retryInterceptor])
)
```

The API gateway logs the header it receives and downstream services propagate it on RPC calls. Grepping logs for one correlation ID across browser, gateway, transfer service, and ledger service produces a complete timeline. Crucially, the correlation ID is *not* an auth token -- it never identifies a user account, only a session's worth of activity, so it is safe to log and safe to include in telemetry payloads.

### The TelemetryService Facade

Errors, structured logs, and custom events route through a single `TelemetryService`. The facade exists for testability and for vendor portability -- swapping Sentry for Datadog should be a one-file change.

```typescript
// libs/shared/observability/src/lib/telemetry.service.ts
import { Injectable, inject } from '@angular/core';
import * as Sentry from '@sentry/angular';
import { CorrelationIdService } from './correlation-id';
import { scrubPii } from './pii-scrubber';

export type LogLevel = 'debug' | 'info' | 'warn' | 'error';

@Injectable({ providedIn: 'root' })
export class TelemetryService {
  private correlationId = inject(CorrelationIdService);

  log(level: LogLevel, message: string, context: Record<string, unknown> = {}): void {
    const payload = {
      level,
      message,
      correlationId: this.correlationId.get(),
      route: window.location.pathname,
      timestamp: new Date().toISOString(),
      ...scrubPii(context),
    };

    if (level === 'error' || level === 'warn') {
      Sentry.captureMessage(message, {
        level,
        extra: payload,
        tags: { correlationId: payload.correlationId },
      });
    }
    console[level === 'debug' ? 'log' : level](`[${level}]`, message, payload);
  }

  trackEvent(name: string, properties: Record<string, unknown> = {}): void {
    const safe = scrubPii(properties);
    Sentry.addBreadcrumb({ category: 'business', message: name, data: safe });
    Sentry.captureMessage(`event:${name}`, {
      level: 'info',
      extra: { ...safe, correlationId: this.correlationId.get() },
    });
  }
}
```

Consumers inject `TelemetryService` rather than Sentry directly, which means feature code never imports the vendor SDK.

---

## Core Web Vitals in Production

[Chapter 9](ch32-performance.md) introduced Core Web Vitals as design constraints measured in the lab. In production, you measure them on real user devices, which vary more than Lighthouse can simulate. The `web-vitals` library exposes each metric as an event callback that fires when the browser has enough data.

```typescript
// libs/shared/observability/src/lib/web-vitals-reporter.ts
import { inject, Injectable } from '@angular/core';
import { onCLS, onFCP, onINP, onLCP, onTTFB, Metric } from 'web-vitals';
import { TelemetryService } from './telemetry.service';

@Injectable({ providedIn: 'root' })
export class WebVitalsReporter {
  private telemetry = inject(TelemetryService);

  start(): void {
    if (!this.shouldSample()) return;

    const report = (metric: Metric) => {
      this.telemetry.trackEvent('web_vital', {
        name: metric.name,
        value: Math.round(metric.value),
        rating: metric.rating,
        navigationType: metric.navigationType,
      });
    };

    onLCP(report);
    onINP(report);
    onCLS(report);
    onFCP(report);
    onTTFB(report);
  }

  private shouldSample(): boolean {
    return Math.random() < 0.1;
  }
}
```

Kick it off in an application initializer: `provideAppInitializer(() => inject(WebVitalsReporter).start())`.

A 10% sample rate is intentional. Reporting Core Web Vitals on every session is redundant -- the 90th percentile of LCP computed from 10,000 samples per day is effectively identical to the 90th percentile from 100,000. What low sampling loses is the ability to debug *individual* sessions, not aggregate trends. Staging with twenty internal users needs 100%; production at a hundred thousand users can drop to 1% and still get reliable percentiles. Put the rate in remote config so SREs can crank it up during an incident without shipping a release.

Each metric answers a different question: `LCP` for perceived load time, `INP` for responsiveness under interaction, `CLS` for visual stability, `FCP` for paint latency, `TTFB` for server+network. An LCP regression paired with a TTFB regression points at the backend; LCP without TTFB points at the frontend.

---

## Custom Business Metrics

Vendor-neutral telemetry for technical concerns is the floor, not the ceiling. The metrics that matter most to a business are the ones no framework knows about: transactions submitted today, users who completed onboarding, clicks on *Cancel* at the transfer-confirmation step. These are the numbers an executive asks for first thing Monday morning.

`TelemetryService.trackEvent()` is the seam. Sprinkling `trackEvent` through business code is mildly controversial -- some teams consider it infrastructure leaking into domain logic -- but the alternative is a fragile proxy layer where feature authors guess which events the product team wants. Explicit wins.

```typescript
// libs/features/transfers/src/lib/transfer.component.ts
@Component({ /* ... */ })
export class TransferComponent {
  private telemetry = inject(TelemetryService);
  private service = inject(TransferService);

  async submit(amountBucket: string, destinationType: 'internal' | 'external'): Promise<void> {
    this.telemetry.trackEvent('transfer.submitted', { amountBucket, destinationType });
    try {
      await this.service.submit();
      this.telemetry.trackEvent('transfer.succeeded', { destinationType });
    } catch (err) {
      this.telemetry.trackEvent('transfer.failed', {
        destinationType,
        reason: err instanceof Error ? err.name : 'unknown',
      });
      throw err;
    }
  }
}
```

The event names are not arbitrary: `transfer.submitted`, `transfer.succeeded`, `transfer.failed` form a funnel. Product analytics tools (Amplitude, Mixpanel, PostHog, Datadog RUM custom actions) derive drop-off rates from the sequence. Client onboarding is the canonical funnel for a financial app: `onboarding.started` → `identity.submitted` → `identity.verified` → `funding.submitted` → `completed`. A 30% drop between `identity.submitted` and `identity.verified` might mean the KYC vendor is rejecting documents, users are uploading blurry photos, or the upload UI is broken on iOS. A single complaint ticket gives you none of that information; the funnel points at the exact step to investigate.

---

## PII Scrubbing Before Transmission

Every event, log, and error that leaves the browser passes through `scrubPii()` first. [Chapter 35](ch18-security-owasp.md) covered OWASP A04 (Insecure Design); this is where that principle becomes code. The threat model: Sentry, Datadog, and your OTEL collector are third-party systems. Account numbers, transaction amounts, full names, SSNs, balances -- none belong in telemetry. An engineer with Sentry dashboard access should not be able to reconstruct a user's financial situation from error reports.

Deny-lists fail because you cannot enumerate every PII-shaped field -- engineers invent new ones next sprint. The robust approach is an allow-list: enumerate what is safe, strip everything else.

```typescript
// libs/shared/observability/src/lib/pii-scrubber.ts
const SAFE_FIELDS = new Set([
  'name', 'step', 'route', 'method', 'status', 'durationMs',
  'destinationType', 'amountBucket', 'featureFlag', 'reason',
  'variant', 'navigationType', 'rating', 'value', 'correlationId',
  'level', 'timestamp',
]);

export function scrubPii<T extends Record<string, unknown>>(input: T): Record<string, unknown> {
  const output: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(input)) {
    if (!SAFE_FIELDS.has(key)) continue;
    if (typeof value === 'string' && looksLikeSensitive(value)) continue;
    output[key] = value;
  }
  return output;
}

function looksLikeSensitive(value: string): boolean {
  return (
    /\b\d{9,}\b/.test(value) ||            // account/card-like sequences
    /\b\d{3}-\d{2}-\d{4}\b/.test(value) || // SSN
    /[\w.+-]+@[\w-]+\.[\w.-]+/.test(value) // email
  );
}
```

Notice `amountBucket`, not `amount`. The bucket is a coarse category: `'<100'`, `'100-1000'`, `'1000-10000'`, `'>10000'`. The product team still analyzes transfer sizes; telemetry never sees an actual dollar figure. Bucketing is the general pattern for any numeric value that could identify a user when combined with other signals.

Defense in depth: even with `scrubPii()` in `TelemetryService`, a third-party library or an Angular error message might embed sensitive data in an error string. Sentry's `beforeSend` hook runs on every outbound event for one last pass:

```typescript
Sentry.init({
  // ...
  beforeSend(event) {
    if (event.request?.url) {
      event.request.url = redactQueryParams(event.request.url);
    }
    if (event.exception?.values) {
      for (const ex of event.exception.values) {
        if (ex.value) ex.value = ex.value.replace(/\b\d{9,}\b/g, '[REDACTED]');
      }
    }
    return event;
  },
});
```

Query parameters are a frequent leak vector: `/api/transfers?from=1234567890&to=9876543210` puts account numbers in every HTTP span. `redactQueryParams` normalizes URLs by stripping values whose parameter names are not on an allow-list.

---

## Alerting

Telemetry without alerting is a dashboard no one looks at until after an incident. Alerting translates metrics into action.

**Error-rate thresholds** are the simplest effective alert. For FinancialApp: if the uncaught-error rate on `/transfers` exceeds three errors per minute for five consecutive minutes, page the on-call engineer. The numbers are not sacred -- they are calibrated from six weeks of historical data, picking a threshold that catches real incidents without firing on normal noise. In Sentry, this is an alert rule on `event.type:error` filtered by `transaction:/transfers`.

**Anomaly detection** catches regressions that static thresholds miss. A payment-success rate that normally sits at 99.8% dropping to 98.5% is a catastrophe, but a static threshold of "alert below 95%" will not fire. Most RUM vendors support alerts of the form "deviates more than N standard deviations from 24-hour moving average." These are noisier and require tuning, but they catch the slow-rolling regressions that static thresholds let through.

**On-call integration** routes alerts into the tool the team already uses -- PagerDuty, Opsgenie, Incident.io. Sentry, Datadog, and Prometheus all have first-party integrations. Email-only alerting fires at 3 AM into an inbox no one reads; configure severity so low issues write Jira tickets, medium issues post to Slack, and high issues page a human. Every alert rule must link to a runbook -- an alert that requires someone to ask team chat "what does this mean" is a failure of the alerting system, not of the responder.

---

## Privacy and Regional Data Residency

Observability is data collection, and data collection is regulated. GDPR, LGPD, PIPEDA, and a patchwork of US state laws constrain what browser telemetry can capture, where it lives, and how users are informed. [Chapter 16](ch46-gdpr-compliance.md) covers the full compliance picture; three observability-specific implications:

**Anonymization at source.** `scrubPii` is the primary tool, paired with IP anonymization (Sentry's `sendDefaultPii: false`, Datadog's `IpAnonymizer` integration). The correlation ID is not a user identifier -- it resets at session end and is never linked to an account ID.

**Regional endpoints.** Sentry and Datadog both offer EU-residency data centers. Users whose account region is EU route to the EU collector, US users to the US collector. Routing uses a server-provided config value rather than client-side logic, so it cannot be tampered with: `dsn: region === 'eu' ? environment.sentryDsnEu : environment.sentryDsnUs`.

**Consent gating.** In EU jurisdictions, analytic and RUM collection typically require opt-in; `WebVitalsReporter.start()` must check a consent signal before attaching callbacks. Uncaught error monitoring usually falls under legitimate interest -- a financial app user reasonably expects the app to be functional -- but this is not universal. Consult counsel.

---

## Testing Telemetry

Telemetry tests are easy to neglect and expensive to lose. When a refactor accidentally removes the `transfer.succeeded` event, the executive dashboard shows zero transfers until someone notices weeks later. The pattern from [Chapter 7](ch07-testing-vitest.md) applies: inject a stub, verify the calls.

```typescript
// libs/features/transfers/src/lib/transfer.component.spec.ts
describe('TransferComponent telemetry', () => {
  const telemetryStub = { trackEvent: vi.fn(), log: vi.fn() };

  beforeEach(() => {
    telemetryStub.trackEvent.mockClear();
    TestBed.configureTestingModule({
      providers: [{ provide: TelemetryService, useValue: telemetryStub }],
    });
  });

  it('emits transfer.submitted with the amount bucket', async () => {
    const fixture = TestBed.createComponent(TransferComponent);
    await fixture.componentInstance.submit('100-1000', 'internal');

    expect(telemetryStub.trackEvent).toHaveBeenCalledWith('transfer.submitted', {
      amountBucket: '100-1000',
      destinationType: 'internal',
    });
    expect(telemetryStub.trackEvent).toHaveBeenCalledWith('transfer.succeeded', {
      destinationType: 'internal',
    });
  });
});
```

The assertion covers two contracts: the event name is correct (the product funnel breaks if it changes), and the properties are scrubbed (no raw amount, only a bucket). Both deserve explicit coverage. For `scrubPii` itself, property-based tests that throw thousands of random objects at it catch sensitive-looking strings you did not think of.

---

## Summary

Observability is the production counterpart to testing. Tests tell you the code works in a controlled environment; observability tells you whether it works for the users who actually matter.

- **Three pillars** -- logs, metrics, traces -- each answer a different question. Logs tell you what happened. Metrics tell you how often. Traces tell you where the time went.
- **Sentry** is the default for Angular error monitoring. Compose `Sentry.createErrorHandler()` with the `GlobalErrorHandler` from [Chapter 40](ch19-error-handling.md) rather than replacing it, so structured logging and analytics keep working alongside Sentry's capture.
- **Datadog RUM** is the session-centric alternative. The script-tag install trades third-party latency for decoupled SDK updates; the NPM install trades bundle size for a locked, auditable version.
- **OpenTelemetry** is vendor-neutral. `@opentelemetry/sdk-trace-web` plus `@opentelemetry/instrumentation-fetch` produces W3C-compatible traces that continue into backend services.
- **Releases and source maps** turn a stack trace into a fixable bug. Match the `release` identifier across `Sentry.init`, `sentry-cli releases new`, and `sentry-cli sourcemaps upload` in CI ([Chapter 34](ch41-ci-cd.md)).
- **Correlation IDs** generated once per session and attached via an HTTP interceptor ([Chapter 48](ch13-initialization-routes.md)) stitch browser events to backend logs. One `X-Correlation-Id` header collapses cross-service debugging from hours to minutes.
- **`TelemetryService`** centralizes log, error, and event dispatch behind one facade. Vendor changes become one-file diffs; testing becomes one stub injection.
- **Core Web Vitals** reported via the `web-vitals` library (`onLCP`, `onINP`, `onCLS`, `onFCP`, `onTTFB`) turn lab metrics into field data. Sample 10% or less in high-volume production; raise toward 100% in low-volume environments.
- **Custom business metrics** via `trackEvent()` feed conversion funnels. Namespace event names (`transfer.submitted`, `onboarding.completed`) so analytics tools can group them.
- **PII scrubbing** uses an allow-list, not a deny-list. `SAFE_FIELDS.has(key)` filters every outbound payload, Sentry's `beforeSend` provides a second pass, and numeric PII gets bucketed before transmission.
- **Alerting** translates metrics into paging. Static thresholds catch known failure modes; anomaly detection catches slow-rolling regressions. Every alert links to a runbook.
- **Privacy and residency** ([Chapter 16](ch46-gdpr-compliance.md)) require regional endpoint routing, consent gating for non-essential telemetry, and anonymization at source.
- **Testing telemetry** is as important as testing business logic. Stub `TelemetryService`, assert the calls, catch funnel-breaking refactors before they ship.

The goal is not to collect every possible data point. The goal is to collect, reliably and without leaking PII, the narrow set of signals that let a human detect and understand a production problem. Instrument the flows that matter, scrub what should not leave the browser, alert on what should wake someone up -- and let everything else fall below the observability threshold.

> **Companion code:** `financial-app/libs/shared/observability/` with `TelemetryService`, Sentry integration, correlation-id interceptor, and web-vitals reporter.
