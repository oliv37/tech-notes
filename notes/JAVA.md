## Collection types

Four main types: `List`, `Set`, `Queue` / `Deque` and `Map`. But a `Map` is **not** a `Collection`: the `Map` interface doesn't extend `Collection`, it's only part of the Collections Framework (`java.util`).

```
Iterable
└── Collection
    ├── List          ordered, duplicates allowed, access by index
    ├── Set           no duplicates
    │   └── SortedSet / NavigableSet
    └── Queue         add at one end, remove at the other
        └── Deque     add and remove at both ends

Map                   key → value pairs, unique keys (outside the Collection hierarchy)
└── SortedMap / NavigableMap
```

Why `Map` isn't a `Collection`: a `Collection` holds single elements (`add(E)`, `contains(E)`, iterating over `E`), a `Map` holds pairs (`put(K, V)`, `get(K)`), so `add()` or iterating would be ambiguous (keys, values or pairs?). Instead, a `Map` exposes three views that are collections: `keySet()` (`Set<K>`), `values()` (`Collection<V>`) and `entrySet()` (`Set<Map.Entry<K, V>>`, to iterate over pairs).

| Type    | Implementation  | Specificity                                                    |
| ------- | --------------- | -------------------------------------------------------------- |
| `List`  | `ArrayList`     | resizable array, O(1) access by index, the default choice      |
|         | `LinkedList`    | linked list, rarely faster in practice                         |
| `Set`   | `HashSet`       | no order, O(1) (a `HashMap` under the hood)                    |
|         | `LinkedHashSet` | insertion order                                                |
|         | `TreeSet`       | sorted (natural order or `Comparator`), O(log n)               |
| `Queue` | `ArrayDeque`    | queue or stack, the default choice (instead of `Stack`)        |
|         | `PriorityQueue` | always removes the smallest element first (heap)               |
| `Map`   | `HashMap`       | no order, O(1), the default choice                             |
|         | `LinkedHashMap` | insertion order (or access order, for an LRU cache)            |
|         | `TreeMap`       | sorted keys, O(log n)                                          |

- Since Java 21, `SequencedCollection` / `SequencedMap` add `getFirst()`, `getLast()` and `reversed()` to collections with a defined order (`List`, `Deque`, `LinkedHashSet`, `TreeSet`, `LinkedHashMap`...).
- For concurrency (`java.util.concurrent`, since Java 5): `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue` (producer / consumer).

## equals and hashCode

By default (`Object`), `equals()` compares references (`==`) and `hashCode()` is derived from the object identity. Overriding them defines a logical equality (two objects with the same values are equal).

When to override: always together and on the same fields, whenever an object has a logical equality based on its data: value objects, map keys or set elements, objects compared or searched in collections (`contains()`, `indexOf()`, `remove()`) and in tests (`assertEquals`).

The contract between the two: if `a.equals(b)`, then `a.hashCode() == b.hashCode()`. The reverse is not required: two different objects can share the same hash (collision).

Why it matters: `HashMap` and `HashSet` first use `hashCode()` to find the bucket, then `equals()` to compare the entries within it. Overriding only `equals()` breaks them:

```java
class User {
  String email;

  User(String email) {
    this.email = email;
  }

  @Override
  public boolean equals(Object o) {
    return o instanceof User u && Objects.equals(email, u.email);
  }
  // no hashCode() override
}

Set<User> users = new HashSet<>();
users.add(new User("a@b.com"));
users.contains(new User("a@b.com")); // false: different hash, wrong bucket
```

Fix: always override both, on the same fields.

```java
@Override
public int hashCode() {
  return Objects.hash(email);
}
```

Pitfalls:

- Don't use mutable fields in `hashCode()`: if a field changes after the object is added to a `HashSet` / used as a `HashMap` key, the object is stored in the wrong bucket and can no longer be found.
- `record` classes generate `equals()` and `hashCode()` from all their components, and are immutable, which avoids both problems.

`HashSet` is a `HashMap` under the hood: the elements are the keys, and every value is the same shared dummy object. Simplified from the JDK source:

```java
public class HashSet<E> {
  private static final Object PRESENT = new Object();
  private final HashMap<E, Object> map = new HashMap<>();

  public boolean add(E e) {
    return map.put(e, PRESENT) == null; // null: the key wasn't there yet
  }

  public boolean contains(Object o) {
    return map.containsKey(o);
  }

  public boolean remove(Object o) {
    return map.remove(o) == PRESENT;
  }
}
```

Hence the same rules for set elements as for map keys (`hashCode()` / `equals()`, no mutation). Same idea for `LinkedHashSet` (backed by a `LinkedHashMap`) and `TreeSet` (backed by a `TreeMap`).

## Interfaces: default, static and private methods

Since Java 8 and 9, an interface can contain more than abstract methods:

```java
interface PriceCalculator {
  double basePrice(); // abstract: must be implemented

  default double priceWithTax() { // Java 8: inherited, can be overridden
    return round(basePrice() * 1.2);
  }

  static PriceCalculator of(double price) { // Java 8: called on the interface
    return () -> price;
  }

  private double round(double value) { // Java 9: internal helper
    return Math.round(value * 100) / 100.0;
  }
}
```

- **`default`**: part of the public API, inherited by all implementations, which can override it. Main purpose: evolving an interface without breaking existing implementations (this is how `Collection.stream()` or `List.sort()` were added in Java 8).
- **`static`**: utility or factory methods tied to the interface (`List.of()`, `Comparator.comparing()`), not inherited and called as `PriceCalculator.of(...)`.
- **`private`**: shares code between `default` (or `static`) methods without exposing it. Not visible to implementations, so not part of the contract.

`default` vs `private`: use `default` for behavior that implementations should get (and may customize), `private` for internal logic that only serves other methods of the interface.

Interface vs abstract class: an interface still can't hold state (only constants) or constructors, and a class can implement several of them. Prefer an interface to define a capability; use an abstract class only when implementations share state (fields) or need a common constructor.

## Multiple inheritance

Java forbids extending several classes, to avoid the diamond problem: if `B` and `C` both extend `A` and override the same method, a class `D` extending both wouldn't know which version to use, nor how to share `A`'s fields.

Two alternatives:

- **Several interfaces**: a class can implement as many as needed. Since Java 8, interfaces can carry behavior (`default` methods), but never state (no instance fields). If two interfaces provide the same `default` method, the compiler forces the class to override it and pick explicitly:

```java
class Duck implements Flyable, Swimmable {
  @Override
  public String move() {
    return Flyable.super.move() + " and " + Swimmable.super.move();
  }
}
```

- **Composition and delegation**: instead of inheriting code, hold an instance of the class and delegate to it ("favor composition over inheritance"). Looser coupling, and the delegate can be swapped or injected.

```java
class Duck implements Flyable, Swimmable {
  private final Wings wings = new Wings();

  public void fly() { wings.fly(); }
}
```

In practice: interfaces define what an object can do, composition brings code reuse.

## Exception types

```
Throwable
├── Error                   # JVM problems: OutOfMemoryError, StackOverflowError
└── Exception               # checked: IOException, SQLException
    └── RuntimeException    # unchecked: NullPointerException, IllegalArgumentException
```

- **`Error`**: serious problems the application can't recover from. Never catch them (except at the very top level, to log before crashing).
- **Checked exceptions** (`Exception` but not `RuntimeException`): the compiler forces the caller to catch them or declare them with `throws`. Meant for expected failures outside the program's control, that the caller can reasonably handle: file not found, network unavailable.
- **Unchecked exceptions** (`RuntimeException`): no compiler obligation. Used for programming errors and violated preconditions: `IllegalArgumentException` (invalid argument), `IllegalStateException` (method called at the wrong time), `NullPointerException`.

When to use which:

- Use a checked exception only if the caller can actually do something about it (retry, fallback, ask the user). Otherwise it just forces every layer to rethrow or wrap it.
- The modern trend favors unchecked exceptions: they don't pollute method signatures and work with lambdas and streams (a `Function` can't throw a checked exception). Spring follows this, wrapping `SQLException` into the unchecked `DataAccessException`.
- Business exceptions (`OrderNotFoundException`, `InsufficientBalanceException`) usually extend `RuntimeException` and are translated at the edge (e.g. into an HTTP status with `@ControllerAdvice`).

