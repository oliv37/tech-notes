## What a signal is, and why Angular added them

> **In short**: A signal is a reactive value read with `count()` and written with `set` / `update`. It gives Angular fine-grained reactivity: Angular knows which templates depend on which values, so it refreshes only what changed, without Zone.js and its whole-tree change detection.

A **signal** is a reactive wrapper around a value: the code that reads it is tracked as a dependency, so Angular knows what to refresh when the value changes (see below). It is read by calling it like a function, and written with `set` / `update`:

```ts
count = signal(0);

count(); // read: 0
count.set(5); // write
count.update((c) => c + 1); // write from the previous value: 6
```

**Why**: before signals, Angular relied on **Zone.js** to know when to update the view. Zone.js patches every async API of the browser (events, `setTimeout`, HTTP, promises) and, after each of them, triggers a change detection of the **whole component tree**, comparing each template binding with its previous value (dirty checking). Angular doesn't know _what_ changed, only that _something might have_.

Signals give Angular **fine-grained reactivity**: a template that reads a signal is registered as its consumer, so when the signal changes, Angular knows exactly which components to refresh.

- **Performance**: only the components depending on the changed signal are checked, not the whole tree.
- **Zoneless**: Zone.js is no longer needed (`provideZonelessChangeDetection()`, stable in Angular 20.2, default for new projects since Angular 21): smaller bundle, simpler stack traces, no "change detection didn't run" surprises.
- **Simpler state management** than RxJS for synchronous state: no subscription, no `async` pipe, no `BehaviorSubject`, always a current value.
- **Explicit derived state** with `computed`, instead of values recomputed in getters at each change detection.

History: developer preview in Angular 16, `signal` / `computed` stable in Angular 17, then signal inputs, queries, `effect`, `linkedSignal` and the resources stabilized over Angular 19 to 21.

## How signals work: the dependency graph

> **In short**: Reading a signal during a reactive run (`computed`, `effect`, template) records it as a dependency. A change notifies only live consumers (effects, templates and the computeds they depend on); a `computed` is recomputed lazily, on read, only if a producer's version changed.

Signals form a graph of **producers** (signals, computed) and **consumers** (computed, effects, templates):

```text
firstName ─┐
           ├─> fullName (computed) ──> template of ProfileComponent
lastName  ─┘
```

1. **Automatic tracking**: when a consumer runs (a `computed` function, an `effect`, a template), every signal read during that run is recorded as a dependency. No dependency list to declare.
2. **Live and non-live consumers**: effects and templates are **live** consumers: their producers keep a reference to them and notify them. A `computed` is live only while a live consumer depends on it (here `fullName`, read by the template); otherwise it's **non-live**: its producers don't know it (no reference, so no memory leak), and it isn't notified at all.
3. **Push, live consumers only**: `set()` on a signal marks its live consumers as dirty, transitively (`fullName`, then the template, which schedules the component for refresh). Nothing is recomputed yet.
4. **Pull, on read**: each signal carries a version number, incremented on each change. When a `computed` is read, it checks whether it was marked dirty (live) or, for a non-live one, **polls the versions of its producers**: if none changed, it returns its cached value without re-running its function. The template is refreshed at the next change detection, only for the components marked dirty.

- **Lazy and memoized**: a `computed` runs only when read, and only if one of its dependencies changed (checked on read); otherwise its cached value is returned.
- **Glitch-free**: consumers never see an inconsistent intermediate state; changing `firstName` then `lastName` refreshes `fullName` once, with both new values.
- **Dynamic dependencies**: only the signals read during the last run are tracked: in `computed(() => showDetails() ? details() : summary())`, `details` is a dependency only when `showDetails()` is true.
- **Equality**: a new value equal to the old one (`Object.is` by default, or the `equal` option) changes neither the version nor anything downstream; a `computed` recomputed to an equal value doesn't propagate either.

## signal, computed, effect

> **In short**: `signal()` holds writable state, `computed()` derives read-only state (lazy, memoized), `effect()` runs side effects towards non-reactive APIs when the signals it reads change. Update signals immutably, and prefer `computed` / `linkedSignal` over effects to propagate state.

