## Lifecycle hooks

> **In short**: Angular calls a component's hooks in a fixed order: `constructor`, `ngOnChanges` (inputs set or changed), `ngOnInit` (once, inputs available), `ngDoCheck`, `ngAfterContentInit` / `Checked` (projected content), `ngAfterViewInit` / `Checked` (own template and children), then `ngOnDestroy`. The `...Init` hooks run once, `ngOnChanges`, `ngDoCheck` and the `...Checked` hooks run again at each change detection.

| Hook                    | When                                                          | Runs                 | Typical use                                                         |
| ----------------------- | ------------------------------------------------------------- | -------------------- | ------------------------------------------------------------------- |
| `constructor`           | the class is created                                          | once                 | `inject()` dependencies; inputs aren't set yet                      |
| `ngOnChanges(changes)`  | an input is set or changes (before `ngOnInit` the first time) | at each input change | react to an input (`changes.userId.previousValue` / `currentValue`) |
| `ngOnInit`              | after the first `ngOnChanges`                                 | once                 | initialization that needs the inputs (load data)                    |
| `ngDoCheck`             | at each change detection of the component                     | often                | custom change detection (rare, costly)                              |
| `ngAfterContentInit`    | the projected content (`<ng-content>`) is ready               | once                 | read `@ContentChild`                                                |
| `ngAfterContentChecked` | after each check of the projected content                     | often                | rare                                                                |
| `ngAfterViewInit`       | the component's template and its children are ready           | once                 | read `@ViewChild`, start a third-party library on an element        |
| `ngAfterViewChecked`    | after each check of the view                                  | often                | rare                                                                |
| `ngOnDestroy`           | just before the component is removed (`@if`, navigation)      | once                 | unsubscribe, clear timers                                           |

- **Content vs view**: the content is what the parent puts between the tags (`<app-card><p>text</p></app-card>`), the view is the component's own template.
- **Parent / child order**: the parent's `ngOnInit` runs before its children's, but a child's `ngAfterViewInit` runs before its parent's, since a view is only "ready" when its children are.
- **Change a value in `ngAfterViewInit`**, and dev mode throws `ExpressionChangedAfterItHasBeenCheckedError`: the parent's bindings have already been checked (see [Angular change detection](ANGULAR_CHANGE_DETECTION.md)).
- Implementing the interface (`implements OnInit`) is optional but catches typos: a method named `ngOnint` is silently never called.

## The modern alternatives with signals and DestroyRef

> **In short**: With signals, most hooks become optional: `computed` replaces `ngOnChanges` for derived values, `effect` reacts to any change instead of `ngDoCheck`, signal queries (`viewChild()`) replace reading `@ViewChild` in `ngAfterViewInit`, and `DestroyRef` / `takeUntilDestroyed()` replace `ngOnDestroy`. `ngOnInit` remains common, for work that needs the inputs once.

| Before                                           | With signals                                                                    |
| ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `ngOnChanges` to recompute a value from an input | `fullName = computed(() => ...)` on signal inputs                               |
| `ngOnChanges` to react to an input (reload data) | `effect()`, or `resource()` / `httpResource()` with the input as parameter      |
| `ngDoCheck`                                      | `effect()` / `computed()`: re-run only when their dependencies change           |
| `@ViewChild` read in `ngAfterViewInit`           | `chart = viewChild<ElementRef>("chart")`, a signal, `undefined` until available |
| `ngOnDestroy` + `Subject` to unsubscribe         | `takeUntilDestroyed()`, or `inject(DestroyRef).onDestroy(() => ...)`            |

```ts
export class UserCardComponent {
  readonly user = input.required<User>();
  protected readonly fullName = computed(
    () => `${this.user().firstName} ${this.user().lastName}`,
  );

  constructor() {
    interval(1000)
      .pipe(takeUntilDestroyed())
      .subscribe(() => this.refresh()); // stops on destroy
    inject(DestroyRef).onDestroy(() => console.log("destroyed"));
  }
}
```

- `takeUntilDestroyed()` and `inject(DestroyRef)` must be called in an injection context (constructor, field initializer), or receive a `DestroyRef` explicitly.
- `DestroyRef` also works in services and directives, and can be passed to helper functions that need to clean up.
- A required signal input read in the constructor throws (`NG0950`): it has no value yet. Read it in a `computed`, an `effect`, or from `ngOnInit`.
- See [Angular signals](ANGULAR_SIGNAL.md) for `computed`, `effect` and `resource`.

## afterNextRender and afterEveryRender

