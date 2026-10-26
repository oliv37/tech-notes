## Angular lazy loading

> **In short**: Lazy loading splits the app into chunks downloaded only when needed: `loadComponent` / `loadChildren` with a dynamic `import()` for routes, `@defer` blocks inside a template. Faster first load; preloading and `canMatch` refine it.

Lazy loading splits the app into separate JS chunks, downloaded only when needed instead of in the initial bundle. Faster first load, especially for large apps with many features.

**Lazy routes**: `loadComponent` (one component) or `loadChildren` (a set of routes) take a dynamic `import()`, which the build turns into a separate chunk, downloaded on the first navigation to the route.

```typescript
export const routes: Routes = [
  { path: "", component: HomeComponent }, // eager: in the main bundle
  {
    path: "admin",
    loadChildren: () =>
      import("./admin/admin.routes").then((m) => m.ADMIN_ROUTES),
  },
  {
    path: "profile",
    loadComponent: () =>
      import("./profile/profile.component").then((m) => m.ProfileComponent),
  },
];
```

- **Preloading**: `provideRouter(routes, withPreloading(PreloadAllModules))` downloads lazy chunks in the background once the app is loaded, so navigation is instant while the first load stays fast.
- **Guards**: `canMatch` prevents downloading the chunk at all when the user isn't allowed (e.g. not an admin), unlike `canActivate` which runs after.
- **Pitfall**: a lazy component imported directly elsewhere (in the eager code) ends up in the main bundle, and lazy loading no longer has any effect.

**`@defer` blocks** (Angular 17, stable since 18): lazy loading inside a template, for heavy components that aren't needed right away (charts, comments, below the fold).

```html
@defer (on viewport) {
<app-comments [postId]="postId" />
} @placeholder {
<p>Comments</p>
} @loading (minimum 300ms) {
<app-spinner />
}
```

- Triggers: `on viewport` (visible on screen), `on idle` (default, browser idle), `on interaction`, `on hover`, `on timer(2s)`, or `when condition`.
- Only standalone components used inside the block are deferred, and only if they aren't referenced elsewhere in the eager code.

## Angular Interceptor

> **In short**: A functional interceptor (`HttpInterceptorFn`) sees every `HttpClient` request and response: the place for cross-cutting concerns like adding the auth token or handling errors globally. Requests and responses are immutable (`clone()`), errors are handled with `catchError` on `next(req)`, and interceptors run in array order for requests, reverse order for responses.

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

**Handling errors**: pipe `catchError` on `next(req)`. An HTTP error arrives as an `HttpErrorResponse` (`status`, `error` = the response body, `status === 0` for a network error or CORS failure).

```typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router); // here, not inside catchError: no injection context there
  const notification = inject(NotificationService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        router.navigate(["/login"]);
      } else if (error.status === 0 || error.status >= 500) {
        notification.error("Server unavailable, try again later");
      }
      return throwError(() => error); // rethrow so the caller still sees the error
    }),
  );
};
```

- Rethrowing with `throwError` lets the component still react (stop a spinner, show a field error). Returning `EMPTY` or `of(...)` instead swallows the error: the caller never knows the request failed.
- `retry({ count: 2, delay: 1000 })` before `catchError` retries transient failures; only for idempotent requests (`GET`), never a `POST` that could create a resource twice.
- To skip the global handling for one request, pass a flag with `HttpContext` (`new HttpContextToken(() => false)`) and read it in the interceptor with `req.context.get(...)`.

**Modifying the response**: `next(req)` doesn't emit only the response but a stream of `HttpEvent` (`Sent`, upload/download progress, then the `HttpResponse`). Filter on `HttpResponse` and copy it with `event.clone(...)`, since it is immutable like the request. E.g. a backend that wraps every body in `{ data: ... }`:

```typescript
const isWrapped = (body: unknown): body is { data: unknown } =>
  typeof body === "object" && body !== null && "data" in body;

export const unwrapInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    map((event) =>
      event instanceof HttpResponse && isWrapped(event.body)
        ? event.clone({ body: event.body.data })
        : event,
    ),
  );
```

- The other events must be passed through unchanged, or progress tracking breaks.
- `http.get<User>()` doesn't check anything at runtime: the generic only types what the interceptor chain produces, so the component gets `User`, not `{ data: User }`.
- Same pattern for converting date strings to `Date`, or reading a response header (`event.headers.get("X-Total-Count")`).

**Refreshing the token on `401`**: catch the `401`, refresh the token, then replay the request with `switchMap`. Several requests can fail at the same time, so the service shares a single in-flight refresh:

