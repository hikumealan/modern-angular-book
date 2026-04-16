# Chapter 5: State Management with Services & Signals

Angular applications begin as a tree of components, each managing its own local state. A login form tracks its username and password fields. A transaction table sorts and filters rows. This local state is simple, self-contained, and easy to reason about -- until it isn't.

The moment two components need access to the same data, local state stops working. A sidebar badge needs the count of pending transactions. A header needs the current account name. A detail view needs the same account object the list view already fetched. Duplicating the data leads to inconsistencies; lifting it into a shared parent leads to prop-drilling through layers of components that have no business knowing about accounts.

Angular's answer to shared state is the **service** -- a plain TypeScript class, managed by the dependency injection system, that lives outside the component tree. Combined with signals (introduced in [Chapter 3](ch03-reactive-signals.md)), services become lightweight, reactive stores that keep state consistent across the entire application without the ceremony of a full state management library.

## Services

### Generating a Service

The Angular CLI scaffolds a service with a single command:

```bash
npx ng generate service accounts/account --project=financial-app
```

This produces two files: `account.service.ts` and `account.service.spec.ts`. The generated service is minimal:

```typescript
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AccountService {}
```

The `providedIn: 'root'` declaration registers the service in the root injector, making it a singleton available throughout the application. No module or component needs to list it in a `providers` array -- Angular tree-shakes it automatically if nothing injects it.

### Implementing a Service

A service encapsulates data access and business logic that components should not own directly. For the FinancialApp, `AccountService` wraps HTTP calls and exposes domain operations:

```typescript
// financial-app/libs/shared/data-access/account.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Account, CreateAccountDto } from '../model/account.model';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class AccountService {
  private http = inject(HttpClient);
  private baseUrl = '/api/accounts';

  getAll(): Observable<Account[]> {
    return this.http.get<Account[]>(this.baseUrl);
  }

  getById(id: string): Observable<Account> {
    return this.http.get<Account>(`${this.baseUrl}/${id}`);
  }

  create(dto: CreateAccountDto): Observable<Account> {
    return this.http.post<Account>(this.baseUrl, dto);
  }

  close(id: string): Observable<void> {
    return this.http.patch<void>(`${this.baseUrl}/${id}`, { status: 'closed' });
  }
}
```

Notice that the service returns `Observable` values from `HttpClient` rather than subscribing internally. This keeps the service stateless and leaves the subscription lifecycle to the consumer.

### Injecting a Service

Components consume a service through the `inject()` function. There is no constructor parameter list to maintain and no ambiguity about where dependencies come from:

```typescript
import { Component, inject } from '@angular/core';
import { AccountService } from '@financial-app/shared/data-access';

@Component({
  selector: 'app-account-list',
  standalone: true,
  template: `<!-- template omitted for brevity -->`,
})
export class AccountListComponent {
  private accountService = inject(AccountService);

  accounts$ = this.accountService.getAll();
}
```

The `inject()` function replaces the older constructor-injection pattern entirely. It is more concise, works in any injection context, and avoids the ceremony of declaring constructor parameters.

### Injection Context

`inject()` must be called inside an **injection context** -- the narrow window during which Angular's dependency injection system is active. In practice, this means:

- **Field initializers** of a class instantiated by DI (components, directives, services, pipes).
- **Constructor bodies** of those same classes.
- **Factory functions** passed to `Provider` configurations or `runInInjectionContext`.

Calling `inject()` outside these contexts -- inside a button click handler, a `setTimeout` callback, or a standalone utility function -- throws a runtime error. If you need a dependency later in a component's lifecycle, capture it during initialization:

```typescript
export class AccountDetailComponent {
  private accountService = inject(AccountService);
  private route = inject(ActivatedRoute);

  loadAccount(id: string) {
    // accountService was captured at construction time -- safe to use here
    this.accountService.getById(id).subscribe(/* ... */);
  }
}
```

### Services with Dependencies

Services inject other services the same way components do. An `AccountService` that needs logging and configuration composes its dependencies at the field level:

