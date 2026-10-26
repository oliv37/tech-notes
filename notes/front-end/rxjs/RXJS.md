## Observable vs Promise

> **In short**: A Promise represents one future value, starts immediately, always delivers its result asynchronously, and has no built-in cancellation. An Observable is a stream of 0 to n values over time, lazy (nothing runs before `subscribe()`), synchronous or asynchronous depending on its source, cancellable (`unsubscribe()`), and composable with operators. Angular's `HttpClient`, router and forms expose Observables.

| | Promise | Observable |
|---|---|---|
| Values | exactly one (or an error) | 0 to n, then complete or error |
| Execution | eager: starts at creation | lazy: starts on `subscribe()`, once per subscriber (cold) |
| Timing | `then` / `await` callbacks always asynchronous (microtask), even when already resolved | synchronous or asynchronous depending on the source: `of()`, `BehaviorSubject` emit during `subscribe()`, `HttpClient`, `interval` later |
| Cancellation | not built in: the underlying operation can be aborted with an `AbortController` (e.g. `fetch`), the promise is then rejected | built in: `unsubscribe()` runs the teardown (cancels an HTTP request) |
| Composition | `then`, `async` / `await` | operators: `map`, `filter`, `switchMap`, `retry`... |
| Typical use | a single async result | events, streams, values changing over time |

```ts
const user = await firstValueFrom(this.http.get<User>("/api/users/1")); // Observable → Promise
const user$ = from(fetch("/api/users/1"));                              // Promise → Observable
```

```ts
of(1, 2).subscribe(v => console.log(v));         // 1, 2, then "after": emitted synchronously
console.log("after");

Promise.resolve(1).then(v => console.log(v));    // "after", then 1: always asynchronous
console.log("after");
```

- `firstValueFrom` / `lastValueFrom` (RxJS 7) replace the deprecated `toPromise()`.

## Hot vs cold observables

> **In short**: A cold observable creates its producer on each `subscribe()` (an HTTP call runs once per subscriber); a hot one shares an existing producer (events, `Subject`), and late subscribers miss past values. `share()` / `shareReplay()` turn a cold observable into a shared one.

The difference is where the producer (the source of values) lives:

- **Cold**: the producer is created on each `subscribe()`. Every subscriber gets its own independent execution, from the start. Examples: `HttpClient` calls, `of()`, `from()`, `interval()`.
- **Hot**: the producer exists independently of subscriptions and is shared. Subscribers receive values from the moment they subscribe and miss the previous ones. Examples: `fromEvent()`, `Subject`, router events.

```typescript
const users$ = this.http.get<User[]>("/api/users"); // cold

users$.subscribe(); // HTTP request #1
users$.subscribe(); // HTTP request #2
```

Classic Angular pitfall: using the same HTTP observable twice in a template with `| async` sends two requests. A cold observable can be made hot (shared) with operators:

- `share()`: one shared execution while there are subscribers; late subscribers miss past values.
- `shareReplay({ bufferSize: 1, refCount: true })`: also replays the last value to late subscribers, useful for caching an HTTP response. `refCount: true` unsubscribes from the source when there are no subscribers left, which matters for sources that never complete (without it, they keep running forever).

```typescript
const users$ = this.http
  .get<User[]>("/api/users")
  .pipe(shareReplay({ bufferSize: 1, refCount: true }));
```

## Observable lifecycle

> **In short**: An observable emits any number of values, then at most one `error` or `complete`, after which nothing is delivered. The end applies to one execution: resubscribing to a cold observable restarts it (`retry`, `repeat`). The teardown logic, and `finalize()` on the consumer side, run once whatever ends it.

An observable follows the contract `next* (error | complete)?`: any number of values, then at most one terminal notification. `error` and `complete` are final and mutually exclusive: once one is emitted, nothing else is delivered (RxJS ignores later calls).

The terminal state applies to one execution, not to the observable itself. Subscribing again to a cold observable starts a new execution: this is what `retry()` (after an error) and `repeat()` (after a complete) do. A `Subject`, on the other hand, stays terminated: `next()` is ignored and new subscribers immediately receive the `complete` / `error`.

The teardown logic is returned by the function passed to `new Observable()`. It runs exactly once, whatever ends the execution: `complete`, `error` or `unsubscribe()`.

```typescript
const tick$ = new Observable<number>((subscriber) => {
  let count = 0;
  const id = setInterval(() => subscriber.next(count++), 1000);

  return () => clearInterval(id); // teardown
});
```

On the consumer side, `finalize()` runs a callback in the same three cases, e.g. to hide a loader:

