# Chapter 21: Security & OWASP Best Practices

A financial application that leaks account numbers through a reflected XSS attack is not merely a bug -- it is a breach. FinancialApp handles portfolio balances, transaction histories, and personally identifiable information. Every page that renders user-supplied content, every API call that carries credentials, and every third-party script loaded from a CDN is an attack surface. Frontend security is not a backend concern that the Angular team can delegate away; it is a shared responsibility, and Angular provides strong defaults to help carry it.

Angular's security posture is best described as *secure by default*. The framework sanitizes interpolated values, refuses to compile user-generated templates at runtime, and provides escape hatches that are deliberately verbose so developers cannot bypass protections by accident. But defaults only protect you from attacks the framework anticipated. The OWASP Top 10:2025 catalogs the broader threat landscape, and several of its categories -- broken access control, supply chain attacks, insecure design -- require architectural decisions that no framework can make for you.

This chapter maps Angular's built-in protections to the threats that matter most for a financial SPA, walks through Content Security Policy configuration, and systematically addresses each OWASP Top 10:2025 category with Angular-specific mitigations. Authentication and session management are covered in depth in [Chapter 16](ch16-auth-patterns.md); this chapter focuses on the non-auth security concerns that are equally critical and often overlooked.

> **Companion code:** `financial-app/libs/shared/security/` contains sanitization utilities, a security header interceptor, and a CSP violation reporter.

---

## Angular's Built-in XSS Protection

Cross-site scripting (XSS) is the most common client-side attack vector. An attacker injects malicious script into a page, and the browser executes it with the same privileges as your application code. Angular treats all values as untrusted by default and sanitizes them before inserting them into the DOM.

### Template Sanitization

When you bind a value with interpolation or property binding, Angular inspects the target context and sanitizes the value accordingly. A binding like `<p>{{ userComment }}</p>` is safe even if `userComment` contains `<script>alert('xss')</script>` -- Angular escapes the HTML entities before rendering. The same applies to property bindings like `[innerHTML]`, where Angular strips dangerous elements and attributes (event handlers, `<script>`, `<object>`) while preserving safe formatting tags.

This sanitization is not a simple string replacement. Angular maintains a whitelist-based sanitizer that understands HTML structure. It parses the input, walks the DOM tree, and removes anything that could execute code. The result is that developers can safely render user-provided rich text without building custom sanitization logic.

### SecurityContext

Angular categorizes bindings into four security contexts, each with its own sanitization rules:

| Context | Applied To | Sanitization Behavior |
|---|---|---|
| `SecurityContext.HTML` | `[innerHTML]` | Strips scripts, event handlers, dangerous elements |
| `SecurityContext.STYLE` | `[style]` bindings | Removes `url()` and `expression()` patterns |
| `SecurityContext.URL` | `[href]`, `[src]` | Blocks `javascript:` and `data:` schemes |
| `SecurityContext.RESOURCE_URL` | `<iframe [src]>`, `<script [src]>` | Blocks all values -- requires explicit trust |

The `RESOURCE_URL` context is the most restrictive. Angular will not allow any dynamically bound value to load an external resource (an iframe source, a script tag) without an explicit bypass. This prevents attackers from injecting a malicious iframe that loads a phishing page within FinancialApp's layout.

### DomSanitizer

Occasionally, you need to bind a value that Angular's sanitizer would strip. A common case in FinancialApp is rendering a chart tooltip as trusted HTML returned from a charting library, or embedding a PDF viewer iframe with a dynamically constructed URL. The `DomSanitizer` service provides bypass methods for each security context:

```typescript
// libs/shared/security/src/lib/safe-html.pipe.ts
import { inject, Pipe, PipeTransform } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Pipe({ name: 'safeHtml' })
export class SafeHtmlPipe implements PipeTransform {
  private readonly sanitizer = inject(DomSanitizer);

  transform(value: string): SafeHtml {
    return this.sanitizer.bypassSecurityTrustHtml(value);
  }
}
```