```typescript
@Injectable({ providedIn: "root" })
export class AuthService {
  private readonly http = inject(HttpClient);
  private refresh$: Observable<void> | null = null;

  refresh(): Observable<void> {
    this.refresh$ ??= this.http.post<void>("/auth/refresh", null).pipe(
      finalize(() => (this.refresh$ = null)), // the next 401 starts a new refresh
      shareReplay(1), // one request for concurrent callers, not cancelled if they all unsubscribe
    );
    return this.refresh$;
  }
}

export const refreshInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status !== 401 || req.url.endsWith("/auth/refresh")) {
        return throwError(() => error);
      }
      return auth.refresh().pipe(
        catchError((refreshError) => {
          router.navigate(["/login"]); // refresh token expired or revoked
          return throwError(() => refreshError);
        }),
        switchMap(() => next(req)), // replayed once, with the new token
      );
    }),
  );
};
```

- The `/auth/refresh` exclusion avoids an infinite loop: the refresh request itself goes through the interceptor.
- With tokens in `HttpOnly` cookies (see below), the replayed request carries the new cookie automatically. With a token in memory, the replay must add it: `next(req.clone({ setHeaders: { Authorization: ... } }))`.
- Without the shared `refresh$`, 5 parallel `401`s start 5 refreshes; with refresh token rotation, the first one invalidates the refresh token and the 4 others fail.
- `shareReplay(1)` rather than `share()`: the replay is never used (`finalize` resets `refresh$`, so later calls create a new refresh), but `share()` unsubscribes from the source when all subscribers leave, which cancels the `POST`. If the only waiting request is cancelled (`switchMap`, component destroyed), the server may have already rotated the refresh token while the response with the new cookies is dropped: the user is logged out. `shareReplay` keeps the source running (`refCount: false` by default); `share({ resetOnRefCountZero: false })` does the same.

## Angular error handling and ErrorHandler

> **In short**: Errors are handled locally (`catchError`), globally for HTTP (an error interceptor), and as a last resort by `ErrorHandler`, which receives every uncaught error. In templates, `@defer` has an `@error` block, resources expose `error()` as a signal, and `@boundary` / `@error` catches rendering errors.

Errors are handled at three levels, from the most specific to the most global:

1. **Locally**: `catchError` in a service, or the `error` callback of `subscribe`, when the component can react (show a message, fallback value).
2. **HTTP, globally**: an error interceptor (see above) for cross-cutting cases: `401` → redirect to login, `5xx` → generic toast, retry.
3. **Everything else**: `ErrorHandler`, the last resort for uncaught errors.

