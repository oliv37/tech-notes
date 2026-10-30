## What a TypeScript class compiles to

> **In short**: A TypeScript class is a JavaScript class plus types: with a modern target it's emitted as a native `class`, with `target: "ES5"` as a constructor function and prototypes (see [JavaScript](../javascript/JAVASCRIPT.md#classes-before-es6-constructor-functions-and-prototypes)). The TypeScript-only keywords (`private`, `readonly`, `abstract`, `implements`) are erased; parameter properties are one of the few features that generate code.

```typescript
class User {
  constructor(public name: string) {}

  hello() {
    return `Hi ${this.name}`;
  }
}
```

```javascript
// emitted with target ES2022: the modifiers and types are gone, the parameter property is now code
class User {
  constructor(name) {
    this.name = name;
  }

  hello() {
    return `Hi ${this.name}`;
  }
}
```

- `public name` in the constructor is a **parameter property**: the modifier (`public`, `private`, `protected` or `readonly`) is what declares the field and assigns it. Without a modifier, `constructor(name: string)` only receives a parameter and creates no field. Members are `public` by default, but here `public` can't be omitted.
- It's one of the few TypeScript features that generates code (`this.name = name`), like `enum`: the `erasableSyntaxOnly` flag (TS 5.8) forbids them, for code run by Node's type stripping.
- `private`, `protected`, `readonly`, `abstract`, `implements` and the types produce no code: a `private` field is still readable at runtime (`user["secret"]`). Only `#field` is truly private at runtime (emitted with a `WeakMap` for old targets).
- With `target: "ES5"`, `tsc` emits a constructor function wrapped in an IIFE (`var User = (function () { ... return User; }())`), and inheritance goes through an `__extends` helper that links the prototypes.

Pitfall with `target: "ES5"`: `class NotFoundError extends Error` compiles to the ES5 pattern, and `err instanceof NotFoundError` is `false`, since `Error.call(this)` ignores `this` and returns a new object. The fix was `Object.setPrototypeOf(this, NotFoundError.prototype)` in the constructor. Modern targets (Angular emits ES2022) keep native classes, so the problem is gone, but it still shows up in old code.