This pipe works, but it is a loaded gun. The word "bypass" is not an accident -- you are telling Angular to skip sanitization entirely and trust the value. If that value comes from user input, you have created an XSS vulnerability. The `bypassSecurityTrust*` family should only be used when:

1. The value comes from a source you fully control (your own server, a compiled template).
2. You have validated or sanitized the content yourself before passing it through.
3. You have documented *why* the bypass is necessary, so future developers do not extend it to untrusted sources.

For FinancialApp, the `safeHtml` pipe is used exclusively for server-rendered report fragments that have already been sanitized server-side. It is never exposed to raw user input.

### Trusted Types

Trusted Types is a browser API that enforces type-safe DOM manipulation at the platform level. When enabled, the browser rejects any attempt to assign a raw string to an injection sink like `innerHTML`, `eval`, or `script.src` -- unless the value is wrapped in a Trusted Type created by a registered policy.

Angular supports Trusted Types out of the box. When Angular detects that the browser enforces Trusted Types, its internal sanitizer produces `TrustedHTML` values instead of raw strings. No application code changes are required. To enable enforcement, add the Trusted Types CSP directive (covered in the next section) and Angular takes care of the rest.

This means FinancialApp gets defense in depth: Angular sanitizes values before insertion, and the browser rejects any raw string that somehow bypasses the framework's sanitizer. Two independent layers protecting the same injection sinks.

---

## Content Security Policy

Content Security Policy (CSP) is an HTTP response header (or meta tag) that tells the browser which sources of content are permitted. A well-configured CSP is the single most effective mitigation against XSS after framework-level sanitization, because it prevents injected scripts from executing even if they make it into the DOM.

### Why CSP Matters for SPAs

A traditional server-rendered page loads a handful of scripts. An Angular SPA loads an application bundle, vendor chunks, lazy-loaded feature modules, web fonts, and potentially third-party analytics or charting libraries. Each of these is a potential vector. CSP lets you enumerate exactly which sources are allowed and block everything else.

Without CSP, an attacker who finds an injection point can load an external script from any domain. With CSP, the browser blocks the request because the attacker's domain is not in the allowlist.

### Configuring CSP for Angular

Angular's renderer injects inline styles for some component animations and structural directives. A strict `style-src` policy that blocks all inline styles would break these features. The recommended approach is a nonce-based policy: the server generates a unique nonce for each response and includes it in both the CSP header and a `ngCspNonce` attribute on the root element.

```html
<!-- index.html (server sets nonce dynamically) -->
<!doctype html>
<html lang="en">
<head>
  <meta http-equiv="Content-Security-Policy"
    content="default-src 'self';
             script-src 'self' 'nonce-{{CSP_NONCE}}';
             style-src 'self' 'nonce-{{CSP_NONCE}}';
             img-src 'self' https://cdn.financialapp.com;
             font-src 'self' https://fonts.gstatic.com;
             connect-src 'self' https://api.financialapp.com;
             frame-ancestors 'none';
             base-uri 'self';
             form-action 'self';
             require-trusted-types-for 'script';">
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <app-root ngCspNonce="{{CSP_NONCE}}"></app-root>
</body>
</html>
```

The `{{CSP_NONCE}}` placeholder is replaced by the server on each request with a cryptographically random value. Angular reads the `ngCspNonce` attribute and applies it to any inline styles it generates. The `frame-ancestors 'none'` directive prevents FinancialApp from being embedded in an iframe on another domain (clickjacking protection). The `require-trusted-types-for 'script'` directive activates the Trusted Types enforcement discussed earlier.

For local development, you can use a relaxed policy with `'unsafe-inline'` for styles, but production builds should always use the nonce-based approach.

### CSP Violation Reporting

When the browser blocks a resource that violates your CSP, it fires a `SecurityPolicyViolationEvent`. Capturing these events lets you detect misconfigured policies, attempted attacks, and third-party scripts that you forgot to allowlist:

