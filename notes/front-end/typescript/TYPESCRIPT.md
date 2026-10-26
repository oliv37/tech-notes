## Structural typing and type erasure

> **In short**: TypeScript checks types by their **shape**, not their name: any object with the right properties is accepted (structural typing). And types only exist at compile time: they are erased in the generated JavaScript, so they can't be checked at runtime (`instanceof` on an interface is impossible).

```ts
interface User {
  name: string;
}

class Employee {
  constructor(public name: string, public salary: number) {}
}

const user: User = new Employee("Alice", 3000); // OK: Employee has a `name: string`
```

- Unlike Java (nominal typing), no `implements` is needed: having the properties is enough.
- **Type erasure**: `interface`, `type`, generics and annotations disappear at compilation. A JSON response typed `User` by `http.get<User>()` is never checked: if the API returns something else, TypeScript doesn't know. Runtime validation needs a library (Zod) or a manual type guard.
- Classes and enums exist at runtime (they produce JavaScript), interfaces and types don't.

## interface vs type

> **In short**: Both describe the shape of an object and are interchangeable in most cases. `interface` can be extended and merged (declaration merging); `type` can also name unions, tuples, primitives and computed types. Common convention: `interface` for object shapes, `type` for everything else.

```ts
interface User {
  id: number;
  name: string;
}
interface Admin extends User {
  permissions: string[];
}

type Status = "active" | "suspended";          // union: only with type
type Point = [number, number];                 // tuple
type AdminUser = User & { permissions: string[] }; // intersection, like extends
```

| | `interface` | `type` |
|---|---|---|
| Object shape | yes | yes |
| Unions, tuples, primitives, mapped types | no | yes |
| Extension | `extends` | intersection `&` |
| Declaration merging (two declarations with the same name are merged) | yes, e.g. to extend a library's interface | no, duplicate name error |
| Implemented by a class (`implements`) | yes | yes, if it's an object type |

## any vs unknown vs never

> **In short**: `any` disables type checking: anything is allowed, errors move to runtime. `unknown` also accepts any value, but must be narrowed before use: the safe choice for a value of unknown type (parsed JSON, `catch` error). `never` is the type with no value: a function that never returns, or an impossible case.

```ts
let a: any = JSON.parse(text);
a.foo.bar(); // compiles, may crash at runtime

let u: unknown = JSON.parse(text);
u.foo;       // compile error: narrow it first
if (typeof u === "object" && u !== null && "foo" in u) { ... }

function fail(message: string): never {
  throw new Error(message); // never returns
}
```

- `catch (e)` is `unknown` with `strict` (`useUnknownInCatchVariables`): `if (e instanceof Error) e.message`.
- `never` is used for exhaustive checks: in the `default` of a `switch`, assigning the value to `never` fails to compile if a case is missing (see discriminated unions below).
- `void`: a function returning nothing useful; different from `never`, which never returns at all.

## strict mode and null handling

> **In short**: `"strict": true` in `tsconfig.json` enables the important checks, mainly `strictNullChecks`: `null` and `undefined` are no longer accepted by every type, so `string | undefined` must be checked before use. Optional chaining (`?.`) and nullish coalescing (`??`) make these checks short.

```ts
interface User {
  name: string;
  address?: { city: string }; // optional: { city: string } | undefined
}

user.address.city;                  // error with strictNullChecks: address may be undefined
user.address?.city ?? "unknown";    // ?. stops at undefined, ?? gives a default value
```

- `??` only replaces `null` / `undefined`, unlike `||`, which also replaces `0`, `""` and `false`: `count ?? 10` keeps a `0`.
- Non-null assertion `user.address!.city`: tells the compiler "trust me", no runtime check; to avoid.
- Other strict checks: `noImplicitAny` (a parameter without type is an error instead of `any`), `strictPropertyInitialization` (class fields initialized in the constructor). Angular projects are strict by default.

## Union, intersection and literal types

> **In short**: A union `A | B` is a value of one type or the other; an intersection `A & B` has the properties of both. Literal types (`"GET" | "POST"`, `1 | 2`) restrict a value to a fixed list, often replacing enums.