```typescript
@Injectable({ providedIn: 'root' })
export class AccountService {
  private http = inject(HttpClient);
  private logger = inject(LoggerService);
  private config = inject(AppConfigService);

  private baseUrl = `${this.config.apiRoot}/accounts`;

  getAll(): Observable<Account[]> {
    this.logger.debug('Fetching all accounts');
    return this.http.get<Account[]>(this.baseUrl);
  }
}
```

Angular resolves the full dependency graph automatically. If `LoggerService` itself depends on a `CorrelationIdService`, that too is resolved before `AccountService` is instantiated. Circular dependencies produce a clear compile-time or runtime error rather than silent misbehavior.

### Exchanging Services with Providers

Dependency injection is not just a convenience for wiring -- it is a mechanism for **polymorphism at the architecture level**. Angular's provider system lets you swap one implementation for another without touching the consuming code.

The four provider strategies are:

```typescript
// Replace the implementation class entirely
{ provide: AccountService, useClass: MockAccountService }

// Alias one token to another
{ provide: LegacyAccountService, useExisting: AccountService }

// Construct the service with custom logic
{
  provide: AccountService,
  useFactory: () => {
    const http = inject(HttpClient);
    const config = inject(AppConfigService);
    return new AccountService(http, config.accountsEndpoint);
  },
}

// Provide a static value (useful for configuration objects)
{ provide: API_BASE_URL, useValue: '/api/v2' }
```

`useClass` is the workhorse for testing and environment-specific overrides. `useExisting` creates aliases -- useful during migrations when you rename a service but want the old token to keep working. `useFactory` gives you full control over construction, including conditional logic based on runtime configuration. `useValue` is ideal for injection tokens that carry configuration rather than behavior.

### Short-Hand Syntax for Providers

When `useClass` points to the same class as the token, Angular allows a short-hand: simply list the class in the `providers` array.

```typescript
// These two declarations are equivalent:
providers: [AccountService]
providers: [{ provide: AccountService, useClass: AccountService }]
```

The short-hand is convenient for component-local services where you want a fresh instance but do not need to swap implementations.

### Provider Functions

Angular v14 introduced `provide*` helper functions that group related providers into reusable units. You have already seen `provideRouter()` and `provideHttpClient()`. You can author your own:

```typescript
import { EnvironmentProviders, makeEnvironmentProviders } from '@angular/core';

export function provideAccountsApi(apiRoot: string): EnvironmentProviders {
  return makeEnvironmentProviders([
    { provide: API_BASE_URL, useValue: apiRoot },
    AccountService,
    AccountStore,
  ]);
}
```

Consumers call this function in `app.config.ts` or a route's `providers` array, treating the entire accounts data-access layer as a single configuration unit:

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    provideAccountsApi('/api/v2'),
  ],
};
```

This pattern scales well. Each domain can export its own `provide*` function, keeping `app.config.ts` declarative and high-level.

### Component-local Services

By default, `providedIn: 'root'` makes a service a singleton. Sometimes you want a **separate instance** for each component -- for example, a form-editing service that tracks unsaved changes for one specific account, not globally.

Declare the service in the component's `providers` array:

```typescript
@Component({
  selector: 'app-account-editor',
  standalone: true,
  providers: [AccountEditorService],
  template: `<!-- ... -->`,
})
export class AccountEditorComponent {
  private editor = inject(AccountEditorService);
}
```

Angular creates a new `AccountEditorService` instance every time it instantiates `AccountEditorComponent` and destroys it when the component is removed from the DOM. Child components that inject `AccountEditorService` receive the same instance as their parent -- the provider acts as a boundary.

Note that `AccountEditorService` should **not** use `providedIn: 'root'` if it is intended to be component-local. Declare it with a bare `@Injectable()` so that Angular does not register it globally:

```typescript
@Injectable()
export class AccountEditorService {
  // instance-scoped state lives here
}
```

### Route-local Services and Auto Cleanup

Route-level providers sit between the global singleton and the per-component instance. Services provided on a route are instantiated when the route activates and destroyed when the user navigates away:

```typescript
export const accountRoutes: Routes = [
  {
    path: '',
    providers: [AccountStore],
    children: [
      { path: '', component: AccountListComponent },
      { path: ':id', component: AccountDetailComponent },
    ],
  },
];
```

Every component under this route subtree shares a single `AccountStore` instance. When the user navigates to a different section of the application, Angular destroys the route's injector -- and the store with it.

For services that set up subscriptions or timers, `DestroyRef` provides a clean teardown hook:

```typescript
@Injectable()
export class AccountPollingService {
  private http = inject(HttpClient);
  private destroyRef = inject(DestroyRef);