```typescript
// libs/shared/security/src/lib/csp-violation-reporter.ts
import { inject, Injectable, NgZone } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class CspViolationReporter {
  private readonly http = inject(HttpClient);
  private readonly zone = inject(NgZone);

  initialize(): void {
    this.zone.runOutsideAngular(() => {
      document.addEventListener('securitypolicyviolation', (event) => {
        const report = {
          blockedUri: event.blockedURI,
          violatedDirective: event.violatedDirective,
          originalPolicy: event.originalPolicy,
          documentUri: event.documentURI,
          timestamp: new Date().toISOString(),
        };
        this.http
          .post('/api/security/csp-violations', report)
          .subscribe();
      });
    });
  }
}
```

The listener runs outside Angular's zone to avoid triggering unnecessary change detection cycles. In production, FinancialApp routes these reports to a security information and event management (SIEM) system where the security team can correlate CSP violations with other suspicious activity.

---

## OWASP Top 10:2025 Mapped to Angular

The OWASP Top 10 is the industry-standard ranking of the most critical web application security risks. The 2025 edition reshuffled several categories and added new ones. Not every category maps neatly to frontend code -- some are primarily server-side concerns -- but each has implications for how you build and deploy an Angular application.

### A01: Broken Access Control

Route guards are the first thing Angular developers reach for when restricting access. The critical mistake is treating them as security enforcement rather than UX convenience. A route guard runs in the browser, which the user controls. An attacker can bypass it entirely by calling the API directly.

```typescript
// libs/shared/security/src/lib/guards/role.guard.ts
export const requireRole = (requiredRole: string): CanActivateFn => {
  return () => {
    const auth = inject(AuthService);
    const router = inject(Router);
    const userRoles = auth.currentUser()?.roles ?? [];

    if (userRoles.includes(requiredRole)) {
      return true;
    }
    return router.createUrlTree(['/unauthorized']);
  };
};
```

This guard prevents a regular user from navigating to the admin dashboard, which is good UX. But the server must independently verify the user's role on every API request:

```typescript
// Server-side middleware (Node.js / Express)
function requireServerRole(role: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRoles = req.session?.user?.roles ?? [];
    if (!userRoles.includes(role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}
app.delete('/api/admin/users/:id', requireServerRole('admin'), deleteUserHandler);
```

The rule is simple: **every access control decision must be enforced server-side**. The Angular guard is a courtesy, not a gate.

### A02: Security Misconfiguration

Security misconfiguration covers a broad range of mistakes, from exposing debug endpoints in production to missing security headers. For an Angular application, three areas deserve particular attention.

**Environment files.** Angular's `environment.ts` files are compiled into the production bundle. Never put API keys, database credentials, or secrets in them. Anything in these files is visible to anyone who opens the browser's developer tools. FinancialApp's environment files contain only non-sensitive configuration -- API base URLs and feature flags.

**Security headers.** The server that hosts the Angular bundle should set these response headers:

| Header | Value | Purpose |
|---|---|---|
| `X-Content-Type-Options` | `nosniff` | Prevents MIME-type sniffing |
| `X-Frame-Options` | `DENY` | Prevents clickjacking (fallback for older browsers) |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Enforces HTTPS for one year |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limits referrer information leakage |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Disables unused browser features |

**CORS.** The API server's CORS configuration should allowlist only the exact origins that FinancialApp runs on. A wildcard `Access-Control-Allow-Origin: *` on a financial API is an invitation for cross-origin data theft.

### A03: Supply Chain

Your `node_modules` folder is the largest attack surface you do not write. A single compromised dependency can exfiltrate credentials, inject cryptominers, or backdoor your build pipeline.

Defense starts with hygiene:

```bash
# Audit dependencies for known vulnerabilities
npm audit

# Install with lockfile integrity verification
npm ci

# Prevent install scripts from running (review manually first)
npm install --ignore-scripts
```

The `--ignore-scripts` flag deserves special attention. Many npm packages run arbitrary code during installation via `postinstall` scripts. For FinancialApp, the CI pipeline installs dependencies with `--ignore-scripts` and explicitly runs only the install scripts that the team has reviewed and allowlisted.

