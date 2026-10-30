## Prototype chain and inheritance

> **In short**: Every JavaScript object has a hidden link to another object, its prototype. Reading a property that isn't on the object looks it up on the prototype, then the prototype's prototype, up to `null`: the prototype chain. Inheritance in JavaScript is this link between objects (prototypal inheritance), not a copy of a class: `class` and `extends` only build the chain. Reading goes up the chain, but assigning always creates an own property, which hides (shadows) the inherited one.

**Between plain objects**: `Object.create(proto)` creates an object whose prototype is `proto`.

```js
const animal = {
  eats: true,
  describe() {
    return `${this.name} eats: ${this.eats}`;
  },
};

const rabbit = Object.create(animal); // rabbit ──► animal ──► Object.prototype ──► null
rabbit.name = "Rabbit";

rabbit.describe(); // "Rabbit eats: true": describe and eats are found on animal
rabbit.toString(); // found on Object.prototype
rabbit.flies; // undefined: not found anywhere up to null
```

- **Lookup**: own property, then the prototype, and so on until `null`; the first match wins.
- **`this`** is the object the method was called on (`rabbit`), not the object where the method was found (`animal`): that's what makes shared methods work.
- **Assigning shadows**: `rabbit.eats = false` creates an own property on `rabbit`; `animal.eats` stays `true` for every other object inheriting from it (unless the inherited property is a setter or read-only).
- **Mutating doesn't shadow**: with `animal.tags = []`, `rabbit.tags.push("fast")` modifies the array of `animal`, shared by every object inheriting from it. Mutable state belongs on each instance, not on the prototype.

**A hierarchy without `class`**: objects inheriting from objects, each level created with `Object.create`.

```js
const user = {
  init(name) {
    this.name = name;
    return this;
  },
  hello() {
    return `Hi ${this.name}`;
  },
};

const admin = Object.create(user); // admin ──► user
admin.hello = function () {
  // overrides user.hello
  return `${user.hello.call(this)} (admin)`; // the "super" call, explicit
};

const bob = Object.create(admin).init("Bob"); // bob ──► admin ──► user ──► Object.prototype
bob.hello(); // "Hi Bob (admin)"
```

No constructor, no `new`: `user` and `admin` are plain objects acting as prototypes. The other way without `class` is the constructor function, what `class` replaced (see classes before ES6 below).

**With classes**: `new` links the instance to `Class.prototype`, and `extends` links the prototypes.

```js
class User {
  constructor(name) {
    this.name = name;
  }
  hello() {
    return `Hi ${this.name}`;
  }
}

class Admin extends User {
  hello() {
    return `${super.hello()} (admin)`;
  }
}

const bob = new Admin("Bob");
```

```text
bob                 Admin.prototype       User.prototype       Object.prototype
{ name: "Bob" } ──► { hello() }      ──►  { hello() }     ──►  { toString(), ... } ──► null
```

- `name` is an own property of `bob`; methods aren't copied into each instance, they exist once on their class's prototype and are shared.
- **Overriding** is shadowing: `Admin.prototype.hello` is found before `User.prototype.hello`; `super.hello()` starts the lookup from `User.prototype`.
- The link is read with `Object.getPrototypeOf(bob)` (the old `bob.__proto__` does the same). `Object.setPrototypeOf()` changes it, but is slow: create objects with the right prototype instead (`Object.create`, `new`).
- **`Object.create(null)`**: an object without prototype, so without `toString` or `hasOwnProperty`; a safe dictionary for arbitrary keys, although a `Map` is usually better.
- **Prototype pollution**: adding a property to `Object.prototype` adds it to every object. A naive deep merge of untrusted JSON containing a `"__proto__"` key can do it: a known security issue in JavaScript libraries.

## for...in vs Object.keys vs Object.getOwnPropertyNames

> **In short**: The three differ on two criteria: own properties vs inherited ones (from the prototype chain), and enumerable vs non-enumerable ones. `for...in` iterates over own **and inherited** enumerable properties; `Object.keys()` returns only the **own enumerable** ones; `Object.getOwnPropertyNames()` returns all the **own** ones, enumerable or not. None of them returns symbol keys: `Reflect.ownKeys()` returns every own key, symbols included.

```js
const parent = { inherited: 1 };
const obj = Object.create(parent); // obj ──► parent
obj.own = 2;
Object.defineProperty(obj, "hidden", { value: 3, enumerable: false });
obj[Symbol("id")] = 4;

for (const key in obj) console.log(key); // "own", "inherited"
Object.keys(obj); // ["own"]
Object.getOwnPropertyNames(obj); // ["own", "hidden"]
Reflect.ownKeys(obj); // ["own", "hidden", Symbol(id)]
```

