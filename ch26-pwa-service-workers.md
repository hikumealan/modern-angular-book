# Chapter 26: PWA & Service Workers

A user opens FinancialApp on the train, checks their account balances, and switches to email. When they switch back two minutes later, the app loads instantly -- no spinner, no network waterfall. The train enters a tunnel and the connection drops. The portfolio overview still renders from cache, a subtle banner confirms they are offline, and the transaction they entered moments ago queues silently for submission when connectivity returns. On their home screen, the app sits alongside native banking apps, launched from an icon with no browser chrome in sight.

This is what a Progressive Web App delivers: offline resilience, instant repeat loads, and a native-like install experience -- built entirely on web standards. Angular's `@angular/pwa` package and the Angular Service Worker (`ngsw`) make this achievable without writing low-level service worker code by hand.

> **Prerequisites:** [Chapter 17](ch17-defer-ssr-hydration.md) (SSR/hydration -- understanding server vs client rendering), [Chapter 24](ch24-performance.md) (performance measurement -- understanding Core Web Vitals and caching).

---

## Adding @angular/pwa

The Angular CLI schematic handles the boilerplate:

```bash
ng add @angular/pwa
```

This single command generates and modifies several files:

- **`manifest.webmanifest`** -- the web app manifest that defines how the app appears when installed (name, icons, theme color, display mode).
- **`ngsw-config.json`** -- the Angular Service Worker configuration that controls caching strategies.
- **Icon assets** -- a set of placeholder icons at various resolutions in `public/icons/`.
- **Registration code** -- the schematic adds `provideServiceWorker` to your application config and links the manifest in `index.html`.

### Service Worker Registration

The schematic wires up registration in `app.config.ts`:

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideServiceWorker } from '@angular/service-worker';

