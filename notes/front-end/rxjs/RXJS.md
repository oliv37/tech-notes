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