Good practices:

- Never swallow an exception (empty `catch`), and keep the cause when wrapping: `throw new OrderException("...", e)`.
- Use try-with-resources to close resources (`AutoCloseable`), even when an exception is thrown.
- Spring `@Transactional` only rolls back on unchecked exceptions by default; a checked exception commits the transaction unless `rollbackFor` is set.

## ClassNotFoundException vs NoClassDefFoundError

- **`ClassNotFoundException`** (checked): a class loaded by name (`Class.forName("...")`, `loadClass()`) isn't on the classpath. Can be caught, e.g. to fall back when an optional library is missing.
- **`NoClassDefFoundError`** (`Error`): a class referenced directly in the code, present at compile time, can't be used at runtime. Either missing at runtime (dependency not deployed, version conflict), or its static initialization failed earlier: first an `ExceptionInInitializerError`, then `Could not initialize class X` on every later access.

## map vs flatMap

Both transform the elements of a `Stream` (or the value of an `Optional`), the difference is what the function returns:

- **`map`**: the function returns a value, one element in → one element out. `Stream<T>` → `Stream<R>`.
- **`flatMap`**: the function returns a `Stream` (0, 1 or n elements), and the resulting streams are flattened into a single one. Avoids ending up with a `Stream<Stream<R>>`.

```java
List<Author> authors = ...;

authors.stream().map(Author::getName);   // Stream<String>: one name per author
authors.stream().map(Author::getBooks);  // Stream<List<Book>>: a list per author
authors.stream()
    .flatMap(a -> a.getBooks().stream()); // Stream<Book>: all books, flattened
```

Same idea with `Optional`, when the function itself returns an `Optional`:

```java
Optional<User> user = findUser(id);

user.map(User::getName);       // Optional<String>
user.map(User::getAddress);    // Optional<Optional<Address>> if getAddress() returns an Optional
user.flatMap(User::getAddress); // Optional<Address>
```

Rule of thumb: if the function already returns a container (`Stream`, `Optional`), use `flatMap` to avoid nesting it.

## Switch expressions, records and sealed classes

Three features (Java 14 to 21) designed to work together for data-oriented code: records carry the data, sealed types close the hierarchy, switch handles every case.

**Switch expressions** (Java 14): `switch` returns a value, with `->` instead of `case X:`.

```java
int days = switch (month) {
  case FEBRUARY -> 28;
  case APRIL, JUNE, SEPTEMBER, NOVEMBER -> 30; // several labels
  default -> {
    log.debug("31 days");
    yield 31; // yield returns the value of a block
  }
};
```

- No fall-through: only the matching branch runs, no `break` needed.
- Exhaustive: as an expression, every case must be handled. On an enum listing all constants, `default` isn't needed, and adding a constant breaks compilation where it's not handled.
- Selector types: `int`, `short`, `byte`, `char` (and their wrappers), `String`, enums, with constant labels. Since Java 21, any object type with patterns (see pattern matching below). `long`, `float`, `double` and `boolean` are not accepted (still in preview in Java 25).

**Records** (Java 16): immutable data classes. The compiler generates the `private final` fields, the canonical constructor, the accessors (`x()`, not `getX()`), `equals` / `hashCode` / `toString`.

```java
public record Point(int x, int y) {}
```

- Implicitly `final`, can't extend a class (they extend `Record`), but can implement interfaces and add methods or static fields. No other instance fields.
- Shallow immutability: a `List` component can still be modified, hence a defensive copy (`List.copyOf()`).
- Use cases: DTOs, value objects, query projections. Not JPA entities, which must be mutable, non-final (for proxies) and have a no-arg constructor.

A **compact constructor** customizes the canonical one without repeating the parameters: it validates or normalizes them, and the fields are assigned automatically at the end.

```java
public record Email(String value) {
  public Email {
    Objects.requireNonNull(value);
    value = value.trim().toLowerCase(); // reassigns the parameter, not the field
    if (!value.contains("@")) {
      throw new IllegalArgumentException("Invalid email: " + value);
    }
  } // this.value = value is done here
}
```

Assigning `this.value` inside it doesn't compile: only the parameter can be changed.

**Sealed classes** (Java 17): a class or interface lists which classes can extend it.

