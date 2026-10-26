## Authentication vs authorization

> **In short**: Authentication checks **who** the caller is (password, token, certificate); authorization checks **what** they're allowed to do (roles, permissions, ownership of the resource). Failing the first gives `401`, failing the second `403`.

| | Authentication | Authorization |
|---|---|---|
| Question | who are you? | are you allowed to do this? |
| Based on | credentials: password, token, certificate, second factor | roles, permissions, scopes, ownership |
| HTTP failure | `401 Unauthorized` | `403 Forbidden` |
| Example | login with email + password | only the owner or an admin can cancel an order |

- Authorization must be checked on the server for every request, including **ownership**: `GET /orders/42` by a logged-in user must check that order 42 is theirs (broken access control, the #1 risk of the OWASP Top 10, see below).
- Hiding a button in the front-end isn't authorization: the API is callable directly.
- **MFA** (multi-factor authentication): something you know (password) + something you have (TOTP app, security key) or are; passkeys (WebAuthn) replace the password with a key pair stored on the device.

## Storing passwords

> **In short**: Passwords are **hashed** with a slow, salted algorithm designed for passwords (Argon2, bcrypt, scrypt, PBKDF2), never stored in clear, encrypted, or hashed with a fast hash like SHA-256. The salt makes identical passwords produce different hashes, the slowness makes brute force expensive.

- **Hashing, not encryption**: encryption can be reversed with the key; a hash can't. To check a login, the password typed is hashed again and compared.
- **Salt**: a random value per user, stored with the hash. Two users with the same password get different hashes, and precomputed tables (rainbow tables) are useless.
- **Slow on purpose**: SHA-256 computes billions of hashes per second on a GPU; bcrypt / Argon2 have a configurable cost (bcrypt cost 10 to 12, Argon2 memory-hard), slow enough for an attacker, fast enough for a login.
- In Spring: `PasswordEncoderFactories.createDelegatingPasswordEncoder()` stores `{bcrypt}$2a$10$...`, the algorithm can evolve without breaking the existing hashes (see [Spring Security](../back-end/spring/SPRING_SECURITY.md)).
- Also: rate limiting and lockout on the login, and a generic message ("invalid credentials") that doesn't reveal whether the account exists.

## SQL injection and other injections

> **In short**: An injection happens when user input is concatenated into a command (SQL, shell, LDAP...) and interpreted as code. The fix is to never build commands by concatenation: parameterized queries (`PreparedStatement`, JPA parameters) send the values separately from the SQL, so they can't change it.

```java
// ❌ vulnerable: name = "x' OR '1'='1" returns every user
String sql = "SELECT * FROM users WHERE name = '" + name + "'";

// ✅ parameterized: the value is never interpreted as SQL
PreparedStatement ps = connection.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, name);

// ✅ JPA / Spring Data: parameters are bound too
@Query("select u from User u where u.name = :name")
```

- JPA and Spring Data are safe as long as values are passed as parameters; a JPQL or native query built by concatenation is vulnerable just the same.
- Identifiers can't be parameters (a column name for sorting, a table name): validate them against a whitelist.
- Other injections, same principle: OS command (`Runtime.exec` with user input), LDAP, NoSQL queries, template injection, log injection (newlines in logged values).
- Defense in depth: a database user with minimal rights, input validation.

## XSS and CSRF

> **In short**: XSS: the attacker's JavaScript runs in the page, with all its rights (Angular escapes and sanitizes by default; CSP adds a second line of defense). CSRF: another site makes the browser send an authenticated request with the cookies (blocked by `SameSite` and CSRF tokens; not an issue with an `Authorization` header). An XSS bypasses any CSRF protection.

**XSS** (Cross-Site Scripting): an attacker manages to run their own JavaScript in the application's page, usually by injecting content (a comment, a username, a URL parameter) that ends up rendered as HTML. The script runs with the same rights as the application: it can read `localStorage`, call the APIs as the user, modify the page.

- Angular escapes interpolation (`{{ }}`) and sanitizes `[innerHTML]` by default.
- The risks come from bypassing it: `bypassSecurityTrustHtml()`, direct DOM access (`nativeElement.innerHTML`), or a compromised third-party script or npm dependency.
- A Content Security Policy (`Content-Security-Policy` header) limits which scripts can run, as a second line of defense.

**CSRF** (Cross-Site Request Forgery): a malicious site makes the user's browser send a request to the application (a hidden form, an image). The browser automatically attaches the application's cookies, so the request is authenticated without the user knowing. The attacker can't read the response, but the action (transfer, deletion) is performed.

- Only affects authentication sent automatically by the browser (cookies), not a token added in the `Authorization` header by JavaScript.
- `SameSite=Lax` (default in modern browsers) or `Strict` blocks most cases. `Lax` still sends the cookie on top-level `GET` navigations, so `GET` requests must never modify data.
- `SameSite` works at site level, not origin: `evil.example.com` is the same site as `app.example.com`. A CSRF token covers this case.

The two are linked: an XSS bypasses any CSRF protection, since the malicious script runs from the application itself and can read the CSRF token.

## Session vs token authentication

> **In short**: With a **session**, the server stores the login state and the browser only holds a session id in a cookie: easy to revoke, but needs shared storage with several instances. With a **token** (JWT), the state is in the token itself, signed by the server and sent in the `Authorization` header: stateless and easy to use between services, but hard to revoke before it expires.

| | Session (cookie) | Token (JWT in `Authorization`) |
|---|---|---|
| State | on the server (memory, Redis) | in the token, signed |
| Sent by | the browser automatically (cookie) | the client code (`Authorization: Bearer`) |
| Revocation | immediate: delete the session | hard: valid until it expires (short lifetime + refresh token) |
| Several instances | shared session store (Spring Session + Redis) | nothing to share: each instance checks the signature |
| CSRF | to protect (`SameSite`, CSRF token) | not affected, since not sent automatically |
| XSS | `HttpOnly` cookie not readable | token in `localStorage` readable by any script |
| Typical use | server-rendered web app, BFF | APIs, mobile apps, service-to-service |

For a browser SPA, the BFF pattern combines both: tokens kept on the server, a session cookie for the browser (see [Angular](../front-end/angular/ANGULAR.md), storing the auth token).

## JWT

> **In short**: A JWT (JSON Web Token) is `header.payload.signature`, each part Base64URL-encoded: the payload holds claims (user id, roles, expiry), readable by anyone, and the signature proves the server issued it and nothing was modified. It's **signed, not encrypted**: no secret goes in it. The receiver checks the signature, the expiry, the issuer and the audience.

```text
eyJhbGciOiJSUzI1NiJ9 . eyJzdWIiOiI0MiIsInNjb3BlIjoib3JkZXJzIiwiZXhwIjoxNzYxNDc4MDAwfQ . kD3x...
     header                        payload                                              signature
{"alg": "RS256"}        {"sub": "42", "scope": "orders", "exp": 1761478000, "iss": "https://auth.example.com"}
```

| Claim | Meaning |
|---|---|
| `sub` | subject: the user id |
| `iss` | issuer: who issued the token |
| `aud` | audience: for which API it's meant |
| `exp`, `iat`, `nbf` | expiry, issued at, not before |
| `scope`, `roles` | permissions (custom claims) |

- **Signature**: `HS256` uses a shared secret (whoever can verify can also forge); `RS256` / `ES256` use a key pair: the authorization server signs with its private key, the APIs verify with the public key, published at a JWKS URL. Asymmetric is the norm between services.
- **Validation**: signature with an expected algorithm (reject `alg: none`, a classic attack), `exp`, `iss`, `aud`. Libraries and Spring's resource server do it; never decode a JWT and trust it without checking the signature.
- **Revocation**: a JWT stays valid until `exp`. Hence short-lived access tokens (5 to 15 minutes) and a refresh token to get new ones, revocable on the authorization server.
- Size: a JWT with many claims makes every request heavier; it's not a place for user data.

## OAuth 2 and OpenID Connect

> **In short**: OAuth 2 is a **delegated authorization** protocol: an application (client) gets an **access token** from an authorization server (Keycloak, Auth0, Google) to call an API on behalf of a user, without ever seeing their password. OpenID Connect adds **authentication** on top: an **ID token** (JWT) that tells the client who the user is. For web and mobile apps, the flow to use is the authorization code flow with PKCE.

| Role | Example |
|---|---|
| Resource owner | the user |
| Client | the Angular app, the BFF, a mobile app |
| Authorization server | Keycloak, Auth0, Okta, Spring Authorization Server |
| Resource server | the API, which validates the access token |

**Authorization code flow with PKCE**:

1. The client redirects the user to the authorization server (`/authorize?client_id=...&scope=openid orders&code_challenge=...`).
2. The user logs in **on the authorization server** (the client never sees the password) and consents.
3. The authorization server redirects back to the client with a short-lived **authorization code**.
4. The client exchanges the code for tokens (`/token`), sending the `code_verifier` matching the `code_challenge` (PKCE): an intercepted code is useless without it.
5. The client calls the API with `Authorization: Bearer <access token>`; the API validates it.

| Token | For | Content |
|---|---|---|
| Access token | the API (resource server) | permissions (`scope`), short-lived |
| ID token (OIDC) | the client | who the user is (`sub`, `email`, `name`); never sent to an API |
| Refresh token | the authorization server | gets new access tokens without logging in again; long-lived, revocable |

| Flow | Use |
|---|---|
| Authorization code + PKCE | any app with a user (SPA, BFF, mobile) |
| Client credentials | service to service, no user |
| Device code | devices without a browser (TV, CLI) |
| Implicit, password (ROPC) | deprecated: tokens exposed in the URL / password given to the client; removed in OAuth 2.1 (draft) |

- Scopes limit what a token allows (`orders:read`); the user consents to them.
- In Spring: `spring-boot-starter-oauth2-client` for the client side (login, BFF), `oauth2-resource-server` for the API (see [Spring Security](../back-end/spring/SPRING_SECURITY.md)).

## Security headers

> **In short**: HTTP response headers that tell the browser to enable protections: `Content-Security-Policy` (which scripts and resources may load, against XSS), `Strict-Transport-Security` (HTTPS only), `X-Content-Type-Options: nosniff`, `frame-ancestors` / `X-Frame-Options` (against clickjacking), `Referrer-Policy`. Spring Security adds most of them by default, except CSP.

| Header | Protects against |
|---|---|
| `Content-Security-Policy: default-src 'self'; script-src 'self'` | XSS: injected inline scripts and scripts from other domains don't run |
| `Strict-Transport-Security: max-age=31536000; includeSubDomains` | downgrade to HTTP, interception |
| `X-Content-Type-Options: nosniff` | the browser guessing a type (a text file run as a script) |
| `Content-Security-Policy: frame-ancestors 'none'` / `X-Frame-Options: DENY` | clickjacking: the page embedded in an invisible iframe |
| `Referrer-Policy: strict-origin-when-cross-origin` | leaking full URLs (tokens, ids) to other sites |
| `Permissions-Policy: camera=(), geolocation=()` | unwanted use of browser features |

- CSP is the most effective against XSS, and the hardest to set up: inline scripts and styles must be removed or allowed with nonces / hashes. Start with `Content-Security-Policy-Report-Only` to see what would be blocked.
- Angular supports CSP with nonces (`ngCspNonce`, `autoCsp` in recent versions).

## OWASP Top 10

> **In short**: The OWASP Top 10 lists the most critical web application risks. In the 2025 edition: broken access control is first (users reaching data or actions they shouldn't), then security misconfiguration and software supply chain failures (vulnerable or compromised dependencies), cryptographic failures, injection.

| # (2025) | Risk | Example / defense |
|---|---|---|
| A01 | Broken access control (now including SSRF) | `GET /orders/42` of another user; check ownership on the server, deny by default |
| A02 | Security misconfiguration | default credentials, debug endpoints or Actuator exposed, verbose errors |
| A03 | Software supply chain failures | vulnerable or malicious dependency (npm, Maven); dependency scanning, lock files, SBOM |
| A04 | Cryptographic failures | data in clear (HTTP, unencrypted backups), weak password hashing |
| A05 | Injection | SQL, command, XSS; parameterized queries, escaping |
| A06 | Insecure design | no rate limit on a costly operation, business flaws |
| A07 | Authentication failures | no brute-force protection, weak sessions, no MFA |
| A08 | Software or data integrity failures | unsigned updates, insecure deserialization |
| A09 | Security logging and alerting failures | attacks not logged or not detected |
| A10 | Mishandling of exceptional conditions | errors leave the application in an unsafe state: a check failing open (access granted when the permission service is down), stack traces in responses, partial updates without rollback; fail closed, generic error messages |

- 2025 changes from 2021: supply chain failures and mishandling of exceptional conditions are new, SSRF is merged into broken access control, misconfiguration moves to second place.
- **SSRF** (Server-Side Request Forgery): the server fetches a URL given by the user (`?image=http://169.254.169.254/...`) and reaches internal services; validate and whitelist the destinations.
