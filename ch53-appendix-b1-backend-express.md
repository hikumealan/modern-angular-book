# Appendix B1: Backend Adapter -- Node + Express

This appendix is the Node + Express counterpart to [Appendix B0](ch52-appendix-b0-backend-fastapi.md). Read the main appendix first; this one shows only the deltas for a Node backend.

The primary path uses **`@asteasolutions/zod-to-openapi`**: the backend authors Zod schemas and validates requests with middleware built from them, and those same schemas emit `/openapi.json` automatically. Because Angular frontend (Zod-based Signal Forms) and the backend both speak Zod, you get the option of importing one shared schema package across the whole monorepo -- a genuine superpower the Python stack cannot match.

A sidebar covers **`tsoa`** for teams that prefer decorator-style controllers and route registration. Outside OpenAPI, **`ts-rest`** keeps the contract in a shared TypeScript file consumed by both sides; it's not covered in depth because this book is OpenAPI-centric, but it's worth mentioning for full-stack TS teams that don't need OpenAPI's broader ecosystem.

---

## 1. Overview

Express is unopinionated by design, so there is no one canonical OpenAPI workflow. Two stacks dominate in 2026:

- **Zod-first** (recommended here): `@asteasolutions/zod-to-openapi` + `zod` + `express` + a light validation middleware. Zod schemas are the single source of truth; the OpenAPI spec is a derived artifact.
- **Decorator-first** (`tsoa`): TypeScript classes with `@Route`/`@Get`/`@Post` decorators declare controllers; `tsoa` emits routes and OpenAPI in one step. Feels similar to NestJS or Spring for teams transitioning from those stacks.

Prefer the Zod-first path when:

- The frontend already uses Zod for Signal Forms (true for every example in this book).
- You want to eventually share a Zod package between frontend and backend.
- You want Express's "a function and a router" simplicity without framework ceremony.

Prefer `tsoa` when:

- The team comes from NestJS/Spring and wants a decorator-driven DX.
- You want class-based dependency injection and DTO binding without rolling it yourself.

---

## 2. Installing the Tooling

Zod-first minimal stack:

```bash
npm install express zod @asteasolutions/zod-to-openapi
npm install -D @types/express typescript tsx
```

Project shape:

```
backend-express/
|-- package.json
|-- tsconfig.json
|-- src/
|   |-- app.ts             # Express app + middleware
|   |-- schemas.ts         # Zod schemas (imported by frontend too)
|   |-- routes/
|   |   |-- accounts.ts
|   |   |-- transactions.ts
|   |-- dump-openapi.ts    # CLI: write openapi.json
```

Expose a `dev` script that starts the server and a `spec` script that dumps the OpenAPI JSON:

```json
{
  "scripts": {
    "dev": "tsx watch src/app.ts",
    "spec": "tsx src/dump-openapi.ts > ../openapi.json"
  }
}
```

For the `tsoa` variant:

```bash
npm install tsoa
npm install -D @types/express typescript
```

`tsoa` adds a `tsoa.json` config, generates `src/routes/routes.ts` from decorators, and emits `public/swagger.json`.

---

## 3. Equivalent of the FastAPI Example

