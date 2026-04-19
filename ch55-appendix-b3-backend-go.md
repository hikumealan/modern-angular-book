# Appendix B3: Backend Adapter -- Go

This appendix is the Go counterpart to [Appendix B0](ch52-appendix-b0-backend-fastapi.md). Read the main appendix first; this one shows only the deltas for a Go backend.

Go has two common OpenAPI workflows, each suited to a different team style:

- **`oapi-codegen` (spec-first):** you hand-author `openapi.yaml`, the tool generates Go server stubs *and* type-safe handler interfaces. The spec is literally the contract; the Go code is derived. Natural fit when the API team prefers to design the contract before implementation.
- **`swaggo/swag` (code-first):** you annotate Go handlers with structured comments (`// @Summary`, `// @Param`, `// @Success`), and `swag init` emits `docs/swagger.json`. Natural fit when the team prefers inline documentation and incremental spec evolution.

Both produce OpenAPI 3.x JSON the four Angular strategies consume unchanged. The rest of this appendix walks the `oapi-codegen` path with a `swaggo/swag` sidebar, because spec-first is closer to the book's "contract is the source of truth" philosophy.

---

## 1. Overview

Go's type system is strict, its JSON serialization is predictable, and its HTTP story is unopinionated. Several popular routers (Gin, Echo, chi, standard library `net/http`) all speak `http.Handler`, so the generator output works with whichever you prefer.

The pieces:

- **Model structs** with JSON and validation tags define the wire format.
- **Validator** (`go-playground/validator`) enforces constraints -- `binding:"required,gte=0.01"`.
- **Router** (your choice) wires handlers to paths.
- **OpenAPI source** (`openapi.yaml` committed or comment pragmas in handlers) produces the spec.

Prefer **`oapi-codegen`** when:

- The API is designed before implementation (contract-first shops).
- You want client stubs generated for internal service-to-service calls alongside the Angular client.
- The team is comfortable editing YAML.

Prefer **`swaggo/swag`** when:

- The API evolves handler-by-handler.
- You already have handlers that need to be documented (retrofit).
- YAML maintenance feels like friction.

---

## 2. Installing the Tooling

For `oapi-codegen`:

```bash
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.4.1
```

Project layout:

```
backend-go/
|-- go.mod
|-- openapi.yaml             # the authored spec
|-- cmd/server/main.go
|-- internal/
|   |-- api/
|   |   |-- api.gen.go       # generated types + server interface
|   |-- handlers/
|   |   |-- accounts.go
|   |   |-- transactions.go
|-- Makefile                 # generate + build targets
```

`oapi-codegen.yaml` config:

```yaml
# backend-go/oapi-codegen.yaml
package: api
generate:
  models: true
  std-http-server: true
output: internal/api/api.gen.go
```

Run:

```bash
oapi-codegen -config oapi-codegen.yaml openapi.yaml
```

For `swaggo/swag`:

```bash
go install github.com/swaggo/swag/cmd/swag@v1.16.3
# Run inside the backend-go/ directory
swag init --generalInfo cmd/server/main.go --output docs
```

---

## 3. Equivalent of the FastAPI Example

Go structs with JSON + validation tags:

```go
// backend-go/internal/model/account.go
package model

import "github.com/shopspring/decimal"

type AccountType string

const (
    AccountTypeChecking   AccountType = "checking"
    AccountTypeSavings    AccountType = "savings"
    AccountTypeInvestment AccountType = "investment"
)

type CurrencyCode string

const (
    CurrencyUSD CurrencyCode = "USD"
    CurrencyEUR CurrencyCode = "EUR"
    CurrencyGBP CurrencyCode = "GBP"
    CurrencyJPY CurrencyCode = "JPY"
)

type Account struct {
    ID       int             `json:"id" validate:"required"`
    Name     string          `json:"name" validate:"required,min=1,max=100"`
    Type     AccountType     `json:"type" validate:"required,oneof=checking savings investment"`
    Balance  decimal.Decimal `json:"balance" validate:"required,gte=0"`
    Currency CurrencyCode    `json:"currency" validate:"required,oneof=USD EUR GBP JPY"`
    OwnerID  int             `json:"ownerId" validate:"required"`
}
```