  constructor() {
    const subscription = interval(30_000)
      .pipe(switchMap(() => this.http.get<Account[]>('/api/accounts')))
      .subscribe(accounts => this.handleUpdate(accounts));

    this.destroyRef.onDestroy(() => subscription.unsubscribe());
  }

  private handleUpdate(accounts: Account[]) {
    // push updates to a signal or subject
  }
}
```

`DestroyRef.onDestroy()` fires when the injector that created the service is destroyed -- whether that injector belongs to a component, a route, or the application root. This replaces the older `OnDestroy` lifecycle hook for services and avoids the need to implement an interface.

## State Management

### Implementing a Store with Signals

Services handle data access. Stores handle **state**. A store is a service that owns reactive state and exposes it through signals. For the FinancialApp, the `AccountStore` manages the list of accounts, the currently selected account, and loading state:

```typescript
// financial-app/apps/.../accounts/account.store.ts
import { Injectable, inject, signal, computed } from '@angular/core';
import { AccountService } from '@financial-app/shared/data-access';
import { Account } from '@financial-app/shared/model';

@Injectable()
export class AccountStore {
  private accountService = inject(AccountService);

  private readonly state = signal<{
    accounts: Account[];
    selectedId: string | null;
    loading: boolean;
    error: string | null;
  }>({
    accounts: [],
    selectedId: null,
    loading: false,
    error: null,
  });

  readonly accounts = computed(() => this.state().accounts);
  readonly selectedAccount = computed(() =>
    this.state().accounts.find(a => a.id === this.state().selectedId) ?? null
  );
  readonly loading = computed(() => this.state().loading);
  readonly error = computed(() => this.state().error);

  loadAccounts(): void {
    this.state.update(s => ({ ...s, loading: true, error: null }));
    this.accountService.getAll().subscribe({
      next: accounts => this.state.update(s => ({ ...s, accounts, loading: false })),
      error: err => this.state.update(s => ({ ...s, error: err.message, loading: false })),
    });
  }