```typescript
this.loading = true;
this.http
  .get("/api/data")
  .pipe(finalize(() => (this.loading = false)))
  .subscribe();
```

## Subscribing and unsubscribing

> **In short**: `subscribe()` starts the observable and returns a `Subscription`; a stream that never completes (form changes, intervals, store selections) keeps running, and keeps its component in memory, until `unsubscribe()`. In Angular, the `async` pipe, `takeUntilDestroyed()` or `toSignal()` unsubscribe automatically when the component is destroyed.

```ts
@Component({
  template: `@if (user$ | async; as user) { <p>{{ user.name }}</p> }`, // async pipe: subscribes and unsubscribes
})
export class ProfileComponent {
  user$ = this.userService.getUser(1);

  constructor() {
    this.searchControl.valueChanges
      .pipe(takeUntilDestroyed()) // unsubscribes when the component is destroyed
      .subscribe(term => this.search(term));
  }
}
```

- Streams that complete by themselves don't need an unsubscription: `HttpClient` calls (one response then complete), `of()`, `take(1)`, `first()`.
- `takeUntilDestroyed()` must be called in an injection context (constructor, field initializer), or given a `DestroyRef`.
- Before Angular 16: `takeUntil(this.destroy$)` with a `Subject` completed in `ngOnDestroy`.
- Nested `subscribe()` calls (a `subscribe` inside a `subscribe`) are a code smell: use a mapping operator (see `switchMap` below).

## Subjects

> **In short**: A `Subject` is both an Observable and an observer: values pushed with `next()` are multicast to all its subscribers (hot). `BehaviorSubject` holds a current value, given immediately to each new subscriber; `ReplaySubject` replays the last n values; `AsyncSubject` emits only the last value on completion.

| Subject | New subscriber receives | Typical use |
|---|---|---|
| `Subject` | only the values emitted after subscribing | events (clicks, notifications) |
| `BehaviorSubject(initial)` | the current value, then the next ones; `getValue()` reads it | state: current user, cart |
| `ReplaySubject(n)` | the last n values, then the next ones | cache of recent values |
| `AsyncSubject` | the last value, only on `complete()` | rare: a single final result |

```ts
@Injectable({ providedIn: "root" })
export class CartService {
  private readonly items = new BehaviorSubject<CartItem[]>([]);
  readonly items$ = this.items.asObservable(); // read-only for the consumers

  add(item: CartItem) {
    this.items.next([...this.items.getValue(), item]);
  }
}
```

- Expose `asObservable()`, not the subject: only the service can push values.
- For component and service state, signals (`signal()`, see [Angular signals](../angular/ANGULAR_SIGNAL.md)) now replace the `BehaviorSubject` pattern; subjects remain useful for event streams.

## Common operators

> **In short**: Operators are functions chained in `pipe()`, each returning a new Observable: `map` and `filter` transform values, `tap` runs side effects, `debounceTime` and `distinctUntilChanged` limit the emissions of user input, `take` / `first` complete the stream, `startWith` adds an initial value.

```ts
this.searchControl.valueChanges.pipe(
  map(term => term.trim()),
  filter(term => term.length >= 2),
  debounceTime(300),          // waits 300 ms without a new value
  distinctUntilChanged(),     // ignores a value equal to the previous one
  tap(term => console.log("search", term)),
  switchMap(term => this.api.search(term)), // see below
);
```

| Operator | Role |
|---|---|
| `map`, `filter` | transform / filter each value |
| `tap` | side effect (log, loader), without changing the values |
| `debounceTime(ms)` | emits a value only after `ms` of silence (typing) |
| `throttleTime(ms)` | at most one value per `ms` (scroll, resize) |
| `distinctUntilChanged()` | skips consecutive duplicates |
| `take(n)`, `first()`, `takeUntil(notifier$)` | complete after n values / the first / when another stream emits |
| `startWith(value)` | emits a value first |
| `scan(fn, seed)` | accumulates, emitting each intermediate result (like `reduce`, but continuous) |
| `shareReplay(1)` | shares one execution and replays the last value (see hot vs cold above) |
| `finalize(fn)` | runs on complete, error or unsubscribe |

## switchMap vs mergeMap vs concatMap vs exhaustMap

> **In short**: The four operators map each value to an inner Observable (typically an HTTP call) and flatten the results; they differ when a new value arrives while the previous inner Observable is still running. `switchMap` cancels the previous one, `mergeMap` runs them in parallel, `concatMap` queues them in order, `exhaustMap` ignores new values until the current one completes.