Pin your lockfile in version control. The `package-lock.json` file ensures that every environment -- developer machines, CI, staging, production -- installs the exact same dependency tree. A lockfile diff in a pull request is a signal to review what changed and why.

### A04: Cryptographic Failures

The most common cryptographic failure in SPAs is not about encryption algorithms -- it is about token storage. Storing access tokens or session identifiers in `localStorage` makes them accessible to any JavaScript running on the page, including injected scripts from XSS attacks.

FinancialApp uses the BFF pattern described in [Chapter 16](ch16-auth-patterns.md), which eliminates client-side token storage entirely. For applications that must store tokens in the browser, `sessionStorage` is marginally better (cleared when the tab closes) but still vulnerable to XSS. `HttpOnly` cookies are the only browser-native storage mechanism immune to JavaScript theft.

Beyond token storage, two rules apply to any financial application:

1. **Enforce HTTPS everywhere.** The `Strict-Transport-Security` header (HSTS) tells the browser to never make an unencrypted request. Without it, a network attacker can downgrade the connection and intercept traffic.
2. **Never put PII in URLs or query parameters.** URLs are logged by proxies, stored in browser history, and sent in `Referer` headers. A URL like `/api/users?ssn=123-45-6789` leaks sensitive data to every intermediary between the browser and the server.

### A05: Injection

Angular's template compiler is the primary defense against injection attacks. Unlike frameworks that allow arbitrary string-based template construction, Angular compiles templates ahead of time (AOT). There is no runtime `eval()` of template expressions, no `new Function()` from user input, and no way for an attacker to inject template syntax that the compiler will interpret.

The danger arises when developers circumvent these protections:

```typescript
// DANGEROUS: never do this
const userInput = getUserComment();
const template = `<div>${userInput}</div>`;
element.innerHTML = template;
```

This bypasses Angular's sanitizer entirely. The safe alternative is to use Angular's binding:

```html
<!-- Safe: Angular sanitizes the value -->
<div [innerHTML]="userComment()"></div>
```

For HTTP requests, Angular's `HttpClient` naturally parameterizes URLs and serializes request bodies as JSON. Constructing URLs by string concatenation with user input creates injection risks not in Angular, but in the API if the server uses those values in SQL queries or shell commands:

```typescript
// libs/shared/security/src/lib/api/account.service.ts

// Safe: parameterized via HttpParams
loadTransactions(accountId: number, filters: TransactionFilter) {
  const params = new HttpParams()
    .set('accountId', accountId)
    .set('startDate', filters.startDate)
    .set('endDate', filters.endDate);
  return this.http.get<Transaction[]>('/api/transactions', { params });
}
```

Using `HttpParams` ensures values are properly URL-encoded and clearly separated from the URL structure.

### A06: Insecure Design

Insecure design is about missing security controls at the architectural level, not implementation bugs. For FinancialApp, two design-level concerns stand out.

**Input validation.** Frontend validation is a UX feature, not a security boundary -- but it catches mistakes early and reduces the load on server-side validation. The signal-based forms from [Chapter 6](ch06-signal-forms.md) make validation declarative and composable. A transfer form that validates the amount is positive, the account exists, and the description contains no control characters prevents obviously malicious input from reaching the server:

```typescript
// libs/accounts/src/lib/transfer-form.component.ts
amount = new FormField<number>(0, {
  validators: [
    required(),
    min(0.01),
    max(1_000_000),
  ],
});
```

**Error disclosure.** A stack trace in an error message reveals framework versions, file paths, and internal service names -- intelligence an attacker can use to craft targeted exploits. FinancialApp's error handling (detailed in [Chapter 23](ch23-error-handling.md)) maps all server errors to generic user-facing messages and logs the full detail server-side.

### A07: Authentication Failures

Authentication and session management are covered comprehensively in [Chapter 16](ch16-auth-patterns.md), including cookie security attributes, XSRF protection, OAuth 2 with PKCE, and the Backend-for-Frontend pattern. FinancialApp delegates authentication to a dedicated authorization server and manages sessions through `HttpOnly` cookies via the BFF, which eliminates the most common authentication failure modes for SPAs: token theft through XSS, insecure token storage, and missing session expiration.