**`signal()`**: writable state.

```ts
@Component({
  selector: "app-cart",
  template: `
    <p>{{ items().length }} items, total: {{ total() }} €</p>
    <button (click)="add(product)">Add</button>
  `,
})
export class CartComponent {
  items = signal<CartItem[]>([]);
  total = computed(() =>
    this.items().reduce((sum, item) => sum + item.price, 0),
  );

  add(product: Product) {
    this.items.update((items) => [
      ...items,
      { name: product.name, price: product.price },
    ]); // new array
  }
}
```

- **Immutable updates**: `this.items().push(item)` mutates the array without calling `set`: the signal isn't notified, the view isn't refreshed. Always create a new value.
- `asReadonly()` exposes a read-only signal, e.g. from a service: `readonly items = this._items.asReadonly();`.

**`computed()`**: read-only state derived from other signals, recomputed when they change.

```ts
filter = signal("");
visibleProducts = computed(() =>
  this.products().filter((p) =>
    p.name.toLowerCase().includes(this.filter().toLowerCase()),
  ),
);
```

**`effect()`**: runs a side effect each time the signals it reads change, for non-reactive APIs: logging, `localStorage`, a third-party library, the DOM.

```ts
constructor() {
  effect(() => {
    localStorage.setItem('cart', JSON.stringify(this.items())); // re-runs when items changes
  });

  effect((onCleanup) => {
    const timer = setInterval(() => this.refresh(), this.period());
    onCleanup(() => clearInterval(timer)); // before the next run, and on destroy
  });
}
```

- Created in an **injection context** (constructor, field initializer), destroyed with the component or service.
- Runs asynchronously, during change detection: several `set()` in a row trigger one run.
- **Not for propagating state**: `effect(() => this.total.set(...))` should be a `computed`, or a `linkedSignal` (see below). Effects are the last resort, for the outside world.
- Signals read after an `await` aren't tracked: read them before.
- **`untracked()`**: reads a signal without making it a dependency.

```ts
effect(() => {
  const user = this.currentUser(); // dependency
  untracked(() => this.logger.log(user, this.page())); // page isn't a dependency
});
```

## Signal-based component APIs: input, model, output, queries

> **In short**: `input()`, `model()` (two-way binding) and the queries `viewChild()` / `contentChild()` are signals, so a `computed` can derive from them without `ngOnChanges` or `ngAfterViewInit`. `output()` replaces `@Output()` + `EventEmitter`, but is an event emitter, not a signal.

Component inputs and queries are signals too, so a `computed` can derive from them, without `ngOnChanges`.

```ts
@Component({
  selector: "app-product-card",
  template: `
    <h3>{{ product().name }}</h3>
    <p>{{ priceWithTax() }} €</p>
    <input
      type="number"
      [value]="quantity()"
      (input)="quantity.set(+$any($event.target).value)"
    />
    <button (click)="addToCart.emit(product())">Add</button>
  `,
})
export class ProductCardComponent {
  product = input.required<Product>(); // required input
  taxRate = input(0.2); // optional input with a default value
  quantity = model(1); // two-way binding: [(quantity)]="qty"
  addToCart = output<Product>(); // event (not a signal)

  priceWithTax = computed(() => this.product().price * (1 + this.taxRate()));

  nameInput = viewChild<ElementRef>("name"); // Signal<ElementRef | undefined>
  rows = viewChildren(RowComponent); // Signal<readonly RowComponent[]>
}
```

```html
<app-product-card
  [product]="selected()"
  [(quantity)]="qty"
  (addToCart)="onAdd($event)"
/>
```

