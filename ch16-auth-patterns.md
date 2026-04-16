# Modern Patterns for Authentication & Authorization

Every application that handles sensitive data must answer two questions: *who is this user?* (authentication) and *what are they allowed to do?* (authorization). For a financial application like FinancialApp -- where users view portfolio balances, execute trades, and manage account details -- getting these answers wrong means exposing real money to real attackers.

This chapter surveys the two dominant approaches to securing web applications: cookie-based authentication and token-based security. We will trace the evolution from simple session cookies through OAuth 2 and OpenID Connect, examine the threat models that shaped each protocol, and arrive at the current recommendation for Angular SPAs: the Backend-for-Frontend (BFF) pattern with server-side OAuth 2.

> **Prerequisites:** This chapter assumes familiarity with Angular's router guards and HTTP interceptors covered in [Chapter 12](ch12-initialization-routes.md). We will use `HttpInterceptorFn` and `CanActivateFn` extensively.

---

## Cookie-based Authentication

The simplest authentication model is also the oldest. The server creates a session after the user logs in, stores session state (in memory, a database, or a distributed cache), and sends back a cookie containing a session identifier. The browser automatically attaches this cookie to every subsequent request to the same origin.

For FinancialApp, a cookie-based login flow looks like this:

1. The user submits credentials to `POST /api/login`.
2. The server validates credentials, creates a session, and responds with `Set-Cookie: SESSION_ID=abc123`.
3. The browser stores the cookie and includes it on every request to the API.
4. The server looks up the session on each request to identify the user.

This model is straightforward, but the security of the entire system depends on the attributes set on that cookie.

### Security-Attributes for Cookies

A session cookie without proper attributes is an invitation for attack. Modern servers should set every security-relevant attribute:

| Attribute    | Value        | Purpose                                                       |
| ------------ | ------------ | ------------------------------------------------------------- |
| `HttpOnly`   | `true`       | Prevents JavaScript from reading the cookie, mitigating XSS.  |
| `Secure`     | `true`       | Cookie is only sent over HTTPS.                               |
| `SameSite`   | `Strict/Lax` | Restricts cross-origin sending, mitigating CSRF.              |
| `Path`       | `/api`       | Limits the cookie to API routes.                              |
| `Max-Age`    | `3600`       | Expires the session after a bounded time window.              |

A well-configured `Set-Cookie` header for FinancialApp looks like:

```
Set-Cookie: SESSION_ID=abc123; HttpOnly; Secure; SameSite=Lax; Path=/api; Max-Age=3600
```

The `HttpOnly` flag is especially critical. If an attacker manages to inject a script (XSS), they cannot steal the session cookie with `document.cookie`. The `Secure` flag ensures the cookie never travels over an unencrypted connection -- non-negotiable for financial data.

### Cookies and XSRF

Even with `SameSite` cookies, defense-in-depth demands explicit XSRF (Cross-Site Request Forgery) protection. The classic attack: a malicious page tricks the user's browser into making a state-changing request to FinancialApp while the session cookie is still valid. If the server cannot distinguish a legitimate request from a forged one, it will happily execute a fund transfer.

