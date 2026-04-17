# Appendix B2: Backend Adapter -- Spring Boot

This appendix is the Spring Boot counterpart to [Appendix B](appendix-b-e2e-workflow.md). Read the main appendix first; this one shows only the deltas for a Java backend.

Spring Boot is the enterprise default in 2026 for JVM-backed APIs. The OpenAPI story is mature: **`springdoc-openapi`** reads Jakarta Bean Validation annotations on records or POJOs and exposes `/v3/api-docs` (JSON) and `/swagger-ui.html` (interactive docs). The four Angular-side strategies consume that spec identically to the FastAPI case.

Strategy D is particularly natural in Java shops: OpenAPI Generator has first-class Gradle and Maven plugins, so the Angular client library becomes a module in the same JVM build pipeline -- the generator runs as part of `./gradlew build`, the TypeScript output is bundled into a Jenkins/Artifactory artifact, and multi-app consumers depend on it through the same dependency graph as their Java libraries.

---

## 1. Overview

**`springdoc-openapi`** is the de facto SpringBoot + OpenAPI bridge. Add the starter, run the app, and `/v3/api-docs` is live. It reads:

- Controller method signatures (`@GetMapping`, `@PostMapping`, `@PathVariable`, `@RequestBody`).
- Jakarta Bean Validation annotations (`@NotNull`, `@NotBlank`, `@Min`, `@Max`, `@Pattern`, `@Size`, `@Positive`, `@DecimalMin`).
- Swagger annotations (`@Operation`, `@Schema`) for finer control when the defaults aren't enough.
- Jackson serialization metadata (property names, inclusion rules, custom serializers).

The combination gives you a spec that matches what actually ships -- validation annotations are enforced at runtime by Spring's validator, and JSON shapes follow Jackson's configured output. Drift between "what the spec says" and "what the server produces" is rare if Jackson is configured consistently (see Gotchas below).

Prefer Spring Boot + springdoc when:

- The backend team is Java-first and uses Jakarta Validation anyway.
- You're running on the JVM for reasons unrelated to Angular (existing ecosystem, Kafka integrations, banking frameworks).
- You want Strategy D's Angular library shipped as part of the JVM artifact pipeline.

---

## 2. Installing the Tooling

Gradle (Kotlin DSL) for a Spring Boot 3.5+ project targeting Java 21:

```kotlin
// backend-spring/build.gradle.kts
plugins {
  id("org.springframework.boot") version "3.5.1"
  id("io.spring.dependency-management") version "1.1.7"
  id("org.openapi.generator") version "7.10.0"
  kotlin("jvm") version "2.1.0"
}

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-validation")
  implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.7.0")
}
```

The `org.openapi.generator` plugin is optional but recommended for Strategy D -- it runs OpenAPI Generator as a Gradle task, allowing the Angular client library to be built alongside the JAR.

The `/v3/api-docs` endpoint is available once the app starts. Dump it to disk with:

```bash
./gradlew bootRun &
curl http://localhost:8080/v3/api-docs > ../openapi.json
```

Or use springdoc's build-time generator plugin to emit the spec without running the server:

```kotlin
plugins { id("com.github.joschi.licenser") version "0.6.1" }
openApi {
  apiDocsUrl.set("http://localhost:8080/v3/api-docs")
  outputDir.set(file("$rootDir/.."))
  outputFileName.set("openapi.json")
}
```

---

## 3. Equivalent of the FastAPI Example

Java records + Jakarta Validation:

```java
// backend-spring/src/main/java/app/financial/model/Account.java
package app.financial.model;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record Account(
    @NotNull Integer id,
    @NotBlank @Size(min = 1, max = 100) String name,
    @NotNull AccountType type,
    @NotNull @DecimalMin("0.0") BigDecimal balance,
    @NotNull CurrencyCode currency,
    @NotNull Integer ownerId
) {}

public enum AccountType { checking, savings, investment }
public enum CurrencyCode { USD, EUR, GBP, JPY }
```

```java
// backend-spring/src/main/java/app/financial/model/TransactionDraft.java
package app.financial.model;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;
import java.time.LocalDate;

public record TransactionDraft(
    @NotNull Integer accountId,
    @NotNull @DecimalMin(value = "0.01", inclusive = false) BigDecimal amount,
    @NotNull TransactionType type,
    @NotBlank @Size(min = 1, max = 50) String category,
    @NotNull LocalDate date,
    @Size(max = 500) String description,
    boolean pending
) {}

public enum TransactionType { credit, debit }
```

```java
// backend-spring/src/main/java/app/financial/web/AccountsController.java
package app.financial.web;

import app.financial.model.Account;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Tag(name = "accounts", description = "Account CRUD operations.")
@RestController
@RequestMapping("/api/accounts")
public class AccountsController {

  private final AccountRepository accounts;

  public AccountsController(AccountRepository accounts) {
    this.accounts = accounts;
  }

  @Operation(summary = "List all accounts")
  @GetMapping
  public List<Account> listAccounts() {
    return accounts.findAll();
  }

  @Operation(summary = "Get an account by id")
  @GetMapping("/{accountId}")
  public ResponseEntity<Account> getAccount(@PathVariable @Min(1) Integer accountId) {
    return accounts.findById(accountId)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
  }
}
```

Jackson default settings produce `ownerId` (camelCase) automatically from `ownerId()` accessor, matching the wire format the Angular frontend expects. No `@JsonProperty` annotations needed for the common case.

---

## 4. Per-Strategy Adjustments

**Strategy A -- openapi-zod-client.** Point the generator at the springdoc-produced spec:

```bash
curl http://localhost:8080/v3/api-docs > openapi.json
npx openapi-zod-client openapi.json \
  -o libs/shared/api-client/src/generated/generated-zodios.ts \
  --export-schemas --with-docs
```