```java
public sealed interface PaymentResult permits Success, Failure {}
public record Success(String transactionId) implements PaymentResult {}
public record Failure(String reason) implements PaymentResult {}
```

- Each subclass must be `final` (records are), `sealed` (closed again) or `non-sealed` (open again to any subclass).
- Subclasses must be in the same module (or the same package without modules). `permits` can be omitted if they're in the same file.
- vs enum: each variant can carry different data (`transactionId` vs `reason`).

Main benefit: a `switch` on a sealed type is exhaustive without `default`. Adding a new variant (`Pending`) breaks compilation in every `switch` that doesn't handle it, instead of silently going to `default`.

```java
String message = switch (result) {
  case Success s -> "Paid: " + s.transactionId();
  case Failure(var reason) -> "Failed: " + reason; // record pattern: destructures the record
};
```

## Pattern matching

A pattern tests the shape of a value (its type, its components) and extracts parts of it in a single step, replacing the `instanceof` + cast combo. Usable in two places only: `instanceof` (in an `if` or any boolean expression) and `case` labels of a `switch`. Not in assignments or loops (no destructuring like `var (x, y) = point`).

**Type pattern with `instanceof`** (Java 16): tests the type and declares a variable already cast.

```java
// before
if (obj instanceof String) {
  String s = (String) obj;
}

// after
if (obj instanceof String s && s.length() > 5) { // s usable right after the test
  s.toUpperCase();
}

if (!(obj instanceof String s)) {
  return;
}
s.length(); // still in scope: the compiler knows obj is a String here
```

**Type pattern in `switch`** (Java 21): the selector can be any object type, and each `case` tests a type instead of comparing a constant. No sealed type needed:

```java
String describe(Object obj) {
  return switch (obj) {
    case Integer i when i > 100 -> "big number"; // guard
    case Integer i -> "number " + i;
    case String s -> "text of length " + s.length();
    case null -> "null"; // otherwise a null throws a NullPointerException
    default -> "something else";
  };
}
```

- **Exhaustive**: a pattern `switch` must handle every case, even as a statement. On `Object`, `default` is mandatory; on a sealed type or an enum, it can be omitted if all subtypes or constants are listed (see sealed classes above).
- **Order matters**: the first matching `case` runs. A `case` that can never be reached doesn't compile (`case CharSequence cs` followed by `case String s`), hence the guarded `case` before the general one.

**Record patterns** (Java 21): deconstruct a record into its components, possibly nested, in `instanceof` and `switch`.

```java
record Point(int x, int y) {}
record Line(Point start, Point end) {}

if (obj instanceof Point(int x, int y)) {
  System.out.println(x + y);
}

switch (shape) {
  case Line(Point(var x1, var y1), Point end) -> ... // nested, var infers the type
  case Point(var x, _) -> ... // _ (Java 22): ignored component
}
```

**Primitive patterns** (preview in Java 25): `case int i when i >= 500 ->`, `switch` on `long` / `double` / `boolean`, and `value instanceof byte b` (matches if the `int` fits in a `byte` without loss). Constant `case` labels on `int`, `char`, etc. have always worked; only primitive patterns are new.

Typical use: replacing chains of `instanceof` + cast, or the visitor pattern, to process a closed hierarchy (sealed interface + records).

## InterruptedException

Java has no way to forcibly stop a thread: interruption is a **cooperative** cancellation request. Calling `thread.interrupt()` sets an interrupt flag on the target thread; blocking methods (`Thread.sleep()`, `Object.wait()`, `Thread.join()`, `BlockingQueue.take()`, `Future.get()`) react by throwing `InterruptedException`, and **clear the flag** at the same time. This is how `ExecutorService.shutdownNow()` or `Future.cancel(true)` stop running tasks.

It's a checked exception, and the classic mistake is to swallow it:

```java
try {
  Thread.sleep(1000);
} catch (InterruptedException e) {
  // ❌ the flag is cleared: the cancellation request is lost, the thread keeps running
}
```

Two correct options:

- **Propagate it**: declare `throws InterruptedException` and let the caller decide.
- **Restore the flag** when propagating isn't possible (e.g. in a `Runnable`), then stop the current work:

```java
try {
  Thread.sleep(1000);
} catch (InterruptedException e) {
  Thread.currentThread().interrupt(); // ✅ restore the flag for the code above
  return; // or throw an unchecked exception, keeping e as cause
}
```

Code that doesn't block (e.g. a long computation loop) must check the flag itself: `while (!Thread.currentThread().isInterrupted()) { ... }`.

Watch out for `catch (Exception e)`: it also catches `InterruptedException`, and silently loses the interruption.

## Thread states

A thread is always in one of the 6 states of the `Thread.State` enum (visible with `thread.getState()` or in a thread dump):

```
NEW ──start()──> RUNNABLE ──────────────────────────> TERMINATED
                  │    ▲
                  ├──> BLOCKED         (waiting for a synchronized lock)
                  ├──> WAITING         (wait(), join(), park())
                  └──> TIMED_WAITING   (sleep(ms), wait(ms), join(ms))
```

- **`NEW`**: created but `start()` not called yet.
- **`RUNNABLE`**: running, or ready to run and waiting for a CPU. The JVM doesn't distinguish the two. A thread blocked on I/O (reading a socket) is also reported as `RUNNABLE`.
- **`BLOCKED`**: waiting to acquire a monitor lock to enter a `synchronized` block held by another thread.
- **`WAITING`**: waiting indefinitely for another thread to act: `Object.wait()` (until `notify()`), `Thread.join()` (until the other thread ends), `LockSupport.park()`.
- **`TIMED_WAITING`**: same, but with a timeout: `Thread.sleep(ms)`, `wait(ms)`, `join(ms)`.
- **`TERMINATED`**: `run()` has finished, normally or with an exception. A thread can't be restarted: calling `start()` again throws `IllegalThreadStateException`.

Pitfalls:

- A thread waiting for a `ReentrantLock` is `WAITING` (it uses `park()` internally), not `BLOCKED`, which is reserved for `synchronized`.
- In a thread dump, many threads `BLOCKED` on the same lock point to contention; two threads each `BLOCKED` on the lock held by the other is a deadlock (detected and reported by `jstack`).

## JVM diagnostic tools

Command-line tools shipped with the JDK (not the JRE), to inspect a running JVM by its PID:

| Command                                              | Purpose                                                                                                 |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `jps -l`                                             | List running JVMs with their PID and main class                                                         |
| `jstack <pid>`                                       | Thread dump: state and stack trace of every thread, detected deadlocks                                  |
| `jcmd <pid> Thread.print`                            | Same thread dump, through `jcmd`                                                                        |
| `jcmd <pid> GC.heap_info`                            | Heap usage per generation                                                                               |
| `jcmd <pid> GC.class_histogram`                      | Number of instances and memory used per class (spot what fills the heap)                                |
| `jcmd <pid> GC.heap_dump /tmp/heap.hprof`            | Full heap dump, to analyze with Eclipse MAT or VisualVM                                                 |
| `jcmd <pid> VM.flags` / `VM.system_properties`       | JVM options and system properties actually in use                                                       |
| `jcmd <pid> JFR.start duration=60s filename=rec.jfr` | Java Flight Recorder: low-overhead profiling (CPU, allocations, locks), opened with JDK Mission Control |
| `jstat -gcutil <pid> 1000`                           | GC statistics refreshed every second (occupancy, GC count and time)                                     |
| `kill -3 <pid>`                                      | Thread dump written to the application's standard output, when no JDK tool is available                 |

`jcmd` is the modern all-in-one tool and covers most of what `jstack`, `jmap` and `jinfo` do (`jcmd <pid> help` lists the available commands).

Useful in practice:

- Take several thread dumps a few seconds apart: a thread stuck on the same line in all of them reveals a blocking point or an infinite loop.
- Start production JVMs with `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps` to get a heap dump automatically when memory runs out.
- The tools must run as the same user as the JVM. In a container, run them inside it (`docker exec <container> jcmd 1 Thread.print`), which requires a JDK-based image.

## Virtual threads

Since Java 21. A classic (platform) thread is mapped 1:1 to an OS thread: expensive (~1 MB of stack, slow to create), so applications are limited to a few hundred or thousand of them, managed in pools. A **virtual thread** is managed by the JVM: it costs a few KB and millions can run at the same time.

