## Angular Interceptor

An interceptor is a function that sits between `HttpClient` and the backend. Every HTTP request goes through it, so it is the place for cross-cutting concerns: adding an auth token, handling errors globally, logging, showing a loader.

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (!token || !req.url.startsWith("/api")) {
    return next(req);
  }

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` },
  });

  return next(authReq);
};
```

- `HttpRequest` is immutable: it must be copied with `req.clone(...)` to change it.
- `next(req)` passes the request to the next interceptor (or to the backend for the last one) and returns an `Observable` of the response, which can be piped (`catchError`, `retry`, `finalize`...).
- The URL check prevents sending the token to third-party domains.
- `inject()` works because functional interceptors run in an injection context.

Interceptors are registered when providing `HttpClient`:

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor])),
  ],
};
```

The order matters: requests go through interceptors in the array order (`auth` → `error` → backend), and responses come back in reverse order (backend → `error` → `auth`). Here `errorInterceptor` sees the request with the token already added, and handles errors before `authInterceptor` gets the response.

## Storing the auth token in Angular

Where the token lives defines what an attacker can do with it:

- **`localStorage` / `sessionStorage`**: simple, but readable by any JavaScript running on the page. A single XSS flaw (or a compromised npm dependency) can steal the token and use it from anywhere.
- **In memory** (a service field): not persisted, so lost on page refresh, and still reachable by an XSS while the page is open. Often combined with a refresh token stored in an `HttpOnly` cookie to get a new access token after a refresh.
- **`HttpOnly` cookie** set by the backend: JavaScript can't read it, so an XSS can't steal it (it can still send requests from the page while it runs). The browser sends it automatically, so no interceptor is needed to add it, but this opens the door to CSRF.

Protecting a cookie against CSRF:

- `SameSite=Lax` or `Strict` (plus `Secure`): the browser doesn't send the cookie on requests initiated by another site.
- A CSRF token: Angular's `HttpClient` supports it out of the box. It reads the `XSRF-TOKEN` cookie set by the backend and sends it back in the `X-XSRF-TOKEN` header (only on mutating requests to relative URLs). Spring Security supports this convention with `CookieCsrfTokenRepository.withHttpOnlyFalse()` (the CSRF cookie must be readable by JavaScript, unlike the auth cookie).

The most secure option is the **BFF** (Backend For Frontend) pattern: a backend handles the OAuth flow and keeps the tokens server-side, the browser only gets an `HttpOnly` session cookie. No token is ever exposed to JavaScript.

How it works in practice:

- The BFF is a confidential OAuth client: it performs the login flow, exchanges the `code` for tokens server-side, and refreshes them when they expire.
- The cookie only holds an opaque session id (`HttpOnly; Secure; SameSite`), the tokens stay in the server-side session.
- Angular calls the BFF without any header; the BFF adds `Authorization: Bearer <token>` and forwards the request to the APIs (in Spring: Spring Cloud Gateway + `TokenRelay`).
- To run several BFF instances, store the sessions in a shared store (Redis with Spring Session), so any instance can serve any request and logout simply deletes the session.

A stateless BFF (encrypted tokens directly in the cookie) also scales without Redis, but comes with trade-offs: cookie size (often > 4 KB), concurrent refreshes that can revoke the session when refresh token rotation is enabled, and no server-side revocation. The stateful BFF with a shared session store remains the recommended approach.

## Forcing the caller to handle errors

With an `Observable`, errors are untyped and optional to handle: nothing forces the caller to pass an `error` callback or a `catchError`, and TypeScript can't say which errors can occur. To make error handling mandatory, the service can catch the errors itself and return a **result type**, a discriminated union that the caller must check before accessing the value:

```typescript
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

type UserError = "NOT_FOUND" | "FORBIDDEN" | "UNKNOWN";

getUser(id: string): Observable<Result<User, UserError>> {
  return this.http.get<User>(`/api/users/${id}`).pipe(
    map((value) => ({ ok: true as const, value })),
    catchError((err: HttpErrorResponse) =>
      of({ ok: false as const, error: toUserError(err.status) }),
    ),
  );
}
```

On the caller side, `result.value` doesn't compile until `result.ok` has been checked (TypeScript narrows the type):

```typescript
this.userService.getUser(id).subscribe((result) => {
  if (!result.ok) {
    this.showError(result.error); // typed: "NOT_FOUND" | "FORBIDDEN" | "UNKNOWN"
    return;
  }
  this.user = result.value; // only accessible here
});
```

To force handling **every** error case, use a `switch` with an exhaustive check: adding a new value to `UserError` then breaks compilation wherever a case is missing.

```typescript
switch (result.error) {
  case "NOT_FOUND": ...; break;
  case "FORBIDDEN": ...; break;
  case "UNKNOWN": ...; break;
  default: {
    const _exhaustive: never = result.error; // compile error if a case is missing
  }
}
```

Benefits: the error becomes part of the method's contract, and the observable never errors, so it can't kill a long-lived stream (see `catchError` placement in RxJS). Libraries like `neverthrow` provide a ready-made `Result` type with helpers (`map`, `match`...).