### A08: Software and Data Integrity

Integrity failures occur when code or data is modified without verification. For a frontend application, the primary risks are a compromised build pipeline and tampered CDN assets.

**CI verification.** FinancialApp's CI pipeline verifies the integrity of every build artifact. The `npm ci` command (not `npm install`) ensures the installed dependencies exactly match the lockfile. If the lockfile has been tampered with, the install fails. The pipeline also runs `npm audit` and blocks deployment if critical vulnerabilities are found.

**Subresource Integrity (SRI).** If you load scripts or stylesheets from a CDN, SRI ensures the browser verifies a cryptographic hash before executing them:

```html
<script
  src="https://cdn.financialapp.com/analytics.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxAh0zaw9EYEguB7IOhTqw2K3zTvDJ"
  crossorigin="anonymous">
</script>
```

If an attacker compromises the CDN and modifies `analytics.js`, the browser detects the hash mismatch and refuses to execute the script. For FinancialApp, all CDN-hosted assets include SRI hashes generated during the build process.

### A09: Security Logging and Monitoring

An application that cannot detect attacks cannot respond to them. The CSP violation reporter shown earlier in this chapter is one component of a broader security monitoring strategy. FinancialApp also logs authentication failures, authorization denials, and suspicious request patterns.

On the frontend, security-relevant events are captured and forwarded through the `GlobalErrorHandler` (detailed in [Chapter 23](ch23-error-handling.md)):

```typescript
// libs/shared/security/src/lib/security-event.service.ts
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

export type SecurityEventType =
  | 'csp-violation'
  | 'auth-failure'
  | 'forbidden-navigation'
  | 'sanitization-bypass';

@Injectable({ providedIn: 'root' })
export class SecurityEventService {
  private readonly http = inject(HttpClient);

  report(type: SecurityEventType, detail: Record<string, unknown>): void {
    const event = {
      type,
      detail,
      url: location.href,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent,
    };
    this.http.post('/api/security/events', event).subscribe();
  }
}
```

This service is intentionally fire-and-forget. A failed security report should never disrupt the user's workflow. The server-side logging pipeline is where alerts, dashboards, and incident response workflows are built.

### A10: Mishandling of Exceptional Conditions

Unhandled exceptions are a security concern, not just a reliability concern. A raw server error message displayed in the UI can reveal database schema details, internal service URLs, or middleware versions. An unhandled promise rejection that crashes the application can leave the user in an inconsistent state where they believe a transaction succeeded when it did not.

FinancialApp addresses this with two principles. First, server errors are never shown to the user verbatim. The `GlobalErrorHandler` (see [Chapter 23](ch23-error-handling.md)) maps HTTP error responses to safe, generic messages:

```typescript
// libs/shared/security/src/lib/sanitize-error.ts
export function sanitizeErrorMessage(status: number): string {
  switch (true) {
    case status === 403:
      return 'You do not have permission to perform this action.';
    case status === 404:
      return 'The requested resource was not found.';
    case status >= 500:
      return 'An unexpected error occurred. Please try again later.';
    default:
      return 'Something went wrong.';
  }
}
```

Second, error boundaries ensure that a failure in one feature does not cascade. If the portfolio chart fails to load, the rest of the dashboard remains functional. The user sees a recovery message, not a blank screen with a console full of stack traces.

---

## Security Header Interceptor

While security headers are primarily a server-side concern, an `HttpInterceptorFn` can add client-side security headers to outgoing requests. This is useful for signaling the client's security capabilities to the server and for adding correlation headers that link frontend events to backend logs:

```typescript
// libs/shared/security/src/lib/interceptors/security-header.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const securityHeaderInterceptor: HttpInterceptorFn = (req, next) => {
  const securedReq = req.clone({
    setHeaders: {
      'X-Content-Type-Options': 'nosniff',
      'X-Request-Id': crypto.randomUUID(),
    },
  });
  return next(securedReq);
};
```