| API                                        | Replaces                            | Type                                                 |
| ------------------------------------------ | ----------------------------------- | ---------------------------------------------------- |
| `input()`, `input.required()`              | `@Input()`                          | `InputSignal<T>`, read-only in the component         |
| `input(0, { transform: numberAttribute })` | `@Input({ transform })`             | converts the bound value                             |
| `model()`                                  | `@Input()` + `@Output() xChange`    | `ModelSignal<T>`, writable, propagates to the parent |
| `output()`                                 | `@Output()` + `EventEmitter`        | `OutputEmitterRef<T>` (`emit()`), not a signal       |
| `viewChild()`, `viewChildren()`            | `@ViewChild`, `@ViewChildren`       | signal, updated when the view changes                |
| `contentChild()`, `contentChildren()`      | `@ContentChild`, `@ContentChildren` | signal                                               |

No more `ngOnChanges` to react to an input, nor `ngAfterViewInit` to wait for a `@ViewChild`: a `computed` or an `effect` reading the signal does it.

## linkedSignal: writable state derived from another signal

> **In short**: `linkedSignal()` is writable like a `signal`, but reset whenever its source changes, like a `computed`. Typical use: a selection reset (or kept, using the previous value) when the list of options changes.

A `computed` can't be written; a `signal` doesn't follow another one. **`linkedSignal()`** is both: writable, and **reset** each time its source changes. Typical case: a selection that must be reset when the list changes.

```ts
shippingOptions = input.required<ShippingMethod[]>();

// shorthand: reset to the first option when the list changes
selectedOption = linkedSignal(() => this.shippingOptions()[0]);

select(option: ShippingMethod) {
  this.selectedOption.set(option); // writable, until the next change of shippingOptions
}
```

With the previous value, to keep the selection if it still exists in the new list:

```ts
selectedOption = linkedSignal<ShippingMethod[], ShippingMethod>({
  source: this.shippingOptions,
  computation: (newOptions, previous) =>
    newOptions.find((opt) => opt.id === previous?.value.id) ?? newOptions[0],
});
```

Several sources: in the shorthand form, every signal read is a source. In the object form, `source` is typed `() => S`: a function (or a `computed`) combining several signals:

```ts
selectedOption = linkedSignal<
  { options: ShippingMethod[]; country: string },
  ShippingMethod
>({
  source: () => ({ options: this.shippingOptions(), country: this.country() }),
  computation: ({ options, country }, previous) => {
    const available = options.filter((o) => o.country === country);
    return available.find((o) => o.id === previous?.value.id) ?? available[0]; // keeps the selection if still available
  },
});
```

- Reset when `shippingOptions` or `country` changes; signals read in `computation` are tracked too.
- `previous.source` is the previous source object, `previous.value` the previous value of the `linkedSignal` (possibly set by `set()`): `previous?.source.country !== country` tells which source changed.
- With `previous`, the generic types must be given explicitly: source type, then value type.

It replaces the `effect(() => this.selected.set(...))` pattern, which is discouraged.

## Async data: resource, rxResource, httpResource

> **In short**: A resource loads data asynchronously from reactive `params`, and exposes `value()`, `isLoading()`, `error()` and `status()` as signals. It reloads when the params change, cancelling the previous load. `httpResource()` uses `HttpClient`, `rxResource()` an Observable; resources are meant for reading, not writing.

Signals are synchronous: a **resource** brings asynchronous loading into the signal graph. It reloads when its `params` change, and exposes the value, the status and the error as signals.

```ts
userId = input.required<number>();

user = resource({
  params: () => ({ id: this.userId() }), // reactive: reloads when userId changes
  loader: ({ params, abortSignal }) =>
    fetch(`/api/users/${params.id}`, { signal: abortSignal }).then(
      (r) => r.json() as Promise<User>,
    ),
});
```

```html
@if (user.isLoading()) {
<app-spinner />
} @else if (user.error()) {
<p>Loading failed</p>
} @else if (user.hasValue()) {
<p>{{ user.value().name }}</p>
}
```

- Signals: `value()`, `hasValue()`, `error()`, `isLoading()`, `status()` (`idle`, `loading`, `reloading`, `resolved`, `error`, `local`); methods `reload()`, `set()` / `update()` (local value).
- A change of `params` during a load aborts it (`abortSignal`); `params` returning `undefined` doesn't load (`idle`).
- `value()` throws in `error` status: guard with `hasValue()`, or with an `@boundary` / `@error` block (see [Angular](ANGULAR.md)).

