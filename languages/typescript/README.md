# TypeScript

## The use case

JavaScript is dynamically typed: type errors only surface at runtime, refactoring a large codebase is risky, and editors can't reliably autocomplete or catch mistakes ahead of time. TypeScript was created by Microsoft to add optional static typing on top of JavaScript — catching type errors at compile time — while still compiling down to plain JavaScript that runs anywhere JS does.

## Key concepts

### `satisfies` vs explicit type annotation

Both check a value against a type, but they affect the value's inferred type differently:

- **Explicit annotation** (`const x: Foo = {...}`) widens the value to exactly type `Foo`, so afterward `x` only has the properties/types `Foo` declares — any more specific (literal) type information is lost.
- **`satisfies`** (`const x = {...} satisfies Foo`) validates the value against `Foo` without changing its inferred type — `x` keeps its most specific inferred type, so consumers still benefit from literal types.

```ts
type Palette = Record<"red" | "green" | "blue", string>;

const explicit: Palette = { red: "#f00", green: "#0f0", blue: "#00f" };
// explicit.red is typed as `string`

const withSatisfies = { red: "#f00", green: "#0f0", blue: "#00f" } satisfies Palette;
// withSatisfies.red is typed as the literal `"#f00"`
```

Both reject a value that doesn't match `Palette` (e.g. a missing key or a non-string value), but only `satisfies` preserves the narrower literal types for later use. That precision has a cost: since `withSatisfies.red` is typed as the literal `"#f00"`, reassigning it to any other string fails (`withSatisfies.red = "#ff0000"` errors), while `explicit.red` — typed as plain `string` — accepts any string.

### Type narrowing

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

### `as const`

By default, TypeScript widens literal expressions to their general type: an array literal becomes a mutable array of a union type, a string literal becomes `string`. `as const` tells it to do the opposite — infer the narrowest possible literal types, and make the whole structure `readonly`.

```ts
const colors = ["red", "green", "blue"];
// string[] — mutable, values widened to `string`

const colorsConst = ["red", "green", "blue"] as const;
// readonly ["red", "green", "blue"] — a fixed-length tuple of literal types
```

Unlike `satisfies`, `as const` doesn't check the value against anything — it only changes how the value is inferred. They're often combined (`{...} as const satisfies Foo`) to get both: literal, readonly types, validated against a shape.