How it works: virtual threads run on a small pool of platform threads called **carrier threads** (about one per CPU core). When a virtual thread blocks (I/O, `sleep()`, waiting for a lock), the JVM unmounts it from its carrier, which immediately runs another virtual thread. Blocking no longer wastes an OS thread.

|               | Platform thread                    | Virtual thread                                     |
| ------------- | ---------------------------------- | -------------------------------------------------- |
| Managed by    | the OS (1:1 with an OS thread)     | the JVM (runs on a carrier thread)                 |
| Cost          | ~1 MB of stack, slow to create     | a few KB, cheap to create                          |
| How many      | hundreds to thousands              | millions                                           |
| When blocking | the OS thread waits, doing nothing | unmounted, the carrier runs another virtual thread |
| Usage         | pooled (`ThreadPoolExecutor`)      | one per task, never pooled                         |
| Daemon        | configurable                       | always                                             |

Both are `java.lang.Thread`: same API, the code doesn't change.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
  for (int i = 0; i < 10_000; i++) {
    executor.submit(() -> callRemoteApi()); // one virtual thread per task
  }
}

Thread.ofVirtual().start(() -> doWork());
```

In Spring Boot (3.2+): `spring.threads.virtual.enabled=true` makes Tomcat handle each request in a virtual thread.

When to use them: I/O-bound workloads with high concurrency (HTTP calls, database queries, a request waiting on other services). They keep the simple blocking "one thread per request" style while scaling like reactive code (WebFlux), without its complexity. They bring nothing for CPU-bound work: the number of cores stays the limit.

Pitfalls:

- **Don't pool them**: they're cheap, create one per task. To limit concurrency (e.g. calls to a fragile API), use a `Semaphore`.
- **Pinning**: before Java 24, a virtual thread blocking inside a `synchronized` block can't be unmounted and keeps its carrier busy; prefer `ReentrantLock` on older versions. Native calls still pin.
- **Downstream limits remain**: 10,000 virtual threads still share the same database connection pool (HikariCP), which becomes the bottleneck.
- **`ThreadLocal`**: still works, but with millions of threads, heavy per-thread caches consume a lot of memory.

## ScopedValue

Final since Java 25. Shares contextual data (current user, request id) with every method called in a scope, without passing it as a parameter. Same purpose as `ThreadLocal`, designed for virtual threads.

```java
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// bind the value for the duration of run()
ScopedValue.where(CURRENT_USER, user).run(() -> handleRequest());

// anywhere down the call stack
void audit() {
  User user = CURRENT_USER.get(); // NoSuchElementException if not bound
}
```

- `run()` for a `Runnable`, `call()` to return a value. `isBound()` and `orElse()` handle the unbound case.
- A nested `where(...)` can bind another value for a sub-scope; the previous one comes back when it ends.

Differences with `ThreadLocal`:

- **Immutable**: no `set()`, the value can't be changed by a called method, which makes the data flow easy to follow.
- **Bounded lifetime**: the value is unbound automatically when `run()` returns. No leak from a forgotten `remove()` (a classic bug with thread pools, where the next task sees the previous value).
- **Cheap**: no per-thread map to maintain, and child threads created with structured concurrency (`StructuredTaskScope`, still in preview) share the value without copying it, unlike `InheritableThreadLocal`. Matters with millions of virtual threads.

`ThreadLocal` remains useful for a mutable per-thread cache (e.g. a non-thread-safe `SimpleDateFormat`), or when the value is set and read in unrelated places without a clear scope.

## Singleton

A singleton guarantees a single instance of a class, with a global access point. Common base: a `private` constructor and a static accessor. Several implementations, mainly differing in thread safety and lazy initialization:

**Eager initialization**: the instance is created when the class is loaded. Simple and thread-safe (class initialization is guaranteed by the JVM), but created even if never used.

```java
public class Config {
  private static final Config INSTANCE = new Config();
  private Config() {}
  public static Config getInstance() { return INSTANCE; }
}
```

**Lazy with `synchronized`**: created on first call, thread-safe, but every call pays the cost of the lock.

```java
public static synchronized Config getInstance() {
  if (instance == null) instance = new Config();
  return instance;
}
```

**Double-checked locking**: only locks while the instance doesn't exist. The field must be `volatile`, otherwise another thread can see a non-null reference to a partially constructed object.

```java
private static volatile Config instance;