Options of `resource()`: only `loader` (or `stream`) is required.

| Option | Role |
|---|---|
| `params` | reactive function computing the request (`request` before Angular 20); its result changing reloads. Without `params`, the loader runs only on `reload()`. Returning `undefined`: no load (`idle`) |
| `loader` | async function returning a `Promise`: one value per load |
| `stream` | instead of `loader`, for a continuous source (WebSocket, SSE): several values over time |
| `defaultValue` | value of `value()` while there is no result (loading, idle), instead of `undefined`; the type becomes `T` instead of `T \| undefined` |
| `equal` | equality function for the loaded values |
| `injector` | to create the resource outside an injection context |
| `id` | cache key to transfer the loaded data from the server to the client in SSR (`TransferState`) |

- The loader receives `params`, `abortSignal` (to pass to `fetch`) and `previous` (`{ status }`, e.g. no spinner during a `reload`).
- Only the signals read in `params` trigger a reload: everything the request depends on goes through `params`.

**`httpResource()`** (`@angular/common/http`): the same, built on `HttpClient` (interceptors included), with a URL or a request computed from signals:

```ts
user = httpResource<User>(() => `/api/users/${this.userId()}`);
orders = httpResource<Order[]>(() => ({
  url: "/api/orders",
  params: { status: this.status(), page: this.page() },
}));
```

- First argument: a function returning a URL (GET) or an `HttpResourceRequest` (`url`, `method`, `params`, `headers`, `body`...); `undefined` sends nothing.
- Options (second argument): `defaultValue`, `parse` (validate or transform the response, e.g. with a Zod schema), `equal`, `injector`, `debugName`.
- JSON by default; `httpResource.text()`, `httpResource.blob()`, `httpResource.arrayBuffer()` for other response types.
- Like `resource()`, a change of the URL or request during a load cancels the previous request (unsubscription from `HttpClient`): a slow response for an old value can't overwrite the new one, as with `switchMap`.

```ts
users = httpResource<User[]>(() => `/api/teams/${this.teamId()}/users`, {
  defaultValue: [],                          // value(): User[], never undefined
  parse: (raw) => userListSchema.parse(raw), // validates the response
});
avatar = httpResource.blob(() => `/api/users/${this.userId()}/avatar`);
```

**`rxResource()`** (`@angular/core/rxjs-interop`): the loader returns an `Observable` (`stream` option), to reuse an existing service based on `HttpClient` / RxJS:

```ts
user = rxResource({
  params: () => this.userId(),
  stream: ({ params: id }) => this.userService.getUser(id),
});
```

**`debounced()`** (Angular 22, experimental): a debounced version of a signal, e.g. to wait for the user to stop typing before searching:

```ts
query = signal("");
debouncedQuery = debounced(this.query, 300);
results = httpResource<Product[]>(
  () => `/api/products?q=${this.debouncedQuery.value()}`,
);
```

Resources are for **reading** data. For writes (POST, PUT), call `HttpClient` from a method, then `reload()` or `set()` the resource.

## RxJS interop: toSignal and toObservable

