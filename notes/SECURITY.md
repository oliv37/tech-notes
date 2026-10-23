## XSS and CSRF

**XSS** (Cross-Site Scripting): an attacker manages to run their own JavaScript in the application's page, usually by injecting content (a comment, a username, a URL parameter) that ends up rendered as HTML. The script runs with the same rights as the application: it can read `localStorage`, call the APIs as the user, modify the page.

- Angular escapes interpolation (`{{ }}`) and sanitizes `[innerHTML]` by default.
- The risks come from bypassing it: `bypassSecurityTrustHtml()`, direct DOM access (`nativeElement.innerHTML`), or a compromised third-party script or npm dependency.
- A Content Security Policy (`Content-Security-Policy` header) limits which scripts can run, as a second line of defense.

**CSRF** (Cross-Site Request Forgery): a malicious site makes the user's browser send a request to the application (a hidden form, an image). The browser automatically attaches the application's cookies, so the request is authenticated without the user knowing. The attacker can't read the response, but the action (transfer, deletion) is performed.

- Only affects authentication sent automatically by the browser (cookies), not a token added in the `Authorization` header by JavaScript.
- `SameSite=Lax` (default in modern browsers) or `Strict` blocks most cases. `Lax` still sends the cookie on top-level `GET` navigations, so `GET` requests must never modify data.
- `SameSite` works at site level, not origin: `evil.example.com` is the same site as `app.example.com`. A CSRF token covers this case.

The two are linked: an XSS bypasses any CSRF protection, since the malicious script runs from the application itself and can read the CSRF token.