Angular provides built-in XSRF protection through `withXsrfConfiguration()`:

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',
        headerName: 'X-XSRF-TOKEN',
      })
    ),
  ],
};
```

The double-submit cookie pattern works as follows: the server sets a non-`HttpOnly` XSRF token cookie. Angular's `HttpClient` reads this cookie and attaches its value as a custom header (`X-XSRF-TOKEN`) on mutating requests. Since a cross-origin attacker cannot read cookies from FinancialApp's domain, they cannot forge the header, and the server rejects the request.

> **Note:** `SameSite=Lax` alone prevents CSRF on most modern browsers, but older browsers and edge cases (e.g., top-level navigations with `GET` side effects) still warrant the double-submit pattern.

---

## Token-based Security

Cookie-based authentication works well when the browser and the API share the same origin. But modern architectures often separate the frontend from the API, run multiple microservices, or integrate with third-party identity providers. This is where token-based security shines.

Instead of maintaining server-side sessions, the server issues a token -- a self-contained credential -- that the client stores and presents on each request. The token carries claims about the user and is cryptographically signed so the server can verify it without a session lookup.

### OAuth 2

OAuth 2 is an *authorization* framework (RFC 6749). It was not designed to authenticate users -- it was designed to let a user grant limited access to their resources on one service to another service, without sharing credentials.

The key roles in OAuth 2:

- **Resource Owner** -- the user (the FinancialApp customer).
- **Client** -- the application requesting access (FinancialApp's Angular SPA).
- **Authorization Server** -- the service that authenticates the user and issues tokens (e.g., Azure AD, Auth0, Keycloak).
- **Resource Server** -- the API that holds protected resources (FinancialApp's backend).

OAuth 2 defines several *grant types* (flows) for different scenarios. The critical insight is that OAuth 2 alone tells the client *what the user can access*, but not *who the user is*. That gap is filled by OpenID Connect.

### Authenticating Users with OpenID Connect

OpenID Connect (OIDC) is a thin identity layer built on top of OAuth 2. Where OAuth 2 provides an **access token** (authorizing API calls), OIDC adds an **ID token** (asserting the user's identity).

The ID token answers the authentication question:

- **Who is this user?** -- The `sub` (subject) claim uniquely identifies them.
- **When did they authenticate?** -- The `auth_time` claim records it.
- **How did they authenticate?** -- The `amr` (authentication methods) claim lists the factors used.

For FinancialApp, OIDC means we delegate the hard problem of credential management (password hashing, MFA enrollment, brute-force protection) to a dedicated authorization server, and our Angular application receives a signed assertion that the user is who they claim to be.

### JSON Web Token

Both the access token and the ID token are typically encoded as JSON Web Tokens (JWTs). A JWT has three Base64url-encoded parts separated by dots:

```
header.payload.signature
```

- **Header** -- specifies the signing algorithm (`RS256`, `ES256`).
- **Payload** -- contains the claims (`sub`, `iss`, `exp`, `aud`, roles, permissions).
- **Signature** -- the cryptographic signature that proves the token was issued by the authorization server and has not been tampered with.

An example decoded payload for a FinancialApp access token:

```json
{
  "sub": "user-42",
  "iss": "https://auth.financialapp.com",
  "aud": "https://api.financialapp.com",
  "exp": 1750000000,
  "scope": "portfolio:read trades:write",
  "roles": ["investor"]
}
```

> **Security warning:** Never store sensitive data in a JWT payload. The payload is encoded, not encrypted -- anyone who intercepts the token can decode and read the claims.

### OAuth 2 and OIDC Flows

OAuth 2 defines multiple grant types. The choice of flow depends on the type of client and the security requirements:

| Flow                              | Client Type            | Tokens Returned              | Suitable For             |
| --------------------------------- | ---------------------- | ---------------------------- | ------------------------ |
| Authorization Code                | Server-side app        | Access + ID + Refresh        | Traditional web apps     |
| Authorization Code + PKCE         | Public client (SPA)    | Access + ID (+ Refresh)      | SPAs, mobile apps        |
| Client Credentials                | Machine-to-machine     | Access only                  | Service accounts, daemons|
| ~~Implicit~~                      | ~~Public client~~      | ~~Access + ID~~              | **Deprecated.** Do not use. |
| ~~Resource Owner Password~~       | ~~Trusted client~~     | ~~Access + ID + Refresh~~    | **Deprecated.** Do not use. |

The **Authorization Code flow with PKCE** (Proof Key for Code Exchange) is the only recommended flow for public clients like Angular SPAs. The implicit flow is deprecated because it exposes tokens in URL fragments, making them vulnerable to browser history leaks and referer header exfiltration.

The PKCE extension protects against authorization code interception attacks by binding the code to the client that requested it:

```mermaid
sequenceDiagram
    participant User as User (Browser)
    participant SPA as FinancialApp (Angular)
    participant AS as Authorization Server
    participant API as Resource Server (API)

    SPA->>SPA: Generate code_verifier (random) + code_challenge (SHA256)
    SPA->>AS: GET /authorize?response_type=code&code_challenge=...&code_challenge_method=S256
    AS->>User: Show login page
    User->>AS: Enter credentials + MFA
    AS->>SPA: Redirect with authorization code
    SPA->>AS: POST /token (code + code_verifier)
    AS->>AS: Verify SHA256(code_verifier) == code_challenge
    AS->>SPA: Access token + ID token
    SPA->>API: GET /api/portfolio (Authorization: Bearer <access_token>)
    API->>API: Validate token signature + claims
    API->>SPA: Portfolio data
```

The `code_verifier` is a high-entropy random string generated by the SPA. The `code_challenge` is its SHA-256 hash. Even if an attacker intercepts the authorization code (e.g., via a malicious browser extension), they cannot exchange it for tokens without the original `code_verifier`.

### Client-side OAuth 2

Historically, Angular SPAs performed the full OAuth 2 flow in the browser. The application redirected to the authorization server, received tokens, and stored them in memory or `localStorage`. An `HttpInterceptorFn` then attached the access token to outgoing API requests:

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.accessToken();

  if (token && req.url.startsWith('/api')) {
    const authedReq = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
    return next(authedReq);
  }

  return next(req);
};
```

A route guard restricts navigation to authenticated users:

```typescript
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login']);
};
```

This approach works, but it has a fundamental security problem: **the browser is a hostile environment for storing tokens**. Any JavaScript running on the page -- including injected scripts from XSS vulnerabilities, compromised third-party libraries, or malicious browser extensions -- can access tokens stored in `localStorage`, `sessionStorage`, or even JavaScript closures (via prototype pollution or debugging tools).

For a financial application, this risk is unacceptable. If an attacker steals an access token, they can make API calls as the user -- viewing balances, initiating transfers -- until the token expires.

### Current Recommendation: Server-side OAuth 2

The industry consensus for securing SPAs has shifted decisively toward the **Backend-for-Frontend (BFF)** pattern. Rather than handling tokens in the browser, FinancialApp delegates the OAuth 2 flow to a lightweight backend proxy.

The BFF architecture works as follows:

1. **The Angular SPA never sees tokens.** It authenticates by calling the BFF, which initiates the OAuth 2 Authorization Code flow with PKCE server-side.
2. **The BFF stores tokens in a server-side session**, secured by an `HttpOnly`, `Secure`, `SameSite` cookie -- exactly the cookie-based model from the first section of this chapter.
3. **The BFF proxies API requests**, attaching the access token from its session store. The browser only sends the session cookie.
4. **Token refresh happens server-side.** The BFF uses the refresh token to obtain new access tokens without any browser involvement.

This pattern combines the security strengths of both approaches:

- **Cookie security** -- the session cookie is `HttpOnly` (immune to XSS theft) and `SameSite` (resistant to CSRF).
- **OAuth 2/OIDC benefits** -- the application still uses standard protocols, delegated identity, and fine-grained scopes.
- **No tokens in the browser** -- the largest attack surface is eliminated entirely.

From Angular's perspective, the BFF pattern simplifies the frontend code dramatically. The interceptor no longer manages tokens -- it just ensures credentials (cookies) are included:

```typescript
export const bffInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.url.startsWith('/api')) {
    const bffReq = req.clone({ withCredentials: true });
    return next(bffReq);
  }
  return next(req);
};
```

The route guard checks authentication status via a simple BFF endpoint:

```typescript
export const authGuard: CanActivateFn = () => {
  const http = inject(HttpClient);
  const router = inject(Router);

  return http.get<{ authenticated: boolean }>('/bff/session').pipe(
    map((session) => session.authenticated || router.createUrlTree(['/login'])),
    catchError(() => of(router.createUrlTree(['/login'])))
  );
};
```

The BFF itself is a thin server (Node.js, .NET, Python) that handles:

- Initiating the authorization code flow
- Exchanging the code for tokens
- Storing tokens in a server-side session
- Proxying API calls with the access token
- Refreshing tokens before they expire
- Revoking tokens on logout

This pattern is recommended by the IETF's "OAuth 2.0 for Browser-Based Applications" (draft-ietf-oauth-browser-based-apps), the OpenID Foundation, and major identity providers including Microsoft, Auth0, and Okta.

> **Trade-off:** The BFF adds operational complexity -- another service to deploy, monitor, and scale. For applications with lower security requirements, client-side PKCE with in-memory token storage and short-lived access tokens may be acceptable. For FinancialApp, the BFF is non-negotiable.

---

## Summary

Authentication and authorization are not features to bolt on -- they are architectural decisions that shape the entire application.

- **Cookie-based authentication** is simple and well-understood. Proper cookie attributes (`HttpOnly`, `Secure`, `SameSite`) and XSRF protection are essential. Angular provides built-in XSRF support via `withXsrfConfiguration()`.
- **OAuth 2** is an authorization framework that delegates access decisions to an authorization server. **OpenID Connect** adds an identity layer, providing user authentication via ID tokens.
- **JWTs** encode claims in a signed, self-contained format. They are the standard token format for both access tokens and ID tokens. Never store sensitive data in the payload.
- **Authorization Code + PKCE** is the only recommended OAuth 2 flow for public clients. The implicit flow and resource owner password grant are deprecated.
- **Client-side token storage is inherently risky** in browser environments. XSS, malicious dependencies, and browser extensions can all exfiltrate tokens.
- **The BFF pattern** is the current best practice for SPAs handling sensitive data. It moves token management server-side, secures the browser session with `HttpOnly` cookies, and eliminates the token-in-browser attack surface.

For FinancialApp, the BFF pattern provides the strongest security posture while keeping the Angular frontend clean and simple. The functional guards and interceptors from [Chapter 12](ch12-initialization-routes.md) remain the same -- only what they attach to requests changes (cookies instead of bearer tokens).

In the next chapter, we will explore how to test these security patterns, including mocking authentication state in unit tests and verifying guard behavior in integration tests.