public static Config getInstance() {
  if (instance == null) {
    synchronized (Config.class) {
      if (instance == null) instance = new Config();
    }
  }
  return instance;
}
```

**Holder class**: lazy and thread-safe without any lock. The static nested class is only initialized (and the instance created) on the first access to `Holder.INSTANCE`, i.e. the first call to `getInstance()`. The JVM guarantees a class is initialized only once, other threads waiting until it's done (JLS §12.4.2).

```java
public class Config {
  private Config() {}
  private static class Holder { static final Config INSTANCE = new Config(); }
  public static Config getInstance() { return Holder.INSTANCE; }
}
```

**Enum**: recommended by _Effective Java_. Thread-safe, and the only version protected against reflection (calling the private constructor) and serialization (deserializing creates a new instance, unless `readResolve()` is implemented).

```java
public enum Config {
  INSTANCE;
  public String get(String key) { ... }
}
```

In practice: a singleton is global state, hard to replace in tests. In a Spring application, beans are already singletons by default (one instance per container) and injected, which gives the same guarantee while keeping the code testable.

## Main evolutions since Java 8

Since Java 10, a new version every 6 months, and an LTS (long-term support) every 2 years since Java 17. Projects usually move from LTS to LTS: 8 → 11 → 17 → 21 → 25.

**Java 8 (2014, LTS)**: the big shift to functional style.

- Lambdas and method references (`Author::getName`), functional interfaces (`Function`, `Predicate`, `Supplier`)
- Stream API, `Optional`
- `default` and `static` methods in interfaces
- `java.time` (`LocalDate`, `Instant`, `Duration`), replacing `Date` and `Calendar`
- `CompletableFuture`

**Java 9 to 11 (LTS)**

- Modules (JPMS, `module-info.java`): strong encapsulation of the JDK, rarely used in applications
- Immutable collection factories: `List.of()`, `Set.of()`, `Map.of()`
- `var` for local variables (Java 10): `var authors = new ArrayList<Author>();`
- `HttpClient` (HTTP/2, sync and async), new `String` methods (`isBlank()`, `strip()`, `lines()`, `repeat()`)
- Running a single file without compiling: `java Main.java`
- Java EE modules (JAXB, JAX-WS) removed from the JDK

**Java 12 to 17 (LTS)**: less boilerplate, data-oriented code.

- Switch expressions: `case A, B -> ...`, the `switch` returns a value
- Text blocks: multi-line strings with `"""`
- Records: immutable data classes, with constructor, accessors, `equals` / `hashCode` / `toString` generated

```java
public record Point(int x, int y) {}
```

- Pattern matching for `instanceof`: `if (obj instanceof String s) { s.length(); }`, no cast needed
- Sealed classes: `sealed interface Shape permits Circle, Square`, restricts which classes can implement it
- Helpful `NullPointerException` messages, naming what was null (`because "user.getAddress()" is null`)

**Java 18 to 21 (LTS)**

- Virtual threads (see above)
- Pattern matching for `switch` and record patterns: combined with sealed types, the compiler checks every case is handled

```java
double area = switch (shape) {
  case Circle c -> Math.PI * c.radius() * c.radius();
  case Square(double side) -> side * side; // record pattern: destructures the record
};
```

- Sequenced collections: `getFirst()`, `getLast()`, `reversed()` on `List`, `LinkedHashSet`, `LinkedHashMap`
- UTF-8 by default (Java 18)

**Java 22 to 25 (LTS)**

- Unnamed variables `_`: `catch (Exception _)`, for values that aren't used
- Stream gatherers (`stream.gather(...)`): custom intermediate operations (sliding windows, etc.)
- No more pinning of virtual threads in `synchronized` (Java 24)
- Compact source files and instance `main` (Java 25): `void main() { IO.println("Hello"); }`, no class declaration needed
- Flexible constructor bodies: code allowed before `super(...)` (e.g. validating arguments)
- Scoped values: an immutable, cheaper alternative to `ThreadLocal`, suited to virtual threads