> **In short**: `afterNextRender` runs once, and `afterEveryRender` after every render, once Angular has finished updating the DOM of the **whole application**, not of one component. They're meant for direct DOM work: measuring an element, focusing it, starting a chart library. They never run on the server (SSR, prerendering), which makes them the safe place for browser-only code. Callbacks can be split into phases (`write`, `read`...) to avoid layout thrashing.

```ts
export class ChartComponent {
  private readonly canvas = viewChild.required<ElementRef<HTMLCanvasElement>>("canvas");

  constructor() {
    afterNextRender(() => {
      new Chart(this.canvas().nativeElement, { ... }); // the DOM exists, and we're in the browser
    });
  }
}
```

**vs `ngAfterViewInit`**:

|                       | `ngAfterViewInit`                                 | `afterNextRender`                                  |
| --------------------- | ------------------------------------------------- | -------------------------------------------------- |
| Scope                 | this component's view is created                  | the whole application has been rendered to the DOM |
| Server-side rendering | runs on the server too (no `window`, no real DOM) | never runs on the server                           |
| Declared              | as a method of the component                      | as a call in the constructor (injection context)   |
| Order with other code | before the DOM is fully painted                   | after every component has been rendered            |

**Phases**: reading a layout property (`offsetHeight`) right after writing a style forces the browser to recompute the layout, and alternating them in several components makes it recompute many times (layout thrashing). The phases group all the writes, then all the reads, across the application:

```ts
afterEveryRender({
  write: () => {
    panel.style.height = "auto";
    return true; // passed to the next phase
  },
  read: (didWrite) => {
    if (didWrite) this.height.set(panel.offsetHeight);
  },
});
```

- Order: `earlyRead`, `write`, `mixedReadWrite` (the default, when a plain function is passed), `read`. A phase's return value is passed to the next one.
- Like the other hooks, they must be called in an injection context (constructor), or receive an `injector` option. They return a ref whose `destroy()` stops the callback.
- `afterEveryRender` runs after **each** render of the application, triggered by any component: keep it cheap, or prefer `afterNextRender`.
- `afterEveryRender` was called `afterRender` before Angular 20.
- In zoneless applications, they replace `NgZone.onStable`, which never emits (see [Angular change detection](ANGULAR_CHANGE_DETECTION.md#zoneless-removing-zonejs)).

**Outside the constructor**: these functions use `inject()` internally, to find the application's render scheduler, the platform (nothing runs on the server) and the component's `DestroyRef` (to unregister the callback when the component is destroyed). `inject()` only works in an injection context: the constructor, a field initializer, a provider factory. Called from a click handler or a `subscribe`, they throw `NG0203`. The fix is to capture the injector in the constructor and pass it:

```ts
export class SearchPanelComponent {
  private readonly injector = inject(Injector);      // captured in the injection context
  private readonly destroyRef = inject(DestroyRef);
  private readonly input = viewChild.required<ElementRef<HTMLInputElement>>("input");
  protected readonly open = signal(false);

  onOpen() {
    this.open.set(true);
    afterNextRender(() => this.input().nativeElement.focus(), { injector: this.injector }); // focus once the input exists
  }

  onSearch(query: Observable<string>) {
    query.pipe(takeUntilDestroyed(this.destroyRef)).subscribe(...); // DestroyRef passed explicitly
  }
}
```

| Function                                                    | Outside an injection context                                                               |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `afterNextRender`, `afterEveryRender`, `effect`, `toSignal` | `{ injector: this.injector }` option                                                       |
| `takeUntilDestroyed`                                        | `takeUntilDestroyed(this.destroyRef)`                                                      |
| any code calling `inject()`                                 | `runInInjectionContext(this.injector, () => ...)` runs a function as if in the constructor |

## Which one to use

> **In short**: Initialize with the inputs in `ngOnInit` (or derive with `computed`), react to changes with `computed` / `effect`, touch the DOM in `afterNextRender`, clean up with `DestroyRef` / `takeUntilDestroyed()`. The `...Checked` hooks and `ngDoCheck` are rarely needed.

| Need                                               | Use                                                                |
| -------------------------------------------------- | ------------------------------------------------------------------ |
| inject dependencies                                | `constructor` / field initializers with `inject()`                 |
| initialize with the inputs, once                   | `ngOnInit`                                                         |
| a value derived from inputs or state               | `computed()`                                                       |
| react to an input or state change                  | `effect()`, `resource()`                                           |
| access an element of the template                  | `viewChild()` (signal)                                             |
| measure, focus, start a DOM library (browser only) | `afterNextRender()`                                                |
| sync with the DOM after each render                | `afterEveryRender()`                                               |
| clean up                                           | `takeUntilDestroyed()`, `DestroyRef.onDestroy()`, or `ngOnDestroy` |
