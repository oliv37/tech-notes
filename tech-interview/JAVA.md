## equals and hashCode

By default (`Object`), `equals()` compares references (`==`) and `hashCode()` is derived from the object identity. Overriding them defines a logical equality (two objects with the same values are equal).

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

**Holder class**: lazy and thread-safe without any lock. The nested class is only loaded (and the instance created) on the first call to `getInstance()`.

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