> **In short**: `toSignal()` turns an Observable into a signal (subscribed for the component's lifetime), `toObservable()` the opposite. Signals for state and templates, RxJS for complex event flows (`debounceTime`, `switchMap`), converted at the boundary.

Signals and RxJS coexist (`@angular/core/rxjs-interop`):

```ts
// Observable → Signal: subscribes immediately, unsubscribes on destroy
products = toSignal(this.http.get<Product[]>("/api/products"), {
  initialValue: [],
});
params = toSignal(this.route.paramMap);

// Signal → Observable: to use RxJS operators on a signal
results = toSignal(
  toObservable(this.query).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap((q) => this.http.get<Product[]>(`/api/products?q=${q}`)),
  ),
  { initialValue: [] },
);
```

- Without `initialValue`, the signal is `T | undefined` until the first emission; `requireSync: true` for an Observable emitting synchronously (`BehaviorSubject`).
- `takeUntilDestroyed()`: unsubscribes from an Observable when the component is destroyed.

| Signals                               | RxJS                                                                                         |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| state: a current value at any time    | events and streams over time                                                                 |
| synchronous, read with `count()`      | asynchronous, needs a subscription                                                           |
| derived state (`computed`), templates | complex async flows: `debounceTime`, `switchMap`, retries, WebSocket, combination of streams |

The trend: signals for the state and the templates, RxJS kept for complex event flows, converted at the boundary.

## Other signal APIs

> **In short**: `afterRenderEffect()` for DOM work after rendering, Signal Forms (`@angular/forms/signals`) for forms whose state is signals, and utilities such as `isSignal()` and the `equal` option.

- **`afterRenderEffect()`**: an effect run after the DOM has been rendered, to read or write the DOM (measure an element, draw on a canvas), with phases (`earlyRead`, `write`, `mixedReadWrite`, `read`).
- **Signal Forms** (`@angular/forms/signals`, experimental in Angular 21): forms whose model and validation state are signals: `form(signal({ email: '' }), path => { required(path.email); email(path.email); })`.
- Utilities: `isSignal()`, `isWritableSignal()`, `untracked()`, the `equal` option on `signal` / `computed`.

## Summary of the signal APIs

> **In short**: The signal APIs at a glance, with their package: state (`signal`, `computed`, `linkedSignal`, `effect`), components (`input`, `model`, `output`, queries), async (`resource`, `httpResource`, `rxResource`) and RxJS interop.

| API                                                                    | Package                      | Role                                        |
| ---------------------------------------------------------------------- | ---------------------------- | ------------------------------------------- |
| `signal()`                                                             | `@angular/core`              | writable state                              |
| `computed()`                                                           | `@angular/core`              | derived, read-only, lazy and memoized       |
| `effect()`                                                             | `@angular/core`              | side effect when signals change             |
| `untracked()`                                                          | `@angular/core`              | read without tracking                       |
| `linkedSignal()`                                                       | `@angular/core`              | writable, reset when its source changes     |
| `input()`, `input.required()`                                          | `@angular/core`              | component input                             |
| `model()`                                                              | `@angular/core`              | two-way binding                             |
| `output()`                                                             | `@angular/core`              | component event (not a signal)              |
| `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()` | `@angular/core`              | queries                                     |
| `resource()`                                                           | `@angular/core`              | async loading with a Promise                |
| `httpResource()`                                                       | `@angular/common/http`       | async loading with `HttpClient`             |
| `rxResource()`                                                         | `@angular/core/rxjs-interop` | async loading with an Observable            |
| `debounced()`                                                          | `@angular/core`              | debounced signal (experimental, Angular 22) |
| `toSignal()`, `toObservable()`                                         | `@angular/core/rxjs-interop` | conversion with RxJS                        |
| `afterRenderEffect()`                                                  | `@angular/core`              | effect after rendering, for the DOM         |
| `form()`                                                               | `@angular/forms/signals`     | Signal Forms                                |

## Pitfalls

> **In short**: Mutating a signal's value in place notifies nothing, effects used to sync state create hidden flows, signals read after an `await` aren't tracked, and a `computed` must stay pure.

- **Mutation in place**: `items().push(x)` or `user().name = 'x'` doesn't notify anything: `update(items => [...items, x])`, `update(u => ({ ...u, name: 'x' }))`.
- **Effects to sync state**: setting a signal from an effect creates hidden flows and extra change detections: prefer `computed` or `linkedSignal`.
- **Reading after `await`** in an effect or a `computed`: not tracked.
- **Calling a signal in the template** (`{{ count() }}`) is the intended usage, unlike calling a method: reading a signal is cheap and tracked.
- **`computed` with side effects**: a `computed` must be pure, it may run zero or several times.
- **Writing a signal in a `computed`**: forbidden (error); in a template expression too.