Register the interceptor alongside the others in `provideHttpClient`:

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([securityHeaderInterceptor]),
    ),
  ],
};
```

The `X-Request-Id` header is particularly valuable for security investigations. When a CSP violation or suspicious request pattern is detected, the request ID lets the security team correlate the frontend event with the exact backend log entry, cutting investigation time from hours to minutes.

---

## Security-Focused ESLint Rules

Static analysis catches security mistakes before they reach production. A handful of ESLint rules, configured in FinancialApp's shared lint configuration, flag the most dangerous patterns:

```jsonc
// .eslintrc.json (security-relevant rules)
{
  "rules": {
    "no-eval": "error",
    "no-implied-eval": "error",
    "no-new-func": "error",
    "no-script-url": "error",
    "@angular-eslint/template/no-any": "error"
  },
  "overrides": [
    {
      "files": ["*.ts"],
      "rules": {
        "no-restricted-properties": [
          "error",
          {
            "object": "document",
            "property": "write",
            "message": "document.write is a security risk. Use Angular's renderer."
          }
        ],
        "no-restricted-syntax": [
          "error",
          {
            "selector": "MemberExpression[property.name='innerHTML']",
            "message": "Direct innerHTML assignment bypasses Angular's sanitizer. Use [innerHTML] binding instead."
          },
          {
            "selector": "CallExpression[callee.property.name=/^bypassSecurityTrust/]",
            "message": "bypassSecurityTrust* requires security review. Add a comment explaining why the bypass is necessary."
          }
        ]
      }
    }
  ]
}
```

These rules do not prevent legitimate use -- they force a conversation. A developer who needs `bypassSecurityTrustHtml` must acknowledge the lint warning, add a justification comment, and pass code review. The friction is the point. Security bypasses should be deliberate, documented, and rare.

For teams using Nx, these rules can be enforced at the workspace level so they apply to every library and application consistently, using the shared configuration patterns from [Chapter 14](ch14-monorepos-libraries.md).

---

## Summary

Security is not a feature you add to FinancialApp -- it is a property the application either has or does not. Angular provides strong defaults, but defaults only protect against anticipated attacks.

- **Angular sanitizes all interpolated values** by security context (HTML, Style, URL, Resource URL). The `bypassSecurityTrust*` methods exist for legitimate use cases but require explicit justification and code review.
- **Trusted Types** provide a browser-level enforcement layer that complements Angular's sanitizer. When both are active, two independent systems protect DOM injection sinks.
- **Content Security Policy** restricts which sources the browser will load content from. A nonce-based CSP is the recommended approach for Angular applications. CSP violation reporting surfaces misconfigurations and attack attempts.
- **Route guards are UX, not security.** Every access control decision must be enforced server-side (A01).
- **Environment files are public.** Never commit secrets to files that are compiled into the browser bundle (A02).
- **Your dependencies are your attack surface.** Audit them, verify lockfile integrity, and restrict install scripts (A03).
- **Token storage matters.** The BFF pattern from [Chapter 16](ch16-auth-patterns.md) eliminates client-side token storage entirely. If you must store tokens in the browser, `HttpOnly` cookies are the only option immune to XSS (A04).
- **Angular's AOT compiler prevents template injection.** Never construct templates from user input or use direct `innerHTML` assignment (A05).
- **Frontend validation is UX, not a security boundary**, but it reduces the blast radius of malformed input (A06).
- **Log security events and correlate them** with request IDs so your security team can investigate incidents efficiently (A09).
- **Never display raw server errors to users.** Map them to generic messages and log the details server-side (A10).

Static analysis -- ESLint rules that flag `innerHTML`, `eval`, and `bypassSecurityTrust*` -- catches the most dangerous patterns before they reach production. These rules do not replace security reviews, but they ensure that every bypass is deliberate and documented.

Security is a practice, not a checklist. The protections described in this chapter reduce risk; they do not eliminate it. Regular dependency audits, penetration testing, and security-focused code reviews are the ongoing work that keeps FinancialApp trustworthy.