```ts
type Id = string | number;                       // union
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE"; // literal types
type Timestamped = { createdAt: Date };
type Order = { id: number; total: number } & Timestamped; // intersection

function request(method: HttpMethod, url: string) { ... }
request("PATCH", "/api"); // error: not in the union
```

- On a union, only the members common to all types are accessible until the type is narrowed (see below).
- **Literal union vs `enum`**: a union of strings is erased at compilation (no JavaScript generated) and works directly with the values of an API; an `enum` generates a runtime object. Unions (or `as const` objects) are usually preferred.

## `as const`

> **In short**: `as const` infers the narrowest literal types and makes the whole structure `readonly`: `["red", "green"] as const` is `readonly ["red", "green"]` instead of `string[]`.

By default, TypeScript widens literal expressions to their general type: an array literal becomes a mutable array of a union type, a string literal becomes `string`. `as const` tells it to do the opposite — infer the narrowest possible literal types, and make the whole structure `readonly`.

```ts
const colors = ["red", "green", "blue"];
// string[] — mutable, values widened to `string`

const colorsConst = ["red", "green", "blue"] as const;
// readonly ["red", "green", "blue"] — a fixed-length tuple of literal types
```

## `satisfies` vs explicit type annotation

> **In short**: Both check a value against a type, but an annotation (`const x: Foo`) widens the value to `Foo`, while `satisfies Foo` validates it and keeps its precise inferred (literal) type.

Both check a value against a type, but they affect the value's inferred type differently:

- **Explicit annotation** (`const x: Foo = {...}`) widens the value to exactly type `Foo`, so afterward `x` only has the properties/types `Foo` declares — any more specific (literal) type information is lost.
- **`satisfies`** (`const x = {...} satisfies Foo`) validates the value against `Foo` without changing its inferred type — `x` keeps its most specific inferred type, so consumers still benefit from literal types.

```ts
type Palette = Record<"red" | "green" | "blue", string>;

const explicit: Palette = { red: "#f00", green: "#0f0", blue: "#00f" };
// explicit.red is typed as `string`

const withSatisfies = {
  red: "#f00",
  green: "#0f0",
  blue: "#00f",
} satisfies Palette;
// withSatisfies.red is typed as the literal `"#f00"`
```

Both reject a value that doesn't match `Palette` (e.g. a missing key or a non-string value), but only `satisfies` preserves the narrower literal types for later use. That precision has a cost: since `withSatisfies.red` is typed as the literal `"#f00"`, reassigning it to any other string fails (`withSatisfies.red = "#ff0000"` errors), while `explicit.red` — typed as plain `string` — accepts any string.

## Type narrowing

> **In short**: Inside a branch guarded by a runtime check TypeScript recognizes (`typeof`, `instanceof`, `in`, equality, a custom `x is Type` guard), a broad type is narrowed to a more specific one. The narrowing only lasts within that branch.

Given a broader type (e.g. a union), TypeScript narrows it to a more specific one inside a branch, based on a runtime check it recognizes — `typeof`, `instanceof`, `in`, equality, truthiness, or a custom type guard (`x is Type`).

```ts
function printId(id: string | number) {
  if (typeof id === "string") {
    id.toUpperCase(); // id is narrowed to `string` here
  } else {
    id.toFixed(2); // id is narrowed to `number` here
  }
}

function isString(value: unknown): value is string {
  return typeof value === "string";
}
```

The narrowing only lasts within the branch where the check holds — outside of it (or if the variable is reassigned), TypeScript falls back to the original, wider type.

## Discriminated unions and exhaustive checks

> **In short**: A discriminated union is a union of object types sharing a literal property (the discriminant, e.g. `kind` or `status`). Checking that property narrows the type to the matching case, and a `never` check in the `default` branch makes the compiler report any unhandled case.

```ts
type PaymentState =
  | { status: "pending" }
  | { status: "paid"; transactionId: string }
  | { status: "failed"; reason: string };

function label(state: PaymentState): string {
  switch (state.status) {
    case "pending": return "Waiting";
    case "paid":    return `Paid (${state.transactionId})`; // transactionId accessible here only
    case "failed":  return `Failed: ${state.reason}`;
    default: {
      const unhandled: never = state; // compile error if a new status isn't handled
      return unhandled;
    }
  }
}
```