The Zod-first translation of the FastAPI model + handler from [Appendix B0 §2](ch52-appendix-b0-backend-fastapi.md#2-shared-fastapi-backend):

```typescript
// backend-express/src/schemas.ts
import { z } from 'zod';
import { extendZodWithOpenApi, OpenAPIRegistry } from '@asteasolutions/zod-to-openapi';

extendZodWithOpenApi(z);

export const AccountSchema = z
  .object({
    id: z.number().int(),
    name: z.string().min(1).max(100),
    type: z.enum(['checking', 'savings', 'investment']),
    balance: z.number().nonnegative(),
    currency: z.enum(['USD', 'EUR', 'GBP', 'JPY']),
    ownerId: z.number().int(),
  })
  .openapi('Account');

export const TransactionDraftSchema = z
  .object({
    accountId: z.number().int(),
    amount: z.number().gt(0.01),
    type: z.enum(['credit', 'debit']),
    category: z.string().min(1).max(50),
    date: z.string().date(),
    description: z.string().max(500).optional(),
    pending: z.boolean().default(false),
  })
  .openapi('TransactionDraft');

export const TransactionSchema = TransactionDraftSchema.extend({
  id: z.number().int(),
}).openapi('Transaction');

export type Account = z.infer<typeof AccountSchema>;
export type TransactionDraft = z.infer<typeof TransactionDraftSchema>;
export type Transaction = z.infer<typeof TransactionSchema>;

export const registry = new OpenAPIRegistry();
registry.register('Account', AccountSchema);
registry.register('TransactionDraft', TransactionDraftSchema);
registry.register('Transaction', TransactionSchema);
```

```typescript
// backend-express/src/routes/accounts.ts
import { Router, type Request, type Response } from 'express';
import { AccountSchema, registry } from '../schemas';

export const accountsRouter = Router();
const accounts = new Map<number, Account>(/* ... seed data ... */);

accountsRouter.get('/api/accounts', (_req, res: Response) => {
  res.json([...accounts.values()]);
});

accountsRouter.get('/api/accounts/:id', (req: Request, res: Response) => {
  const account = accounts.get(Number(req.params.id));
  if (!account) {
    res.status(404).json({ code: 'NOT_FOUND', message: 'Account not found' });
    return;
  }
  res.json(AccountSchema.parse(account));
});

registry.registerPath({
  method: 'get',
  path: '/api/accounts',
  tags: ['accounts'],
  responses: {
    200: {
      description: 'List of accounts',
      content: { 'application/json': { schema: AccountSchema.array() } },
    },
  },
});

registry.registerPath({
  method: 'get',
  path: '/api/accounts/{id}',
  tags: ['accounts'],
  request: { params: z.object({ id: z.coerce.number().int() }) },
  responses: {
    200: { description: 'Account', content: { 'application/json': { schema: AccountSchema } } },
    404: { description: 'Not found' },
  },
});
```

```typescript
// backend-express/src/dump-openapi.ts
import { OpenApiGeneratorV31 } from '@asteasolutions/zod-to-openapi';
import { registry } from './schemas';
import './routes/accounts';
import './routes/transactions';

const generator = new OpenApiGeneratorV31(registry.definitions);
const spec = generator.generateDocument({
  openapi: '3.1.0',
  info: { title: 'FinancialApp API (Express)', version: '1.0.0' },
});

process.stdout.write(JSON.stringify(spec, null, 2));
```

The `dump-openapi.ts` CLI is the Node equivalent of FastAPI's `python -m backend.dump_openapi`. Run it in CI to produce the canonical `openapi.json`.

---

## 4. Per-Strategy Adjustments

All four strategies from [Appendix B0](ch52-appendix-b0-backend-fastapi.md) work against the Express-produced spec with minimal changes.

**Strategy A -- openapi-zod-client.** Unchanged. Point the generator at the committed `openapi.json`:

```bash
npx openapi-zod-client ../openapi.json \
  -o libs/shared/api-client/src/generated/generated-zodios.ts \
  --export-schemas --with-docs
```

**Strategy B -- Zod + Zod co-equal with CI diff.** This is where Node shines. Because both sides speak Zod, there are two options:

- **Two Zod schemas, diff the OpenAPI outputs.** Follow Appendix B0 §4 as-is.
- **One shared Zod package, zero diff.** Hoist `schemas.ts` into an Nx lib (`libs/shared/contract-schemas`), import it from both the Express backend and the Angular frontend. The diff script becomes a no-op because there is exactly one source. This is a genuine monorepo superpower -- frontend and backend cannot disagree because they are literally the same file.

```typescript
// libs/shared/contract-schemas/src/index.ts (shared)
export * from './account.schema';
export * from './transaction.schema';

// backend-express/src/app.ts
import { AccountSchema } from '@financial-app/shared/contract-schemas';

// apps/financial-app/src/app/features/.../transaction-entry.component.ts
import { TransactionDraftSchema } from '@financial-app/shared/contract-schemas';
```

The Strategy B diff still runs in CI as a safety net, but expects zero differences.

**Strategy C -- datamodel-code-generator + ts-to-zod.** Unchanged. Same pipeline, same output; the generator doesn't care the spec came from Node.

**Strategy D -- OpenAPI Generator typescript-angular.** Unchanged. Works against any OpenAPI 3.x spec. Because the spec is produced by a Node toolchain, the whole pipeline (backend + frontend + contract tests) can run in a single-language CI workflow with just `actions/setup-node@v4`. No separate Python or Java setup step.

---

## 5. Backend-Specific Gotchas

**`date` strings vs `Date` objects.** `z.string().date()` declares a "YYYY-MM-DD" string; `z.date()` declares a runtime `Date` object. Express deserializes JSON payloads as strings -- use `.date()` on the string or coerce with `z.coerce.date()` depending on whether downstream handlers expect a string or a `Date`.

**Middleware validation.** Unlike FastAPI (which validates automatically), Express requires explicit validation middleware. The `@asteasolutions/zod-to-openapi` companion `zod-express-middleware` package or a hand-rolled `(req, res, next) => schema.safeParse(req.body)` is the minimum. Without it, Express happily accepts malformed bodies and breaks at runtime.

```typescript
// backend-express/src/validate.ts
import type { RequestHandler } from 'express';
import type { ZodType } from 'zod';

export const validateBody =
  <T>(schema: ZodType<T>): RequestHandler =>
  (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      res.status(422).json({ code: 'VALIDATION_ERROR', message: 'Invalid body', details: result.error.flatten().fieldErrors });
      return;
    }
    req.body = result.data;
    next();
  };
```

**Shared Zod across the monorepo.** If you go with the shared-schemas approach (Strategy B's no-op variant), make sure:

- The shared lib uses a version of Zod compatible with the frontend. Pin `zod` in the workspace root, not in individual app `package.json`s.
- Backend runtime never imports frontend-only modules (Angular, RxJS). Keep the shared lib framework-agnostic.

**Decimal precision for currency.** FastAPI's `Decimal` translates to OpenAPI `number` with a `format: decimal` hint some generators respect. Node's JSON doesn't have a decimal type. If you need exact money math, wrap `amount` in a `z.string().regex(/^\d+\.\d{2}$/)` and parse to `bigint` on the backend.

**CORS.** FastAPI can be configured via `CORSMiddleware` in three lines. Express needs `cors()` middleware with explicit `origin` / `credentials` options. Set `credentials: true` if you use cookie-based auth ([Chapter 27](ch17-auth-patterns.md)).

---

## 6. Running the Contract Tests Against This Backend

Update the Playwright `webServer` entry to spawn the Node backend:

```typescript
// apps/financial-app-e2e/playwright.config.ts
webServer: [
  { command: 'npx nx serve financial-app', url: 'http://localhost:4200' },
  {
    command: 'npm run dev',
    url: 'http://localhost:8000/api/health',
    cwd: '../../backend-express',
  },
],
```

CI setup is simpler than the FastAPI case -- one language, no `actions/setup-python`:

```yaml
- uses: actions/setup-node@v4
  with: { node-version: '22' }
- run: npm ci
- run: npm run --prefix backend-express spec > openapi.json
- run: npm run generate:zod-client
- run: git diff --exit-code -- libs/shared/api-client
```

---

## 7. See Also

- [Appendix B0 §3](ch52-appendix-b0-backend-fastapi.md#3-strategy-a----fastapipydantic-as-single-source-of-truth) -- Strategy A details that apply unchanged to any backend.
- [Appendix B0 §4](ch52-appendix-b0-backend-fastapi.md#4-strategy-b----zod--pydantic-co-equal-with-ci-diff) -- Strategy B on the "two sources" configuration; contrast with the shared-schemas variant above.
- [Appendix B0 §9](ch52-appendix-b0-backend-fastapi.md#9-testing-against-the-single-source-of-truth-vitest--playwright) -- Vitest + Playwright patterns, unchanged.
- [Chapter 14](ch14-monorepos-libraries.md) -- Nx monorepo layout for hosting the shared Zod package.
- [Chapter 27](ch17-auth-patterns.md) -- cookie-based auth patterns; relevant for CORS configuration.
- [Appendix B2](ch54-ch54-ch54-appendix-b2-backend-spring.md) and [Appendix B3](ch55-ch55-ch55-appendix-b3-backend-go.md) -- equivalent recipes for Spring Boot and Go.