```go
// backend-go/internal/model/transaction.go
package model

import (
    "time"
    "github.com/shopspring/decimal"
)

type TransactionType string

const (
    TransactionCredit TransactionType = "credit"
    TransactionDebit  TransactionType = "debit"
)

type TransactionDraft struct {
    AccountID   int             `json:"accountId" validate:"required"`
    Amount      decimal.Decimal `json:"amount" validate:"required,gt=0.01"`
    Type        TransactionType `json:"type" validate:"required,oneof=credit debit"`
    Category    string          `json:"category" validate:"required,min=1,max=50"`
    Date        time.Time       `json:"date" validate:"required"`
    Description *string         `json:"description,omitempty" validate:"omitempty,max=500"`
    Pending     bool            `json:"pending"`
}

type Transaction struct {
    TransactionDraft
    ID int `json:"id" validate:"required"`
}
```

The `openapi.yaml` describing this (`oapi-codegen` spec-first workflow):

```yaml
# backend-go/openapi.yaml (excerpt)
openapi: "3.1.0"
info: { title: FinancialApp API (Go), version: 1.0.0 }
paths:
  /api/accounts:
    get:
      operationId: listAccounts
      responses:
        "200":
          description: OK
          content: { application/json: { schema: { type: array, items: { $ref: "#/components/schemas/Account" } } } }
  /api/accounts/{accountId}:
    get:
      operationId: getAccount
      parameters:
        - { name: accountId, in: path, required: true, schema: { type: integer, minimum: 1 } }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: "#/components/schemas/Account" } } } }
        "404": { description: Not Found }
components:
  schemas:
    Account:
      type: object
      required: [id, name, type, balance, currency, ownerId]
      properties:
        id:       { type: integer }
        name:     { type: string, minLength: 1, maxLength: 100 }
        type:     { type: string, enum: [checking, savings, investment] }
        balance:  { type: number, minimum: 0 }
        currency: { type: string, enum: [USD, EUR, GBP, JPY] }
        ownerId:  { type: integer }
```

Handler with chi router:

```go
// backend-go/internal/handlers/accounts.go
package handlers

import (
    "encoding/json"
    "net/http"
    "github.com/go-chi/chi/v5"
    "app.financial/internal/model"
)

func (s *Server) ListAccounts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(s.accounts.All())
}

func (s *Server) GetAccount(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "accountId")
    account, ok := s.accounts.Find(id)
    if !ok {
        http.Error(w, `{"code":"NOT_FOUND","message":"Account not found"}`, http.StatusNotFound)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(account)
}
```

---

## 4. Per-Strategy Adjustments

**Strategy A -- openapi-zod-client.** Unchanged. Point the generator at the Go-emitted `openapi.yaml`:

```bash
npx openapi-zod-client backend-go/openapi.yaml \
  -o libs/shared/api-client/src/generated/generated-zodios.ts \
  --export-schemas --with-docs
```

**Strategy B -- co-equal diff.** The Go tag `omitempty` combined with whether a field is in `required: [...]` in the YAML decides the field's optionality. Be explicit: if a field is optional on the frontend (`z.string().optional()`), leave it out of the `required` array in the YAML. If it's required on the backend, include it. The diff tool catches mismatches.

Pointer-typed fields in Go (`*string`) imply "optional, omit when nil"; value-typed (`string`) imply "always present, even if empty". Match these intentionally with your Zod schemas. `*string` <-> `z.string().optional()` is the usual pairing.

**Strategy C -- datamodel-code-generator + ts-to-zod.** Unchanged. The pipeline doesn't care whether the spec was hand-authored, generated by `swaggo/swag`, or emitted by any other tool.

**Strategy D -- typescript-angular.** Unchanged. `oapi-codegen` and OpenAPI Generator coexist happily -- the former generates the Go server, the latter generates the Angular client, both from the same `openapi.yaml`. Make the Angular generator a Makefile target alongside the Go generator:

```makefile
# backend-go/Makefile
.PHONY: generate generate-server generate-angular-client

generate: generate-server generate-angular-client

generate-server:
	oapi-codegen -config oapi-codegen.yaml openapi.yaml

generate-angular-client:
	cd ../.. && npx openapi-generator-cli generate \
	  -i backend-go/openapi.yaml \
	  -g typescript-angular \
	  -o libs/shared/api-client-ng/src/lib/generated \
	  --additional-properties=ngVersion=21.0.0,providedIn=root,stringEnums=true,taggedUnions=true,useSingleRequestParameter=true,withInterfaces=true,enumUnknownDefaultCase=true
```

