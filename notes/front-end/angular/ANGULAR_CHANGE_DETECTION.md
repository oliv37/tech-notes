## Default change detection with Zone.js

> **In short**: Zone.js patches every async browser API (events, `setTimeout`, HTTP, promises). After each callback, it tells Angular "something may have changed", and Angular checks the whole component tree from the root, comparing every template binding with its previous value (dirty checking). Simple, since any field change ends up on screen, but each click or timer checks every component.

```ts
@Component({
  selector: "app-counter",
  template: `<p>{{ count }}</p>
    <button (click)="count = count + 1">+</button>`,
})
export class CounterComponent {
  count = 0;

  constructor() {
    setTimeout(() => (this.count = 10), 1000); // the view updates: Zone.js saw the setTimeout
  }
}
```

- **The trigger**: Zone.js wraps the callback of the `setTimeout`, the click listener, the HTTP response. When the callback and its microtasks are done (`NgZone.onMicrotaskEmpty`), Angular runs `ApplicationRef.tick()`.
- **The check**: every component, top-down, every binding re-evaluated: a method or getter called in a template runs at each tick.
- **Dev mode** checks a second time and throws `ExpressionChangedAfterItHasBeenCheckedError` if a binding changed in between (a value modified during the check itself, e.g. in `ngAfterViewInit`).
- Angular knows **that** something may have changed, never **what**: the cost grows with the size of the tree, not with the size of the change.

## OnPush

> **In short**: An `OnPush` component is checked only when it's marked dirty: a new reference on an input, an event in its template, an `async` pipe emission, a signal read in its template that changes, or `markForCheck()`. Otherwise Angular skips it and its whole subtree. It requires immutable data: mutating an object passed as input isn't detected. OnPush is the default for new components since Angular 22.

```ts
@Component({
  selector: "app-cart",
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `{{ items().length }} items`,
})
export class CartComponent {
  items = input<Item[]>([]);
}

// in the parent
this.items.push(item); // ❌ same reference: CartComponent isn't checked, still shows the old count
this.items = [...this.items, item]; // ✅ new reference: CartComponent is marked dirty
```