- Models states where each case has its own data, instead of one object with many optional fields (`transactionId?`, `reason?`), where impossible combinations compile.
- The TypeScript equivalent of Java's sealed interfaces + pattern matching `switch`.
- Used for error results in Angular services (see [Angular](../angular/ANGULAR.md), forcing the caller to handle errors).

## Generics

> **In short**: Generics write code that works with several types while keeping the type information: `function first<T>(items: T[]): T`. Constraints (`T extends { id: number }`) restrict the accepted types, and `keyof` links a type to the names of its properties.

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}
const n = first([1, 2, 3]); // T inferred as number

interface ApiResponse<T> {
  data: T;
  total: number;
}
http.get<ApiResponse<User[]>>("/api/users");

function byId<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id); // item.id allowed thanks to the constraint
}

function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]; // getProperty(user, "name") is a string, getProperty(user, "nope") doesn't compile
}
```

- Type parameters are usually inferred from the arguments; explicit when nothing to infer from (`http.get<User>()`).
- Default type: `interface Page<T = unknown>`.
- Erased at runtime, like all types: no `new T()` and no `instanceof T`.

## Utility types

> **In short**: Built-in generic types that derive a type from another one instead of rewriting it: `Partial` (all optional), `Required`, `Readonly`, `Pick` / `Omit` (keep / remove properties), `Record` (dictionary), `ReturnType`, `Awaited`, `NonNullable`.

```ts
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type UserUpdate = Partial<Omit<User, "id">>;  // { name?: string; email?: string; password?: string }
type PublicUser = Omit<User, "password">;     // without password
type UserPreview = Pick<User, "id" | "name">; // only id and name
type FrozenUser = Readonly<User>;             // properties read-only
type RolesByUser = Record<number, string[]>;  // { [id: number]: string[] }
```

| Type | Result |
|---|---|
| `Partial<T>` / `Required<T>` | all properties optional / required |
| `Readonly<T>` | all properties `readonly` (shallow) |
| `Pick<T, K>` / `Omit<T, K>` | only / all but the properties `K` |
| `Record<K, V>` | object with keys `K` and values `V` |
| `Exclude<U, X>` / `Extract<U, X>` | union without / with only the members assignable to `X` |
| `NonNullable<T>` | `T` without `null` and `undefined` |
| `ReturnType<F>` / `Parameters<F>` | return type / parameter tuple of a function type |
| `Awaited<T>` | type resolved by a `Promise` |

They are built with mapped and conditional types (see below).

## keyof, typeof, mapped and conditional types

> **In short**: `keyof T` is the union of the property names of `T`, `typeof value` gets the type of a value, and `T[K]` the type of a property. Mapped types (`{ [K in keyof T]: ... }`) transform each property of a type, conditional types (`T extends U ? X : Y`) choose a type depending on another: the tools behind the utility types.

```ts
interface User { id: number; name: string }

type UserKey = keyof User;            // "id" | "name"
type NameType = User["name"];         // string (indexed access)

const config = { apiUrl: "/api", retries: 3 };
type Config = typeof config;          // { apiUrl: string; retries: number }

const ROLES = ["admin", "user"] as const;
type Role = (typeof ROLES)[number];   // "admin" | "user": a union from an array

// mapped type: Partial reimplemented
type MyPartial<T> = { [K in keyof T]?: T[K] };

// conditional type, with infer: the element type of an array
type ElementOf<T> = T extends (infer E)[] ? E : never;
type N = ElementOf<number[]>;         // number
```

- `typeof` in a type position is TypeScript's (type of a value); in an expression, JavaScript's (`"string"`, `"object"`...).
- Template literal types build string types: `` type Event = `on${Capitalize<"click" | "focus">}` `` gives `"onClick" | "onFocus"`.
- Advanced: useful to read library types and write a few helpers, rarely needed in application code.