|                                            | Own          | Inherited             | Non-enumerable | Symbols |
| ------------------------------------------ | ------------ | --------------------- | -------------- | ------- |
| `for...in`                                 | yes          | yes (enumerable ones) | no             | no      |
| `Object.keys()` / `values()` / `entries()` | yes          | no                    | no             | no      |
| `Object.getOwnPropertyNames()`             | yes          | no                    | yes            | no      |
| `Object.getOwnPropertySymbols()`           | symbols only | no                    | yes            | yes     |
| `Reflect.ownKeys()`                        | yes          | no                    | yes            | yes     |

- **Enumerable**: a property flag. Properties created by assignment (`obj.x = 1`) or in an object literal are enumerable; built-in ones (`length` of an array, the methods of `Object.prototype`) and those defined with `Object.defineProperty` (default `enumerable: false`) aren't. That's why `for...in` doesn't list `toString`.
- **Classes**: methods live on the prototype and are non-enumerable, fields are own enumerable properties. For `new User("Alice")`, `Object.keys()` gives `["name"]`, never `hello` (see above).
- **Arrays**: `for...in` returns the indexes as strings (`"0"`, `"1"`) plus any added or inherited enumerable property; use `for...of` for the values. `Object.getOwnPropertyNames([10, 20])` is `["0", "1", "length"]`.
- **Filtering `for...in`**: `if (Object.hasOwn(obj, key))` (ES2022, safer than `obj.hasOwnProperty(key)`) keeps only own properties; `Object.keys()` is usually simpler.
- The same "own enumerable string keys" rule applies to `JSON.stringify()`; spread (`{ ...obj }`) and `Object.assign()` copy own enumerable properties, symbols included.

## instanceof

> **In short**: `x instanceof C` checks whether `C.prototype` is somewhere in the prototype chain of `x`, i.e. whether `x` was created by `new C()` or by a subclass. TypeScript uses it to narrow the type. It only works with classes (values that exist at runtime), not TypeScript interfaces or types, and it fails on objects that weren't created with `new`, like parsed JSON.

```js
bob instanceof Admin; // true
bob instanceof User; // true: User.prototype is found higher in the chain
bob instanceof Object; // true
```

`instanceof` checks **how the object was created**, while TypeScript checks its **shape**:

```ts
const fromJson: User = JSON.parse('{"name":"Alice"}'); // accepted by the compiler
// fromJson ──► Object.prototype

fromJson instanceof User; // false
fromJson.hello(); // TypeError: hello is not a function
```

Pitfalls:

- TypeScript interfaces and types are erased: `x instanceof SomeInterface` doesn't compile. Use `in`, a discriminant (`x.type === ...`) or a custom guard (`x is T`).
- Primitives aren't instances: `"a" instanceof String` is `false`, use `typeof`.
- Two copies of the same class (two versions of a library in the bundle, an object from an iframe) have different prototypes: `instanceof` returns `false` for an object of the right shape.

## Classes before ES6: constructor functions and prototypes

> **In short**: `class` (ES6 / ES2015) is mostly syntax over what existed before: a constructor function called with `new`, methods added to its `prototype` object (`User.prototype.hello = ...`), and inheritance by linking the prototypes (`Object.create`) and calling the parent constructor (`Parent.call(this)`). A few differences make `class` more than syntax: it can't be called without `new`, isn't hoisted, and can extend built-ins like `Error`.

**`new F(args)`** does three things:

1. creates an empty object whose prototype is `F.prototype`;
2. calls `F` with `this` bound to that object;
3. returns the object (unless `F` explicitly returns another object).

**The `User` class above, written before ES6**:

```js
function User(name) {
  // the constructor
  this.name = name;
}
User.count = 0; // a static: a property of the function itself

User.prototype.hello = function () {
  // a method, shared on the prototype
  return "Hi " + this.name;
};

const alice = new User("Alice");
```

**Inheritance**, the `Admin` class above:

```js
function Admin(name) {
  User.call(this, name); // super(name): run the parent constructor on this object
}

Admin.prototype = Object.create(User.prototype); // extends: Admin.prototype ──► User.prototype
Admin.prototype.constructor = Admin; // restore the link overwritten above
Object.setPrototypeOf(Admin, User); // inherit the statics (Admin.count)

Admin.prototype.hello = function () {
  return User.prototype.hello.call(this) + " (admin)"; // super.hello()
};
```

`Object.create(User.prototype)` creates an empty object whose prototype is `User.prototype`: this builds the chain `bob ──► Admin.prototype ──► User.prototype`, which is what makes `instanceof User` true.

**Where `class` is more than syntax**:

|                            | Constructor function                                              | `class`                                |
| -------------------------- | ----------------------------------------------------------------- | -------------------------------------- |
| Called without `new`       | runs, `this` is the global object (or `undefined` in strict mode) | `TypeError`                            |
| Hoisting                   | the function is usable before its declaration                     | not usable before (temporal dead zone) |
| Methods                    | enumerable (show up in `for...in`)                                | non-enumerable                         |
| Strict mode                | opt-in (`"use strict"`)                                           | always                                 |
| Extending `Error`, `Array` | broken                                                            | works                                  |