What marks an `OnPush` component dirty (with Zone.js, it's then checked at the next tick):

- an input receives a **new reference** (`===` comparison);
- an **event bound in its template** (or a child's) fires: `(click)`, `(input)`;
- the **`async` pipe** receives a value (it calls `markForCheck()`);
- a **signal read in its template** changes;
- **`ChangeDetectorRef.markForCheck()`**, which marks the component and its ancestors up to the root.

**An event in an OnPush child**: the event marks the component **and all its ancestors** dirty, up to the root, like `markForCheck()`. Change detection then starts from the root and goes down, skipping the OnPush components that aren't marked, with their subtree.

```text
            AppComponent (Default)
           /                    \
  HeaderComponent (OnPush)    PageComponent (OnPush)          ← ancestor: marked
                               /              \
                 SidebarComponent (OnPush)   ListComponent (OnPush)   ← ancestor: marked
                                                  |
                                            ItemComponent (OnPush)    ← (click) here: marked
                                                  |
                                            BadgeComponent (OnPush)
```

| Component                             | Checked?                              | Why                                                     |
| ------------------------------------- | ------------------------------------- | ------------------------------------------------------- |
| `AppComponent`                        | yes                                   | `Default`: always checked when the traversal reaches it |
| `HeaderComponent`, `SidebarComponent` | no                                    | OnPush, not marked: skipped with their subtree          |
| `PageComponent`, `ListComponent`      | yes                                   | ancestors, marked dirty                                 |
| `ItemComponent`                       | yes                                   | received the event                                      |
| `BadgeComponent`                      | only if an input gets a new reference | OnPush, not marked by the event                         |

- **Checked doesn't mean re-rendered**: the bindings of these components are re-evaluated (dirty checking), but the DOM is only updated for the bindings whose value changed.
- A `Default` component is checked whenever the traversal reaches it, even off the event's path.

Pitfalls:

- A field changed in a `setTimeout`, a `subscribe` or a `then` isn't displayed: none of the cases above happened. Use a signal, the `async` pipe, or `markForCheck()`.
- `markForCheck()` only marks the component for the next check; `detectChanges()` checks it (and its children) right away, synchronously.
- Since Angular 22, a component without a strategy is `OnPush`; the old behavior is called `ChangeDetectionStrategy.Eager` (added in 21.2 as an alias of `Default`), and `ng update` sets it explicitly on existing components.

## Zoneless: removing Zone.js

> **In short**: Without Zone.js, nothing watches the async APIs anymore: Angular only runs change detection when it's notified, by a signal read in a template, `markForCheck()` (and so the `async` pipe), a template or host event listener, or `ComponentRef.setInput()`. Zoneless is the default since Angular 21 (`provideZonelessChangeDetection()` in Angular 20). The code to fix is the code that changes a plain field in an async callback and relies on Zone.js to display it.

Enabling it:

1. Angular 20: add `provideZonelessChangeDetection()` to the providers; Angular 21+: it's the default, as long as `provideZoneChangeDetection()` doesn't bring Zone.js back.
2. Remove `zone.js` and `zone.js/testing` from the `polyfills` of the `build` and `test` targets in `angular.json`, then `npm uninstall zone.js`.
3. Make every component notify its changes: signals, the `async` pipe, `markForCheck()`. OnPush components that already work this way need no change.

What it brings: a smaller bundle, change detection only when something actually changed, readable stack traces (no Zone.js frames), and native `async` / `await` (Zone.js can't patch it, so the CLI had to transpile it).

What to watch out for:

- **Plain fields changed asynchronously**: `this.count = 10` in a `setTimeout` or a `subscribe` no longer updates the view, even in an `Eager` / `Default` component. That's the main migration work.
- **`NgZone.onStable`, `onMicrotaskEmpty`, `onUnstable`** never emit, and `NgZone.isStable` is always `true`: libraries waiting for them break. Use `afterNextRender()` / `afterEveryRender()` instead.
- **Reactive forms**: `setValue()` and `patchValue()` don't schedule change detection: `markForCheck()`, or expose the state through signals.
- **Tests**: `TestBed` is zoneless by default; prefer `await fixture.whenStable()` to `fixture.detectChanges()`. A binding updated without notification throws `ExpressionChangedAfterItHasBeenCheckedError`: the fix goes in the component (a signal or `markForCheck()`), not in the test.
- **SSR**: without Zone.js, Angular doesn't know when the async work is done before serializing: wrap it with the `PendingTasks` service.

## OnPush, zoneless and signals

> **In short**: The three solve the same question: when to refresh, and what. OnPush limits **what** is checked to the components marked dirty, zoneless limits **when** change detection runs to explicit notifications, and signals provide both automatically: a signal that changes marks exactly the components reading it, and schedules change detection. Signals + OnPush is what makes zoneless natural.

When a signal read in a template changes:

1. Angular marks **only the view of that component** for refresh, and its ancestors as "having a child to refresh", without marking the ancestors themselves dirty.
2. In zoneless mode, it schedules a change detection.
3. During the check, Angular goes down to that component and refreshes it, skipping the ancestors' own bindings: only the components reading the signal are re-rendered (local change detection, Angular 17+, with OnPush ancestors).

The difference with an event or `markForCheck()` (see OnPush above):

| Trigger                                        | The ancestors                                              |
| ---------------------------------------------- | ---------------------------------------------------------- |
| template event, `markForCheck()`, `async` pipe | **marked dirty**: their own bindings are re-evaluated      |
| signal read in the template                    | only **traversed**: their own bindings aren't re-evaluated |

In the tree above, a signal read by `ItemComponent` would refresh `ItemComponent` alone: an OnPush application driven by signals does even less work than one driven by `markForCheck()`.

|                            | Zone.js + Default          | Zone.js + OnPush                                   | Zoneless + OnPush + signals                          |
| -------------------------- | -------------------------- | -------------------------------------------------- | ---------------------------------------------------- |
| When change detection runs | after every async callback | after every async callback                         | only when notified                                   |
| What is checked            | the whole tree             | the components marked dirty and their path         | the components reading the changed signals           |
| What the code must do      | nothing                    | immutable inputs, `async` pipe or `markForCheck()` | state in signals (or `async` pipe, `markForCheck()`) |
| Cost                       | grows with the tree        | grows with the changes                             | grows with the changes                               |

- Signals work in every mode: with Zone.js and `Default`, they just add nothing, since the whole tree is checked anyway.
- `computed` replaces getters and methods in templates: recomputed only when its dependencies change, instead of at each check.
- See [Angular signals](ANGULAR_SIGNAL.md) for the signal APIs.
