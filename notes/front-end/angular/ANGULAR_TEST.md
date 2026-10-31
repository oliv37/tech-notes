## Testing Angular components: TestBed and test runners

> **In short**: `TestBed` is Angular's testing module: it configures an environment (component, providers, mocks), creates the component in a real DOM, and gives a `ComponentFixture` to interact with it. Angular Testing Library is a layer over `TestBed` that queries the DOM like a user (by role, label, text) and runs change detection after events. Vitest is the default test runner since Angular 21 (Karma is deprecated).

- **`TestBed.configureTestingModule({ imports, providers })`**: declares the standalone component under test and replaces its dependencies (`{ provide: UserService, useValue: mock }`).
- **`TestBed.createComponent(C)`** returns a `ComponentFixture`: `nativeElement` (the DOM), `componentInstance`, `componentRef.setInput()` for inputs, `whenStable()` / `detectChanges()` for change detection.
- **`TestBed.inject(Service)`** gets a service from the test injector, to test a service alone.
- `TestBed` is zoneless by default: after an action, `await fixture.whenStable()` waits for change detection (see [Angular change detection](ANGULAR_CHANGE_DETECTION.md#zoneless-removing-zonejs)).

The component tested in the two examples below:

```ts
@Component({
  selector: "app-user-search",
  template: `
    <label for="query">Name</label>
    <input id="query" [value]="query()" (input)="query.set($any($event.target).value)" />
    <button (click)="search()">Search</button>
    <ul>
      @for (user of users(); track user.id) {
        <li>{{ user.name }}</li>
      } @empty {
        <p>No user found</p>
      }
    </ul>
  `,
})
export class UserSearchComponent {
  private readonly userService = inject(UserService);
  protected readonly query = signal("");
  protected readonly users = signal<User[]>([]);

  protected search() {
    this.userService.search(this.query()).subscribe((users) => this.users.set(users));
  }
}
```

## A component test with TestBed

> **In short**: Configure `TestBed` with the component and a mocked service, create the fixture, act on the DOM through `nativeElement` (set a value, dispatch an event, click), wait for change detection with `whenStable()`, then check the DOM and the calls to the mock.

```ts
describe("UserSearchComponent", () => {
  let fixture: ComponentFixture<UserSearchComponent>;
  let userService: { search: Mock };

  beforeEach(async () => {
    userService = { search: vi.fn() };
    TestBed.configureTestingModule({
      imports: [UserSearchComponent],
      providers: [{ provide: UserService, useValue: userService }],
    });
    fixture = TestBed.createComponent(UserSearchComponent);
    await fixture.whenStable();
  });

  it("displays the users found", async () => {
    userService.search.mockReturnValue(of([{ id: 1, name: "Alice" }]));
    const element: HTMLElement = fixture.nativeElement;

    const input = element.querySelector<HTMLInputElement>("#query")!;
    input.value = "ali";
    input.dispatchEvent(new Event("input")); // what the browser does when the user types
    element.querySelector("button")!.click();
    await fixture.whenStable();

    expect(userService.search).toHaveBeenCalledWith("ali");
    expect(element.querySelectorAll("li")).toHaveLength(1);
    expect(element.textContent).toContain("Alice");
  });
});
```

- The DOM is reached with CSS selectors (`#query`, `button`, `li`): the test depends on the structure of the template. `fixture.debugElement.query(By.css(...))` is the Angular wrapper around the same idea.
- Typing is simulated by hand: set `value`, then dispatch the `input` event the component listens to.
- `componentInstance` gives access to the component, but testing through the DOM checks what the user actually sees.

## The same test with Angular Testing Library

> **In short**: `render()` configures `TestBed` and creates the component in one call; `screen` finds elements the way a user does (`getByRole`, `getByLabelText`, `getByText`), and `userEvent` simulates real interactions. Change detection runs automatically after each event, and `findBy...` waits for the element to appear. The test no longer depends on CSS selectors or the template's structure.

```ts
import { render, screen } from "@testing-library/angular";
import userEvent from "@testing-library/user-event";

it("displays the users found", async () => {
  const search = vi.fn().mockReturnValue(of([{ id: 1, name: "Alice" }]));
  await render(UserSearchComponent, {
    providers: [{ provide: UserService, useValue: { search } }],
  });
  const user = userEvent.setup();

  await user.type(screen.getByLabelText("Name"), "ali"); // focus + one input event per key
  await user.click(screen.getByRole("button", { name: "Search" }));

  expect(search).toHaveBeenCalledWith("ali");
  expect(await screen.findByText("Alice")).toBeInTheDocument();
  expect(screen.queryByText("No user found")).not.toBeInTheDocument();
});
```

- **Queries, by order of preference**: `getByRole` (accessible role and name: also checks that the page is accessible), `getByLabelText`, `getByText`, then `getByTestId` as a last resort.
- **`getBy...`** throws if the element is missing, **`queryBy...`** returns `null` (to check an absence), **`findBy...`** returns a promise and retries until the element appears (async results).
- **`userEvent`** reproduces the browser's sequence (focus, keydown, input, keyup for each character); `fireEvent` dispatches a single event, closer to the TestBed example.
- `render()` options: `providers`, `imports`, `inputs` (component inputs), `on` (outputs), `routes`; `componentInputs` is deprecated in favor of `inputs`.
- `toBeInTheDocument()` comes from `@testing-library/jest-dom`, registered in the test setup file.

## TestBed vs Angular Testing Library

> **In short**: Both run the component in `TestBed` with the same mocks; the difference is the way the test talks to the DOM. Plain `TestBed` is built in and gives full control, but tests are coupled to the template's structure and change detection is handled by hand. Testing Library tests behave like a user, are shorter and survive template refactorings.

| | TestBed | Angular Testing Library |
|---|---|---|
| Setup | `configureTestingModule` + `createComponent` | `render(Component, { providers })` |
| Finding elements | CSS selectors (`querySelector`, `By.css`) | role, label, text (`getByRole`, `getByLabelText`) |
| User interactions | set values and dispatch events by hand | `userEvent.type`, `userEvent.click` |
| Change detection | `await fixture.whenStable()` / `detectChanges()` | automatic after each event |
| Async results | wait explicitly | `findBy...` retries until it appears |
| Dependency | built into Angular | `@testing-library/angular` + `user-event` + `jest-dom` |

Related tools:

- **HTTP**: `provideHttpClient()` + `provideHttpClientTesting()`, then `HttpTestingController` to check the requests and answer them (`expectOne('/api/users').flush([...])`), instead of mocking the service.
- **Inputs**: `fixture.componentRef.setInput("userId", 42)` with TestBed (works with signal inputs), the `inputs` option with Testing Library.
- **Component harnesses** (Angular CDK): an API to drive Angular Material components (`MatButtonHarness`) without depending on their internal DOM.
