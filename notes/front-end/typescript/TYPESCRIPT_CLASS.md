## Prototype and prototype chain

> **In short**: Every JavaScript object has a hidden link to another object, its prototype. When a property isn't found on the object itself, JavaScript looks it up on the prototype, then on the prototype's prototype, up to `null`: the prototype chain. Methods of a class live once on `Class.prototype` and are shared by all instances.

```typescript
class User {
  constructor(public name: string) {}
  hello() {
    return `Hi ${this.name}`;
  }
}

const alice = new User("Alice");
```

```
alice                    User.prototype           Object.prototype
{ name: "Alice" }  ───►  { hello() }        ───►  { toString(), ... }  ───►  null
```

- `name` is an own property of `alice`; `hello` isn't copied into each instance, it exists once on `User.prototype`.
- `alice.hello()`: not found on `alice`, found on `User.prototype`. `alice.toString()`: found one level higher, on `Object.prototype`.
- `new User(...)` is what sets the link: the new object's prototype is `User.prototype`.
- The link can be read with `Object.getPrototypeOf(alice)` (the old non-standard `alice.__proto__` does the same).

## instanceof

> **In short**: `x instanceof C` is a JavaScript runtime check: is `C.prototype` somewhere in the prototype chain of `x`? TypeScript uses it to narrow the type. It only works with classes (values that exist at runtime), not interfaces or types, and it fails on objects that weren't created with `new`, like parsed JSON.

```typescript
class Admin extends User {}
const bob = new Admin("Bob");
// bob ──► Admin.prototype ──► User.prototype ──► Object.prototype

bob instanceof Admin; // true
bob instanceof User; // true: User.prototype is found higher in the chain
```

TypeScript checks the **shape** of an object, `instanceof` checks **how it was created**:

```typescript
const fromJson: User = JSON.parse('{"name":"Alice"}'); // accepted by the compiler
// fromJson ──► Object.prototype

fromJson instanceof User; // false
fromJson.hello(); // TypeError: hello is not a function
```

Pitfalls:

- Interfaces and types are erased: `x instanceof SomeInterface` doesn't compile. Use `in`, a discriminant (`x.type === ...`) or a custom guard (`x is T`).
- Primitives aren't instances: `"a" instanceof String` is `false`, use `typeof`.
- Two copies of the same class (two versions of a library in the bundle, an object from an iframe) have different prototypes: `instanceof` returns `false` for an object of the right shape.

## Classes before ES6: constructor functions and prototypes

> **In short**: `class` (ES6 / ES2015) is mostly syntax over what existed before: a constructor function called with `new`, methods added to its `prototype` object (`User.prototype.hello = ...`), and inheritance by linking the prototypes (`Object.create`) and calling the parent constructor (`Parent.call(this)`). TypeScript compiled to this when targeting ES5.

**`new F(args)`** does three things:

1. creates an empty object whose prototype is `F.prototype`;
2. calls `F` with `this` bound to that object;
3. returns the object (unless `F` explicitly returns another object).

**A class and its ES5 equivalent**:

```typescript
class User {
  static count = 0;

  constructor(public name: string) {
    User.count++;
  }

  hello() {
    return `Hi ${this.name}`;
  }
}
```

```javascript
// constructor
function User(name) {
  this.name = name; // parameter property `public name`
  User.count++;
}
User.count = 0; // static: a property of the function itself

// method: shared on the prototype
User.prototype.hello = function () {
  return "Hi " + this.name;
};

var alice = new User("Alice");
```

`tsc` wraps it in an IIFE (`var User = (function () { ... return User; }())`), but the content is the same.

- `public name` in the constructor is a **parameter property**: the modifier (`public`, `private`, `protected` or `readonly`) is what declares the field and assigns it. Without a modifier, `constructor(name: string)` only receives a parameter and creates no field. Members are `public` by default, but here `public` can't be omitted.
- It's one of the few TypeScript features that generates code (`this.name = name`), like `enum`: the `erasableSyntaxOnly` flag (TS 5.8) forbids them, for code run by Node's type stripping.

**Inheritance**:

```typescript
class Admin extends User {
  constructor(
    name: string,
    public role: string,
  ) {
    super(name);
  }

  hello() {
    return `${super.hello()} (admin)`;
  }
}
```

```javascript
function Admin(name, role) {
  User.call(this, name); // super(name): run the parent constructor on this object
  this.role = role;
}

Admin.prototype = Object.create(User.prototype); // extends: Admin.prototype ──► User.prototype
Admin.prototype.constructor = Admin; // restore the link overwritten above
Object.setPrototypeOf(Admin, User); // inherit statics (Admin.count)

Admin.prototype.hello = function () {
  return User.prototype.hello.call(this) + " (admin)"; // super.hello()
};
```

- `Object.create(User.prototype)` creates an empty object whose prototype is `User.prototype`: this builds the chain `bob ──► Admin.prototype ──► User.prototype`, which is what makes `instanceof User` true.
- `tsc` puts these lines in an `__extends` helper.
- TypeScript-only keywords produce no code: `private`, `protected`, `readonly`, `abstract`, `implements` and the types are erased. Only `#field` is truly private at runtime (emitted with a `WeakMap` for old targets).

**Where `class` is more than syntax**:

|                            | Constructor function                                              | `class`                                |
| -------------------------- | ----------------------------------------------------------------- | -------------------------------------- |
| Called without `new`       | runs, `this` is the global object (or `undefined` in strict mode) | `TypeError`                            |
| Hoisting                   | the function is usable before its declaration                     | not usable before (temporal dead zone) |
| Methods                    | enumerable (show up in `for...in`)                                | non-enumerable                         |
| Strict mode                | opt-in (`"use strict"`)                                           | always                                 |
| Extending `Error`, `Array` | broken                                                            | works                                  |

Pitfall with the last row: with `target: "ES5"`, `class NotFoundError extends Error` compiles to the ES5 pattern, and `err instanceof NotFoundError` is `false`, since `Error.call(this)` ignores `this` and returns a new object. The fix was `Object.setPrototypeOf(this, NotFoundError.prototype)` in the constructor. Modern targets (Angular emits ES2022) keep native classes, so the problem is gone, but it still shows up in old code.