The spec uses `BigDecimal` for `balance` and `amount`. `openapi-zod-client` maps these to `z.number()` by default. If you need strict decimal precision on the frontend (rare for display, common for arithmetic), post-process with a `.transform()` that parses to a `bigint` or a fixed-point representation.

**Strategy B -- co-equal diff.** The Zod schemas on the frontend describe constraints in JavaScript idioms (`z.number().gt(0.01)`); Spring's `@DecimalMin("0.01")` maps to OpenAPI's `exclusiveMinimum: 0.01`. The diff tool handles these correctly. One subtlety: `Optional<T>` (Java) vs `@Nullable` (kotlin) produces different OpenAPI shapes. Standardize on one style in the backend style guide or the diff tool will flag false positives.

**Strategy C -- datamodel-code-generator + ts-to-zod.** Unchanged. `datamodel-code-generator` handles `allOf` inheritance produced by Java record composition cleanly.

**Strategy D -- typescript-angular via Gradle plugin.** This is the killer feature for Java shops. Add the OpenAPI Generator plugin task:

```kotlin
openApiGenerate {
  generatorName.set("typescript-angular")
  inputSpec.set("$rootDir/build/api-docs/openapi.json")
  outputDir.set("$rootDir/../libs/shared/api-client-ng/src/lib/generated")
  additionalProperties.set(mapOf(
    "ngVersion" to "21.0.0",
    "providedIn" to "root",
    "fileNaming" to "kebab-case",
    "stringEnums" to "true",
    "taggedUnions" to "true",
    "useSingleRequestParameter" to "true",
    "withInterfaces" to "true",
    "supportsES6" to "true",
    "enumPropertyNaming" to "PascalCase",
    "enumUnknownDefaultCase" to "true",
  ))
}
```

Now `./gradlew openApiGenerate` produces the Angular client library as part of the JVM build. Wire it into `./gradlew build` via task dependency so CI runs the entire pipeline in one JDK + Node setup.

---

## 5. Backend-Specific Gotchas

**Jackson date/time format.** `LocalDate` serializes as ISO-8601 date strings (`"2026-04-15"`) by default -- matches Pydantic. `Instant` serializes as nanosecond-precision timestamp strings unless `spring.jackson.serialization.write-dates-as-timestamps=false` is set. Pin this in `application.properties` to avoid generator confusion.

**`Optional<T>` in controller return types.** Springdoc handles `Optional<Account>` as "nullable Account" in the spec, but generator behavior varies. Prefer returning `ResponseEntity<Account>` with explicit 200/404 branches (as in the AccountsController above) for the clearest spec output.

**`MultipartFile` uploads.** Spring Boot accepts them via `@RequestParam("file") MultipartFile file`. The resulting OpenAPI is a `multipart/form-data` schema with `type: string, format: binary`. OpenAPI Generator's `typescript-angular` maps this to a method taking a `Blob` parameter. The generated Angular method handles `FormData` construction automatically -- you just pass the `File` from an `<input type="file">`. See [Chapter 2 "Advanced HTTP"](ch02-signal-components.md) for the frontend side.

**Record vs POJO.** Records (Java 14+) give you the cleanest OpenAPI output. POJOs with Lombok `@Data`/`@Value` work but emit extra properties the generator may or may not filter. Prefer records for new code.

**Validation groups.** Spring's `@Validated` with groups (`@Validated(Create.class)` / `@Validated(Update.class)`) produces different constraints for different contexts. Springdoc supports this but the generators sometimes collapse group-specific rules. If you rely heavily on groups, add `@Schema` overrides to document the final shape explicitly.

**Port mismatch in dev.** Spring Boot defaults to 8080; FastAPI examples default to 8000. Adjust `apps/financial-app/src/environments/environment.contract.ts` to match your Spring port, or run Spring on 8000 via `server.port=8000`.

---

## 6. Running the Contract Tests Against This Backend

Update the Playwright `webServer` config to spawn Spring Boot:

```typescript
// apps/financial-app-e2e/playwright.config.ts
webServer: [
  { command: 'npx nx serve financial-app', url: 'http://localhost:4200' },
  {
    command: './gradlew bootRun -x test',
    url: 'http://localhost:8080/actuator/health',
    cwd: '../../backend-spring',
    timeout: 60_000, // JVM boot is slower than Node/Python
  },
],
```

CI requires both a JDK and Node:

```yaml
- uses: actions/setup-java@v4
  with: { java-version: '21', distribution: 'temurin' }
- uses: actions/setup-node@v4
  with: { node-version: '22' }
- run: ./gradlew openApiGenerate
- run: git diff --exit-code -- libs/shared/api-client-ng
- run: npm ci
- run: npx playwright test --config=apps/financial-app-e2e/playwright.config.ts
```

`openApiGenerate` is a Gradle task; it's the JVM-native equivalent of Node's `npm run generate:ng-client`.

---

## 7. See Also

- [Appendix B §6](appendix-b-e2e-workflow.md#6-strategy-d----openapi-generators-typescript-angular-for-an-angular-native-client-library) -- Strategy D details; the configuration flags explained there apply identically when the generator runs via the Gradle plugin.
- [Chapter 14](ch14-monorepos-libraries.md) -- publishing the generated Angular library as an npm package; relevant when the JVM build also produces an npm artifact.
- [Chapter 23](ch23-error-handling.md) -- retry interceptor patterns; unchanged on the Angular side regardless of backend.
- [Appendix B1](appendix-b1-backend-express.md) and [Appendix B3](appendix-b3-backend-go.md) -- equivalent recipes for Node + Express and Go.