  selectAccount(id: string): void {
    this.state.update(s => ({ ...s, selectedId: id }));
  }
}
```

The entire store state lives in a single `signal()`. Mutations go through `update()`, which receives the previous value and returns the next -- the same pattern as a Redux reducer, but without the boilerplate of actions, action creators, and effect classes.

Computed signals (`accounts`, `selectedAccount`, `loading`, `error`) project slices of the state. Angular only re-renders a component when the specific computed signal it reads actually changes value.

### Consuming the Store

Components inject the store and bind directly to its signals in the template:

```typescript
@Component({
  selector: 'app-account-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    @if (store.loading()) {
      <app-spinner />
    } @else {
      @for (account of store.accounts(); track account.id) {
        <app-account-card
          [account]="account"
          [selected]="account.id === store.selectedAccount()?.id"
          (select)="store.selectAccount(account.id)"
        />
      }
    }

    @if (store.error(); as error) {
      <app-error-banner [message]="error" />
    }
  `,
})
export class AccountListComponent {
  protected store = inject(AccountStore);

  constructor() {
    this.store.loadAccounts();
  }
}
```

There are no subscriptions to manage, no `async` pipes to chain, and no `OnDestroy` cleanup. Signal reads in the template are synchronous and automatically tracked by Angular's change detection.

### Delegated Signals

Some store state derives from external sources that change over time -- a route parameter, a query string, or a parent signal. Angular's `linkedSignal()` creates a writable signal that **resets** whenever its source changes:

```typescript
import { linkedSignal } from '@angular/core';

@Injectable()
export class AccountStore {
  private accountService = inject(AccountService);
  private route = inject(ActivatedRoute);

  readonly selectedId = toSignal(
    this.route.paramMap.pipe(map(params => params.get('id'))),
    { initialValue: null },
  );

  readonly editingName = linkedSignal(() => {
    const id = this.selectedId();
    const account = this.accounts().find(a => a.id === id);
    return account?.name ?? '';
  });

  // editingName is writable -- the user can type a new name --
  // but resets automatically when selectedId changes.
}
```

`linkedSignal()` bridges the gap between derived state and editable state. Without it, developers resort to effects or manual subscriptions to reset form fields when navigation changes -- a pattern that is error-prone and difficult to test. See [Chapter 3](ch03-reactive-signals.md) for a deeper treatment of `linkedSignal` semantics.

### Lifetime and Scopes

The injector hierarchy determines how long a store lives and who shares it:

| Scope | Lifetime | Sharing | Use Case |
|---|---|---|---|
| `providedIn: 'root'` | Application lifetime | Global singleton | User session, auth state |
| Route `providers` | Route activation | All components in route subtree | Feature-level state (accounts) |
| Component `providers` | Component lifetime | Component and its children | Editor state, wizard step |

Choosing the right scope is a design decision, not a technical one. A store that is too global accumulates stale state and becomes a source of subtle bugs. A store that is too local forces re-fetching data that could have been cached.

For the FinancialApp, `AccountStore` lives at the route level. When the user navigates to the accounts section, the store is created and fetches data. When they navigate to portfolios, the store is destroyed, and its memory is freed. If they return to accounts, a fresh store fetches current data -- no risk of showing stale account information from minutes ago.

### Outlook: NgRx Signal Store

The hand-rolled store pattern shown above works well for small-to-medium feature state. As state logic grows -- optimistic updates, entity collections, undo/redo, devtools integration -- the pattern benefits from structure.

NgRx Signal Store provides that structure. It builds on the same primitives (`signal`, `computed`) but adds:

- **`withState()`** for declaring initial state with type inference.
- **`withComputed()`** for derived state that is automatically scoped to the store.
- **`withMethods()`** for state mutations and side effects.
- **`withEntities()`** for normalized entity collections with CRUD operations.
- **`withHooks()`** for lifecycle management tied to the store's injector.

The migration from a hand-rolled signal store to NgRx Signal Store is incremental -- the external API (signals read in templates) remains identical. We will build a full NgRx Signal Store implementation in [Chapter 9](ch09-ngrx-signal-store.md).

## Summary

Services and signals together form Angular's native state management layer. Services provide the structure -- singleton or scoped lifetimes managed by dependency injection. Signals provide the reactivity -- synchronous reads, automatic tracking, and fine-grained updates without subscription management.

The key patterns from this chapter:

- Use `@Injectable({ providedIn: 'root' })` for stateless, application-wide services like `AccountService`.
- Use bare `@Injectable()` with route or component `providers` for scoped, stateful services like `AccountStore`.
- Use `inject()` exclusively -- constructor injection is legacy noise.
- Use `signal()` and `computed()` to build stores that are simple to write, type-safe, and efficient to render.
- Use `linkedSignal()` when derived state needs to be locally editable and auto-reset on source changes.
- Use `DestroyRef` for cleanup in route-scoped services instead of implementing `OnDestroy`.
- Use `provide*` factory functions to package domain-level providers into reusable configuration units.

These primitives handle the vast majority of state management needs. When a feature outgrows them -- when you need entity normalization, devtools, or plugin-based extensibility -- NgRx Signal Store offers a structured upgrade path without abandoning the signal-based model. We will explore that path in [Chapter 9](ch09-ngrx-signal-store.md).