| Operator | New value while the previous call runs | Typical use |
|---|---|---|
| `switchMap` | **cancels** the previous call, keeps the latest | search as you type, route params: only the latest result matters |
| `mergeMap` | runs **in parallel**, results in completion order | independent calls (delete several items), no order needed |
| `concatMap` | **queues**: waits for the previous one, keeps the order | ordered writes (save steps one after the other) |
| `exhaustMap` | **ignores** it until the current one completes | submit / login button: avoids double submission |

```ts
// search: a new term cancels the request of the previous one
results$ = this.term$.pipe(switchMap(term => this.api.search(term)));

// save button: clicks during the save are ignored
this.saveClicks$.pipe(exhaustMap(() => this.api.save(this.form.value))).subscribe();

// route parameter: loads the user of the current id
user$ = this.route.paramMap.pipe(switchMap(params => this.api.getUser(params.get("id")!)));
```

- They replace nested `subscribe()` calls, which can't be cancelled and mix up the results.
- `switchMap` on a write is dangerous: cancelling the request on the client doesn't cancel it on the server, and the result is lost; use `concatMap` or `exhaustMap`.
- `mergeMap(fn, 3)` limits the number of concurrent calls.

## Combining observables

> **In short**: `forkJoin` waits for several Observables to complete and emits their last values once (parallel HTTP calls); `combineLatest` emits the latest value of each source whenever any of them changes (filters, state); `merge` interleaves the values of several streams; `withLatestFrom` adds the latest value of another stream to each emission.

```ts
// parallel calls, one result when all are done (like Promise.all)
forkJoin({ user: this.api.getUser(id), orders: this.api.getOrders(id) })
  .subscribe(({ user, orders }) => ...);

// recomputed when any filter changes
combineLatest([this.status$, this.page$]).pipe(
  switchMap(([status, page]) => this.api.getOrders(status, page)),
);

// several event sources in one stream
merge(this.refreshClicks$, interval(30_000)).pipe(switchMap(() => this.api.load()));

// the click triggers, the form value is just read
this.submit$.pipe(withLatestFrom(this.form.valueChanges));
```

| Operator | Emits | Pitfall |
|---|---|---|
| `forkJoin` | once, when all complete | never emits if one source doesn't complete; fails if one fails |
| `combineLatest` | on each change, once every source has emitted | waits for a first value from each source (`startWith` helps) |
| `merge` | each value of any source | no combination, just interleaving |
| `zip` | pairs the n-th values of each source | rarely what's wanted with independent streams |
| `withLatestFrom` | on each value of the main source | ignores the main source until the other has emitted |

## catchError placement

> **In short**: An error terminates the stream it reaches, and `catchError` replaces that stream rather than repairing it. On a long-lived stream (form changes, clicks), put `catchError` on the inner observable of `switchMap`, otherwise the first error silently kills the whole stream.

An error terminates the observable it reaches. `catchError` doesn't "repair" the stream: it replaces it with another observable. Its placement therefore decides which stream dies.

Wrong: `catchError` on the outer stream.

```typescript
this.results$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  switchMap((term) => this.http.get<Result[]>(`/api/search?q=${term}`)),
  catchError(() => of([])), // ❌
);
```

When an HTTP request fails, the error propagates to the outer stream, `catchError` replaces it with `of([])`, which emits `[]` and completes. The whole stream is over: the search stops reacting to `valueChanges`, silently (no error in the console).

Right: `catchError` on the inner observable.

```typescript
this.results$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  switchMap((term) =>
    this.http.get<Result[]>(`/api/search?q=${term}`).pipe(
      catchError(() => of([])), // ✅
    ),
  ),
);
```

Only the inner HTTP observable completes, the outer stream stays alive and the next keystroke triggers a new request.

Rule: for a long-lived stream (form changes, clicks, router events, `Subject`), handle errors inside the inner observable of `switchMap` / `mergeMap` / `concatMap` / `exhaustMap`.

## Retrying: retry with delay

> **In short**: `retry()` resubscribes to the source after an error, which re-executes a cold Observable (a new HTTP request). With `retry({ count: 3, delay: 1000 })`, the retries are limited and spaced; the delay can be a function for an exponential backoff, and only transient errors should be retried.

```ts
this.http.get<Order[]>("/api/orders").pipe(
  retry({
    count: 3,
    delay: (error, retryCount) =>
      error.status >= 500 ? timer(2 ** retryCount * 500) : throwError(() => error), // 1 s, 2 s, 4 s; no retry on 4xx
  }),
  catchError(() => of([])), // after the last retry
);
```

- Place `retry` before `catchError`: `catchError` handles what's left after the retries.
- `retry` without arguments retries forever.
- Only idempotent requests (`GET`, `PUT`): retrying a `POST` can create a duplicate.
- Usually put in an HTTP interceptor to apply it to every request.