**`ErrorHandler`** receives every uncaught error: exceptions in components, templates, lifecycle hooks, event handlers, and errors from a `subscribe` without `error` callback. The default implementation only logs to the console; a custom one can send errors to a monitoring tool (Sentry) or show a notification.

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly notification = inject(NotificationService);

  handleError(error: unknown): void {
    console.error(error);
    this.notification.error("An unexpected error occurred");
    // e.g. Sentry.captureException(error)
  }
}

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler },
    provideBrowserGlobalErrorListeners(), // Angular 20: also catches window errors and unhandled promise rejections
  ],
};
```

- `handleError` must never throw itself (the error would be lost, or loop).
- It doesn't catch HTTP errors already handled by a `catchError` or an interceptor: only what's left unhandled.

**`@error` in a `@defer` block**: shown when the deferred chunk fails to load (network error, new deployment that removed the old chunk).

```html
@defer (on viewport) {
<app-comments />
} @error {
<p>Comments could not be loaded.</p>
}
```

Signal-based loading (`resource()` / `httpResource()`, introduced as experimental in Angular 19) exposes the error as a signal, displayed directly in the template without `catchError`:

```typescript
user = httpResource<User>(() => `/api/users/${this.userId()}`);
```

```html
@if (user.error()) {
<p>Could not load the user.</p>
} @else if (user.hasValue()) {
<p>{{ user.value().name }}</p>
}
```

**`@boundary` / `@error`** (Angular 22.2, developer preview): an error boundary in the template, like React's. If anything inside throws while rendering (a child component, a resource whose `value()` throws because loading failed), Angular replaces that section with the `@error` block instead of breaking the page.

```html
@boundary {
<p>{{ user.value().name }}</p>
<!-- no manual error() check needed -->
<app-chart />
} @error (let err; retry = $reset; when isNetworkError(err)) {
<p>Network issue: {{ err.message }}</p>
<button (click)="retry()">Retry</button>
} @error {
<p>Something went wrong.</p>
}
```

- `$error` holds the caught error (renamed with `let`), `$reset()` re-renders the content, `when` picks a fallback per error type.
- The nearest enclosing boundary catches the error, so nested boundaries isolate a single widget.
- Not caught: event handlers (`(click)`, they go to `ErrorHandler`), content projected with `<ng-content>`, errors inside `@error` itself (passed to the outer boundary). Caught errors are still reported to `ErrorHandler`.

## Storing the auth token in Angular

> **In short**: `localStorage` is readable by any XSS; memory is lost on refresh; an `HttpOnly` cookie can't be read by JavaScript but needs CSRF protection (`SameSite`, CSRF token). The most secure option is a BFF: tokens stay on the server, the browser only holds an `HttpOnly` session cookie.

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

## BFF vs access token in an HttpOnly cookie

> **In short**: Both keep the token away from JavaScript, so both resist token theft by XSS, and both need CSRF protection. The BFF goes further: the browser only holds an opaque session id, so the refresh token never leaves the server, logout is immediate, and a stolen cookie only works through the BFF. The JWT cookie is stateless (no shared store for several instances), at the cost of revocation, cookie size and refresh handled by hand.

In both cases a Spring backend performs the OAuth login and sets the cookie; the difference is what the cookie contains.

|                                              | `localStorage`   | In memory           | JWT in `HttpOnly` cookie                                   | BFF (session id in `HttpOnly` cookie) |
| -------------------------------------------- | ---------------- | ------------------- | ---------------------------------------------------------- | ------------------------------------- |
| Token readable by an XSS                     | yes              | yes                 | no                                                         | no                                    |
| XSS can send requests while the page is open | yes              | yes                 | yes                                                        | yes                                   |
| CSRF to handle                               | no               | no                  | yes                                                        | yes                                   |
| Survives a page refresh                      | yes              | no                  | yes                                                        | yes                                   |
| Refresh token location                       | browser          | browser (or cookie) | browser cookie                                             | server only                           |
| A stolen cookie gives                        | —                | —                   | a JWT usable against any API that accepts it, until expiry | a session usable only through the BFF |
| Logout / revocation                          | expiry           | expiry              | expiry of the access token                                 | immediate (delete the session)        |
| Size limit                                   | none             | none                | ~4 KB per cookie                                           | none (short id)                       |
| Several instances                            | nothing to share | nothing to share    | nothing to share (stateless)                               | shared session store (Redis)          |
| APIs receive                                 | `Bearer`         | `Bearer`            | the cookie (custom `BearerTokenResolver`), same site only  | standard `Bearer`, added by the BFF   |

Why the BFF is considered better, although the cookie is `HttpOnly` in both cases:

- **The refresh token stays on the server.** With the JWT cookie, the refresh token also lives in a browser cookie: stolen, it gives new access tokens for days. With the BFF, the browser has nothing long-lived beyond a session the server can kill.
- **`HttpOnly` isn't the only way to steal a cookie.** A browser extension with the `cookies` permission, or malware reading the browser's cookie store, can read `HttpOnly` cookies. A stolen JWT works directly against every API that trusts the token, from anywhere; a stolen session id only works through the BFF, which can revoke it.
- **Immediate logout.** Deleting the session cuts access at once; a JWT stays valid until it expires, whatever logout does with the cookies.
- **Downstream APIs stay standard.** They keep validating a `Bearer` token, can live on other domains, and don't need CSRF protection; with the JWT cookie, every API must read the cookie and protect itself against CSRF.

When the JWT cookie is a reasonable choice: a single Spring app that is both the OAuth client and the API, with a short access token (5-15 min) and the refresh cookie restricted to `Path=/auth/refresh`. The security gap with the BFF is then small, and it avoids running Redis. The BFF wins as soon as there are several APIs, long sessions, or a need for immediate revocation.

**Where the refresh token lives** with a JWT cookie: usually in a second `HttpOnly` cookie, set by the backend at the same time, restricted to the refresh endpoint:

```text
Set-Cookie: access_token=eyJ...; HttpOnly; Secure; SameSite=Lax; Path=/api; Max-Age=600
Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict; Path=/auth/refresh; Max-Age=604800
```

- `Path=/auth/refresh`: the refresh token isn't sent with every API call, only to the refresh endpoint. `SameSite=Strict` is fine there, since it's only called by `fetch` from the application's own pages.
- On `401`, Angular calls `POST /auth/refresh` (see the refresh interceptor above); any instance reads the cookie, calls the authorization server's token endpoint with the client secret, and sets both cookies again.
- Variants: encrypt the refresh token in the cookie (a thief can't use it directly at the authorization server, only replay it through `/auth/refresh`); keep it server-side behind a key (no longer stateless: almost a BFF); or no refresh token at all, redirecting to the authorization server, which issues a new `code` silently while its own SSO session is valid (full page reload, the SPA state is lost).

Pitfalls of the JWT cookie:

- Stateless APIs often use `csrf.disable()`, which is safe with a `Bearer` header; with authentication in a cookie, CSRF protection must stay enabled.
- Concurrent `401`s from several requests or tabs trigger several refreshes; with refresh token rotation, all but the first fail. The Angular side must share a single in-flight refresh (see above); between tabs the race remains, some authorization servers accept the previous refresh token for a short grace period.

## Forcing the caller to handle errors

> **In short**: Observable errors are untyped and optional to handle. Returning a `Result` type (a discriminated union `{ ok: true; value } | { ok: false; error }`) makes the error part of the contract: TypeScript forces the caller to check `ok` before using the value, and an exhaustive `switch` forces handling every error case.

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