---

## 5. Backend-Specific Gotchas

**`omitempty` semantics.** `json:"description,omitempty"` means "omit this field from JSON output if it is the zero value (empty string, nil pointer, etc.)". This doesn't match "optional" in OpenAPI's sense. An optional field should use a pointer type *and* the `omitempty` tag; the field is absent from JSON output when `nil`, and Go unmarshals `{}` into a struct with `nil` for that field. Document this convention in the backend README and stick to it.

**Integer and time format inconsistency.** `int` in Go could be 32 or 64 bits depending on the platform. Be explicit: use `int64` in struct fields, and declare `type: integer, format: int64` in the YAML. For timestamps, use `time.Time` with RFC3339 formatting (Go default) and `type: string, format: date-time` in OpenAPI. For dates (no time), use `time.Time` but strip the time portion -- or use the `civil.Date` type from `cloud.google.com/go/civil`.

**`decimal.Decimal` marshal shape.** The `shopspring/decimal` package marshals as a string by default (`"5432.10"`), preserving precision. OpenAPI's `type: number` doesn't declare format strictly, but the generators typically interpret number-typed fields as `z.number()`. If you need to preserve decimal precision end-to-end, either use `type: string, pattern: "^\\d+(\\.\\d{1,4})?$"` in the YAML or post-process the Zod schema with `.transform((s) => new Decimal(s))`.

**Gin vs Echo vs chi vs net/http.** All work identically for OpenAPI purposes. Pick based on team preference. Gin gives you `c.ShouldBindJSON(&draft)` with automatic validator integration; Echo has `c.Bind(&draft)` + `c.Validate(draft)`; chi is a router only, no binding. The generated Angular client doesn't know or care.

**CORS.** Go has `github.com/rs/cors` for a full-featured middleware. Configure with explicit origins and credentials -- like Express, CORS is opt-in, not free.

---

## 6. Running the Contract Tests Against This Backend

Playwright `webServer` spawns the Go binary:

```typescript
// apps/financial-app-e2e/playwright.config.ts
webServer: [
  { command: 'npx nx serve financial-app', url: 'http://localhost:4200' },
  {
    command: 'go run ./cmd/server',
    url: 'http://localhost:8000/api/health',
    cwd: '../../backend-go',
  },
],
```

CI needs Go and Node:

```yaml
- uses: actions/setup-go@v5
  with: { go-version: '1.23' }
- uses: actions/setup-node@v4
  with: { node-version: '22' }
- working-directory: backend-go
  run: make generate-server
- run: npm ci
- run: npm run generate:ng-client
- run: git diff --exit-code -- libs/shared/api-client-ng backend-go
- run: CONTRACT=1 npx nx e2e financial-app-e2e
```

`make generate-server` regenerates the Go server stubs from `openapi.yaml`. The `git diff --exit-code` catches drift on both the Go side (did someone forget to regenerate?) and the Angular side (same question).

---

## 7. See Also

- [Appendix B0 §4](ch52-appendix-b0-backend-fastapi.md#4-strategy-b----zod--pydantic-co-equal-with-ci-diff) -- Strategy B diff; the `omitempty` convention is the Go-specific version of Spring's `Optional<T>` consideration.
- [Appendix B0 §6](ch52-appendix-b0-backend-fastapi.md#6-strategy-d----openapi-generators-typescript-angular-for-an-angular-native-client-library) -- Strategy D details; the Makefile pattern here is the Go-idiomatic equivalent of Spring's Gradle plugin.
- [Chapter 2](ch02-signal-components.md) "Advanced HTTP" -- file uploads, SSE, and streaming responses; Go's `http.ResponseWriter` is the natural streaming surface for server-side events.
- [Appendix B1](ch53-ch53-ch53-appendix-b1-backend-express.md) and [Appendix B2](ch54-ch54-ch54-appendix-b2-backend-spring.md) -- equivalent recipes for Node + Express and Spring Boot.