export const appConfig: ApplicationConfig = {
  providers: [
    provideServiceWorker('ngsw-worker.js', {
      enabled: true,
      registrationStrategy: 'registerWhenStable:30000',
    }),
  ],
};
```

The `registrationStrategy` controls when the service worker registers. `registerWhenStable:30000` waits until the application stabilizes or 30 seconds elapse, whichever comes first. This prevents the service worker's precaching from competing with FinancialApp's initial API calls for bandwidth. Alternative strategies include `registerImmediately` and `registerWhenStable` (no timeout).

### Web App Manifest

The manifest tells the browser how to present the app when installed:

```json
{
  "name": "FinancialApp - Personal Finance Dashboard",
  "short_name": "FinancialApp",
  "theme_color": "#1a237e",
  "background_color": "#fafafa",
  "display": "standalone",
  "scope": "/",
  "start_url": "/",
  "icons": [
    { "src": "icons/icon-144x144.png", "sizes": "144x144", "type": "image/png" },
    { "src": "icons/icon-192x192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/icon-512x512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

The `display: "standalone"` setting removes the browser's address bar and navigation controls, making the installed app feel native. The `theme_color` tints the OS status bar on Android and the title bar on desktop. Replace the placeholder icons with FinancialApp's branding before shipping -- low-resolution or generic icons undermine the install experience.

---

## ngsw-config.json Deep Dive

The Angular Service Worker does not require you to write `fetch` event handlers or manage `CacheStorage` directly. You declare your caching strategy in `ngsw-config.json`, and Angular's build process generates the service worker logic from this configuration.

### Asset Groups

Asset groups control how static assets -- JavaScript bundles, CSS, images, fonts -- are cached.

- **`prefetch`** install mode downloads and caches assets immediately when the service worker installs, before the user requests them. Use this for the app shell: CSS, JavaScript bundles, and `index.html`. These must be available offline for the app to function at all.
- **`lazy`** install mode caches assets only when the user first requests them. Use this for images, fonts, and other assets that may never be needed. There is no point prefetching 50 portfolio chart thumbnails if the user only views three.

The `updateMode` controls what happens when a new version deploys. `prefetch` means the service worker downloads updated assets in the background immediately. `lazy` means updates are fetched on demand.

### Data Groups

Data groups handle API responses -- dynamic data that cannot be bundled at build time.

- **`freshness`** (network-first) -- tries the network first. If the request completes within `timeout`, the response is served and cached. If the network is unavailable or too slow, the cached response is served. Use this for data where staleness matters -- transaction lists, balances, portfolio values.
- **`performance`** (cache-first) -- serves from cache if available, regardless of network state. The cache entry is used until it expires (`maxAge`). Use this for data that changes infrequently -- account metadata, user profiles, institution details.

The `maxSize` parameter caps the number of cached responses in a group. When the limit is reached, the oldest entries are evicted. The `maxAge` parameter accepts durations like `1h`, `1d`, `30m`, or `365d`.

### Navigation URLs

The service worker intercepts navigation requests and serves the cached `index.html`, enabling the single-page app to handle routing offline. You can restrict which routes are intercepted:

```json
{
  "navigationUrls": ["/**", "!/__admin/**", "!/api/**"]
}
```

The `!` prefix excludes patterns, preventing the service worker from accidentally serving the Angular shell for server-side API routes or admin pages.

---

## Service Worker Lifecycle

Understanding the lifecycle clarifies why updates behave the way they do -- and why users sometimes see stale content.

**Install phase.** When the browser downloads the service worker for the first time, the `install` event fires. Angular's service worker prefetches all assets in `prefetch` groups. The new service worker waits in a "waiting" state until all tabs running the old version close.

**Activate phase.** Once the new service worker takes control, it cleans up caches from previous versions, removing stale assets no longer referenced by the current build.

**Fetch phase.** Every network request from a controlled page passes through the service worker. Angular matches the URL against configured asset and data groups, applying the appropriate strategy. Requests that match no group pass through to the network untouched.

**Update phase.** On each navigation, the service worker checks for a new `ngsw.json` manifest (generated during the build). If changed, new assets download in the background. The update does not take effect until the user navigates again or the application explicitly activates it -- a critical behavior managed through `SwUpdate`.

---

## Caching Strategies for FinancialApp

Different data in a financial application has different freshness requirements. A one-size-fits-all strategy leads to either stale balances or unnecessary network requests for static content.

**App shell (prefetch).** HTML, CSS, and JavaScript bundles are always available offline. Repeat visits load instantly with no network dependency.

**Network-first for transactions.** Transaction data changes frequently and accuracy matters. A 3-second timeout ensures the app tries the network first but falls back to cache rather than showing a blank screen.

**Cache-first for account metadata.** Account names, institution logos, and routing numbers rarely change. A `maxAge` of one day eliminates round-trips and makes the accounts list feel instant.

**Stale-while-revalidate for portfolio holdings.** Angular's service worker approximates this with the `performance` strategy and a short `maxAge`. Cached data renders immediately; the next navigation fetches fresh data from the network.

**Versioned cache busting.** Angular's build appends content hashes to filenames (`main.abc123.js`). The `ngsw.json` manifest tracks these hashes. On deployment, the service worker downloads updated bundles automatically.

Here is the complete `ngsw-config.json` for FinancialApp:

```json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app-shell",
      "installMode": "prefetch",
      "updateMode": "prefetch",
      "resources": {
        "files": ["/favicon.ico", "/index.html", "/manifest.webmanifest", "/*.css", "/*.js"]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": ["/assets/**", "/*.(svg|cur|jpg|jpeg|png|apng|webp|avif|gif|otf|ttf|woff|woff2)"]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-transactions",
      "urls": ["/api/transactions/**"],
      "cacheConfig": { "strategy": "freshness", "maxSize": 100, "maxAge": "1h", "timeout": "3s" }
    },
    {
      "name": "api-account-metadata",
      "urls": ["/api/accounts/metadata/**"],
      "cacheConfig": { "strategy": "performance", "maxSize": 50, "maxAge": "1d" }
    },
    {
      "name": "api-portfolio",
      "urls": ["/api/portfolio/**"],
      "cacheConfig": { "strategy": "performance", "maxSize": 50, "maxAge": "5m" }
    },
    {
      "name": "api-clients",
      "urls": ["/api/clients/**"],
      "cacheConfig": { "strategy": "performance", "maxSize": 200, "maxAge": "12h" }
    }
  ],
  "navigationUrls": ["/**", "!/api/**", "!/**/*.*"]
}
```

---

## Offline Support

Caching handles repeat requests, but true offline support requires the application to know it is offline and behave accordingly.

### Offline Banner

An `OfflineBannerComponent` wraps connectivity detection in a signal-based reactive pattern:

```typescript
// financial-app/src/app/shared/offline-banner.component.ts
import { Component, signal, OnDestroy } from '@angular/core';

@Component({
  selector: 'app-offline-banner',
  standalone: true,
  template: `
    @if (isOffline()) {
      <div class="offline-banner" role="alert">
        You are currently offline. Some data may not be up to date.
      </div>
    }
  `,
  styles: `
    .offline-banner {
      position: fixed;
      bottom: 0;
      left: 0;
      right: 0;
      padding: 12px 16px;
      background: #e65100;
      color: white;
      text-align: center;
      font-size: 14px;
      z-index: 1000;
      animation: slideUp 300ms ease-out;
    }
    @keyframes slideUp {
      from { transform: translateY(100%); }
      to { transform: translateY(0); }
    }
  `,
})
export class OfflineBannerComponent implements OnDestroy {
  protected readonly isOffline = signal(!navigator.onLine);

  private readonly onlineHandler = () => this.isOffline.set(false);
  private readonly offlineHandler = () => this.isOffline.set(true);

  constructor() {
    window.addEventListener('online', this.onlineHandler);
    window.addEventListener('offline', this.offlineHandler);
  }

  ngOnDestroy(): void {
    window.removeEventListener('online', this.onlineHandler);
    window.removeEventListener('offline', this.offlineHandler);
  }
}
```

Place `<app-offline-banner />` in the root `AppComponent` template. Because the component reads a signal, it updates automatically when connectivity changes -- no subscriptions to manage.

### Queuing Writes with Background Sync

Reading cached data offline is straightforward. Writing data offline is harder. When a user enters a transaction while offline, the `BackgroundSyncService` stores pending writes in IndexedDB and submits them when connectivity returns:

```typescript
// libs/shared/data-access/src/background-sync.service.ts
import { Injectable, inject, effect, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface PendingRequest {
  id: string;
  url: string;
  method: 'POST' | 'PUT' | 'DELETE';
  body: unknown;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class BackgroundSyncService {
  private readonly http = inject(HttpClient);
  private readonly online = signal(navigator.onLine);

  constructor() {
    window.addEventListener('online', () => this.online.set(true));
    effect(() => {
      if (this.online()) this.flushQueue();
    });
  }

  async queue(url: string, method: PendingRequest['method'], body: unknown): Promise<void> {
    const db = await this.openDB();
    const tx = db.transaction('pending', 'readwrite');
    tx.objectStore('pending').add({
      id: crypto.randomUUID(), url, method, body, timestamp: Date.now(),
    });
  }

  private async flushQueue(): Promise<void> {
    const db = await this.openDB();
    const pending: PendingRequest[] = await new Promise((resolve, reject) => {
      const req = db.transaction('pending', 'readonly').objectStore('pending').getAll();
      req.onsuccess = () => resolve(req.result);
      req.onerror = () => reject(req.error);
    });

    for (const request of pending) {
      try {
        await this.http.request(request.method, request.url, { body: request.body }).toPromise();
        const delTx = db.transaction('pending', 'readwrite');
        delTx.objectStore('pending').delete(request.id);
      } catch {
        break;
      }
    }
  }

  private openDB(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
      const req = indexedDB.open('financial-app-sync', 1);
      req.onupgradeneeded = () => req.result.createObjectStore('pending', { keyPath: 'id' });
      req.onsuccess = () => resolve(req.result);
      req.onerror = () => reject(req.error);
    });
  }
}
```

The key design decision is ordering: `flushQueue` processes requests sequentially and stops on the first failure. For financial transactions, order matters -- a transfer-out followed by a transfer-in must execute in sequence. A transaction form checks connectivity and routes the write accordingly:

```typescript
// financial-app/src/app/features/transactions/add-transaction.component.ts
async submitTransaction(): Promise<void> {
  if (navigator.onLine) {
    this.transactionService.create(this.transactionForm.value).subscribe();
  } else {
    await this.backgroundSync.queue('/api/transactions', 'POST', this.transactionForm.value);
    this.snackbar.open('Transaction saved. It will sync when you reconnect.');
  }
}
```

For routes the user has never visited, there is no cache entry to fall back on. Display a clear message rather than a broken page -- check the offline state and render a "not available offline" fallback alongside `<mat-icon>cloud_off</mat-icon>` instead of a spinner.

---

## Update Management

Deploying a new version does not automatically update users running the old service worker. The old version continues serving cached assets until all tabs close or the app explicitly activates the update. For a financial app, running outdated code is a liability.

### SwUpdate Service

Angular's `SwUpdate` service exposes the update lifecycle as observables:

```typescript
// financial-app/src/app/shared/update-prompt.component.ts
import { Component, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { SwUpdate, VersionReadyEvent } from '@angular/service-worker';
import { filter, map } from 'rxjs';

@Component({
  selector: 'app-update-prompt',
  standalone: true,
  template: `
    @if (updateReady()) {
      <div class="update-banner" role="alert">
        <span>A new version of FinancialApp is available.</span>
        <button (click)="applyUpdate()">Update now</button>
        <button (click)="updateReady.set(false)">Later</button>
      </div>
    }
  `,
  styles: `
    .update-banner {
      position: fixed; top: 0; left: 0; right: 0;
      display: flex; align-items: center; justify-content: center; gap: 16px;
      padding: 12px 16px; background: #1a237e; color: white; z-index: 1000;
    }
    button {
      padding: 6px 16px; border: 1px solid white;
      border-radius: 4px; background: transparent; color: white; cursor: pointer;
    }
    button:first-of-type { background: white; color: #1a237e; }
  `,
})
export class UpdatePromptComponent {
  private readonly swUpdate = inject(SwUpdate);

  readonly updateReady = toSignal(
    this.swUpdate.versionUpdates.pipe(
      filter((e): e is VersionReadyEvent => e.type === 'VERSION_READY'),
      map(() => true),
    ),
    { initialValue: false },
  );

  async applyUpdate(): Promise<void> {
    await this.swUpdate.activateUpdate();
    document.location.reload();
  }
}
```

The `versionUpdates` observable emits `VERSION_DETECTED` (new version exists), `VERSION_READY` (downloaded and ready), and `VERSION_INSTALLATION_FAILED`. Filtering for `VERSION_READY` triggers the user prompt. The `document.location.reload()` after `activateUpdate()` is intentional -- activating swaps the service worker, but the running JavaScript in memory is still the old version.

### Handling Broken Service Workers

Occasionally, a service worker enters an unrecoverable state -- a deployment removed assets the old worker still references, or cache corruption prevents loading. Angular exposes this:

```typescript
// app.config.ts (inside providers)
{
  provide: APP_INITIALIZER,
  multi: true,
  useFactory: () => {
    const swUpdate = inject(SwUpdate);
    return () => swUpdate.unrecoverable.subscribe(event => {
      console.error('SW unrecoverable:', event.reason);
      document.location.reload();
    });
  },
}
```

A hard reload fetches everything from the network and re-registers the service worker. In a financial context, a brief reload is far preferable to a broken screen.

---

## Push Notifications

Push notifications let FinancialApp reach users even when the app is not open -- alerting them to large debits, portfolio threshold breaches, or pending approvals.

```typescript
// financial-app/src/app/core/notification.service.ts
import { Injectable, inject } from '@angular/core';
import { SwPush } from '@angular/service-worker';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private readonly swPush = inject(SwPush);
  private readonly http = inject(HttpClient);
  private readonly VAPID_PUBLIC_KEY = 'BLBx-hf2WoEfhB...your-vapid-public-key';

  async subscribe(): Promise<void> {
    if (!this.swPush.isEnabled) return;
    const sub = await this.swPush.requestSubscription({
      serverPublicKey: this.VAPID_PUBLIC_KEY,
    });
    await this.http.post('/api/notifications/subscribe', sub).toPromise();
  }

  listenForClicks(): void {
    this.swPush.notificationClicks.subscribe(({ action, notification }) => {
      if (action === 'view-transaction') {
        window.open(notification.data?.url, '_blank');
      }
    });
  }
}
```

The subscription flow: `requestSubscription` prompts the user for notification permission. If granted, the browser generates a `PushSubscription` containing the endpoint and encryption keys. The app sends this to the server, which stores it and uses it to push messages later.

On the server side, the **Web Push Protocol** with VAPID (Voluntary Application Server Identification) keys handles message delivery. Libraries like `web-push` (Node.js) manage the cryptography. Generate a VAPID key pair, store each client's `PushSubscription`, and send push messages when events occur -- large debit detected, portfolio drops below threshold, security alert triggered.

For FinancialApp, notifications are opt-in and scoped: users choose categories (transaction alerts, portfolio thresholds, security notices) from a preferences screen. The server only sends messages matching each user's selections.

---

## PWA Install Experience

A PWA becomes installable when it meets the browser's criteria: HTTPS, a valid manifest, and a registered service worker. The browser fires a `beforeinstallprompt` event that the app can capture and defer:

```typescript
// financial-app/src/app/core/install.service.ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class InstallService {
  private deferredPrompt: BeforeInstallPromptEvent | null = null;
  readonly canInstall = signal(false);

  constructor() {
    window.addEventListener('beforeinstallprompt', (event: Event) => {
      event.preventDefault();
      this.deferredPrompt = event as BeforeInstallPromptEvent;
      this.canInstall.set(true);
    });
    window.addEventListener('appinstalled', () => {
      this.canInstall.set(false);
      this.deferredPrompt = null;
    });
  }

  async promptInstall(): Promise<void> {
    if (!this.deferredPrompt) return;
    this.deferredPrompt.prompt();
    const { outcome } = await this.deferredPrompt.userChoice;
    if (outcome === 'accepted') this.canInstall.set(false);
    this.deferredPrompt = null;
  }
}
```

A custom install button in FinancialApp's toolbar surfaces this at the right moment:

```html
<!-- toolbar.component.html -->
@if (installService.canInstall()) {
  <button mat-stroked-button (click)="installService.promptInstall()">
    Install FinancialApp
  </button>
}
```

The button appears only when the browser determines the app is installable and disappears after installation. Placing it in a contextually relevant location -- toolbar, settings page, post-login welcome -- improves install rates significantly over the browser's default prompt.

---

## Testing PWA

PWA features cannot be tested with `ng serve` -- the dev server does not register a service worker. Build in production mode and serve statically:

```bash
ng build --configuration production
npx http-server dist/financial-app/browser -p 8080
```

**Lighthouse PWA audit.** Lighthouse evaluates your PWA against a checklist: valid manifest, registered service worker, HTTPS, offline capability, and fast loading. Run it from Chrome DevTools or from the CLI as part of [Chapter 25](ch25-e2e-playwright.md)'s Lighthouse CI integration. Common failures include missing manifest fields and icons below minimum resolution (512x512 for maskable icons).

**Chrome DevTools Application panel.** The primary debugging tool. The **Service Workers** section shows registration status, running version, and update state -- you can force `skipWaiting` or unregister entirely. **Cache Storage** lists all caches and their entries, letting you verify that `ngsw-config.json` groups are working. **Manifest** displays the parsed manifest and flags errors.

**Testing offline mode.** In the Network tab, check "Offline" and navigate through the app. Verify the shell renders from cache, previously visited routes display cached data, the offline banner appears, queued writes are stored in IndexedDB, and uncached routes show a fallback rather than a broken page. Automate this with Playwright, which can simulate offline mode programmatically and tie PWA verification into the E2E suite from [Chapter 25](ch25-e2e-playwright.md).

---

## Summary

Progressive Web Apps transform FinancialApp from a browser tab into a reliable, installable application that works regardless of network conditions. The Angular Service Worker handles the heavy lifting through declarative configuration rather than imperative service worker code.

- **`@angular/pwa`** scaffolds the manifest, icons, service worker registration, and `ngsw-config.json`. The `provideServiceWorker` function in `app.config.ts` controls registration timing.
- **Asset groups** with `prefetch` mode ensure the app shell is always available offline. Lazy-loaded assets are cached on first use, keeping the initial precache small.
- **Data groups** apply `freshness` (network-first) or `performance` (cache-first) strategies to API responses. Transactions use network-first for accuracy; account metadata uses cache-first for speed.
- **Offline support** combines connectivity detection, an `OfflineBannerComponent` for user awareness, and a `BackgroundSyncService` that queues writes in IndexedDB and flushes them when the connection returns.
- **`SwUpdate`** manages the update lifecycle. The `UpdatePromptComponent` informs users when a new version is available, and `unrecoverable` handles broken service worker states with a forced reload.
- **`SwPush`** integrates push notifications through VAPID-based subscriptions, scoped to user-selected alert categories.
- **The install experience** captures `beforeinstallprompt` and surfaces a custom install button, improving install rates over the browser's default prompt.
- **Testing** requires a production build. Lighthouse audits validate PWA compliance, DevTools inspect service worker and cache state, and Playwright simulates offline scenarios in E2E tests.

The service worker sits between your application and the network -- a powerful position that demands careful configuration. Start with the app shell in `prefetch` mode and conservative `maxAge` values on data groups. Expand caching scope as you gain confidence that your update management handles version transitions cleanly. The performance gains from [Chapter 24](ch24-performance.md) compound with service worker caching: once assets are cached locally, repeat visits skip the network entirely, delivering sub-100ms load times that no server optimization can match.

> **Companion code:** `ngsw-config.json`, `OfflineBannerComponent`, and `UpdatePromptComponent` live in `financial-app/`. The `BackgroundSyncService` demonstrates offline queue patterns in `libs/shared/data-access/`.
