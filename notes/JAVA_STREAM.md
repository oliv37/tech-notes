## What a stream is

The Stream API (Java 8, `java.util.stream`) processes a sequence of elements declaratively: you describe **what** to compute (filter, transform, aggregate), not the loop that does it.

```java
List<String> names = authors.stream()          // source
    .filter(a -> a.getBooks().size() > 2)       // intermediate operation
    .map(Author::getName)                       // intermediate operation
    .sorted()                                   // intermediate operation
    .toList();                                  // terminal operation
```

A stream is a **pipeline**: a source, zero or more intermediate operations, one terminal operation.

| | Collection | Stream |
|---|---|---|
| Role | stores elements | describes a computation on elements |
| Evaluation | elements exist in memory | **lazy**: nothing runs before the terminal operation |
| Use | read many times | **single use**: a second terminal operation throws `IllegalStateException` |
| Source | modified by `add`, `remove` | never modified: each operation returns a new stream |
| Size | finite | can be infinite (`Stream.iterate`, `generate`) with `limit` |
| Iteration | external (`for` loop written by the code) | internal (the stream drives the loop) |

## How it works: laziness and element-by-element processing

Intermediate operations only **build** the pipeline. The terminal operation starts it, and then each element goes through **all** the operations one after the other, before the next element (vertical processing, not one full pass per operation):

```java
Stream.of("a", "bb", "ccc", "dddd")
    .filter(s -> { System.out.println("filter " + s); return s.length() > 1; })
    .map(s -> { System.out.println("map " + s); return s.toUpperCase(); })
    .findFirst();

// filter a
// filter bb
// map bb       → found: "ccc" and "dddd" are never processed
```

- **Short-circuit**: `findFirst`, `anyMatch`, `limit` stop the pipeline as soon as the result is known. This is what makes infinite streams usable: `Stream.iterate(1, n -> n * 2).limit(10)`.
- **Without a terminal operation, nothing runs**: `list.stream().map(this::send);` sends nothing.
- **Stateless** operations (`filter`, `map`) handle each element alone and pass it on immediately. **Stateful** ones (`sorted`, `distinct`, `limit`) keep state about other elements; `sorted()` is a barrier, it reads the whole stream before passing anything on:

```java
Stream.of("c", "a", "b")
    .peek(s -> System.out.println("before " + s))
    .sorted()
    .peek(s -> System.out.println("after " + s))
    .findFirst();

// before c
// before a
// before b   ← sorted reads everything first
// after a    ← then findFirst stops at the first one
```

  A stateful operation breaks the element-by-element flow (memory), doesn't work on an infinite stream (`sorted`), costs more in parallel (the state of each thread must be merged), and its position matters: `filter(...).sorted()` sorts fewer elements than `sorted().filter(...)`.
- Internally, a source is a `Spliterator` (iterates and can split itself for parallel streams), and the operations are chained `Sink`s.

## Creating a stream

```java
list.stream()                          // from a collection
Arrays.stream(array)                   // from an array
Stream.of("a", "b", "c")               // from values
Stream.ofNullable(value)               // 0 or 1 element (Java 9)
IntStream.range(0, 10)                 // 0..9 (rangeClosed: 0..10)
Stream.iterate(1, n -> n * 2)          // infinite: 1, 2, 4, 8...
Stream.iterate(1, n -> n < 100, n -> n * 2) // with a stop condition (Java 9)
Stream.generate(UUID::randomUUID)      // infinite, from a Supplier
Files.lines(path)                      // lines of a file (to close: try-with-resources)
map.entrySet().stream()                // a Map isn't a Collection: go through entrySet / keySet / values
"text".chars()                         // IntStream of characters
```

## Intermediate operations

Two kinds: **stateless** operations handle each element alone and pass it on immediately; **stateful** ones keep state about other elements (see above).

| Operation | Kind | Role |
|---|---|---|
| `filter(predicate)` | stateless | keeps the matching elements |
| `map(function)` | stateless | transforms each element (1 → 1) |
| `flatMap(function)` | stateless | each element → a stream, flattened (1 → n) (see [map vs flatMap](JAVA.md)) |
| `mapMulti(...)` (Java 16) | stateless | 1 → n without creating a stream per element |
| `mapToInt`, `mapToLong`, `mapToDouble` | stateless | to a primitive stream (see below) |
| `peek(action)` | stateless | runs an action on each element without changing it, for debugging |
| `distinct()` | stateful: a `Set` of the elements seen | removes duplicates (`equals` / `hashCode`), elements still flow one by one |
| `sorted()`, `sorted(comparator)` | stateful: **all** the elements | sorts (natural order, or `Comparator.comparing(Author::getName).reversed()`); a barrier |
| `limit(n)`, `skip(n)` | stateful: a counter | keeps the first n / skips the first n; `limit` short-circuits |
| `takeWhile(predicate)`, `dropWhile(predicate)` (Java 9) | stateful: a flag | takes / skips elements while the condition holds; `takeWhile` short-circuits |
| `gather(gatherer)` (Java 24) | either | custom intermediate operation (see below) |

## Terminal operations

| Operation | Result |
|---|---|
| `toList()` (Java 16) | unmodifiable `List` |
| `collect(collector)` | any structure (see Collectors below) |
| `forEach(action)` | runs an action on each element |
| `count()` | `long` |
| `min(comparator)`, `max(comparator)` | `Optional<T>` |
| `findFirst()`, `findAny()` | `Optional<T>` (`findAny` is faster in parallel, any element) |
| `anyMatch`, `allMatch`, `noneMatch` | `boolean`, short-circuit |
| `reduce(identity, operator)` | combines the elements into one value: `reduce(0, Integer::sum)` |
| `toArray()` | array |

- `stream.toList()` returns an unmodifiable list; `collect(Collectors.toList())` returns an `ArrayList` in practice, but without guarantee.
- `Optional` results: `max(...).orElseThrow()`, `findFirst().map(Author::getName).orElse("unknown")`.

## Collectors

`collect()` with a `Collector` builds the result: collections, maps, groups, strings, statistics.

```java
// List, Set, Map
List<String> names = authors.stream().map(Author::getName).collect(Collectors.toList());
Set<String> countries = authors.stream().map(Author::getCountry).collect(Collectors.toSet());
Map<Long, Author> byId = authors.stream().collect(Collectors.toMap(Author::getId, a -> a));

// grouping: Map<country, List<Author>>
Map<String, List<Author>> byCountry = authors.stream()
    .collect(Collectors.groupingBy(Author::getCountry));

// grouping with a downstream collector: Map<country, number of authors>
Map<String, Long> countByCountry = authors.stream()
    .collect(Collectors.groupingBy(Author::getCountry, Collectors.counting()));

// Map<country, List<names>>
Map<String, List<String>> namesByCountry = authors.stream()
    .collect(Collectors.groupingBy(Author::getCountry,
        Collectors.mapping(Author::getName, Collectors.toList())));

// two groups: true / false
Map<Boolean, List<Author>> prolific = authors.stream()
    .collect(Collectors.partitioningBy(a -> a.getBooks().size() > 5));

// string
String csv = authors.stream().map(Author::getName).collect(Collectors.joining(", ", "[", "]"));

// statistics
IntSummaryStatistics stats = books.stream().collect(Collectors.summarizingInt(Book::getPages));
double average = books.stream().collect(Collectors.averagingInt(Book::getPages));
```

- **`toMap` pitfall**: a duplicate key throws `IllegalStateException`. Give a merge function: `toMap(Author::getCountry, Author::getName, (a, b) -> a + ", " + b)`. A `null` value throws a `NullPointerException`.
- `groupingBy` returns a `HashMap` (no order): `groupingBy(Author::getCountry, TreeMap::new, Collectors.toList())` for a sorted map.
- Other downstream collectors: `summingInt`, `averagingDouble`, `maxBy`, `filtering` and `flatMapping` (Java 9), `teeing` (Java 12, combines two collectors).

## Primitive streams

`IntStream`, `LongStream`, `DoubleStream` avoid boxing each value into an `Integer`, and provide numeric operations:

```java
int totalPages = books.stream().mapToInt(Book::getPages).sum();
OptionalDouble average = books.stream().mapToInt(Book::getPages).average();
IntStream.rangeClosed(1, 5).map(i -> i * i).boxed().toList(); // [1, 4, 9, 16, 25]
IntStream.range(0, list.size()).mapToObj(i -> i + ": " + list.get(i)); // with an index
```

- `sum()`, `average()`, `min()`, `max()`, `summaryStatistics()` exist only on primitive streams.
- `boxed()` / `mapToObj()` go back to a `Stream<Integer>` / `Stream<T>`.

## Parallel streams

`parallelStream()` or `.parallel()` splits the source (`Spliterator`) and processes the parts on several threads of the common **`ForkJoinPool`**, then combines the results.

```java
long count = hugeList.parallelStream().filter(this::isValid).count();
```

Rarely worth it in an application:

- Useful only for CPU-heavy work on a large amount of data, with a source that splits well (`ArrayList`, arrays, `IntStream.range`; not `LinkedList`, `Stream.iterate`, I/O).
- The common pool is shared by the whole JVM (and has as many threads as cores minus one): a blocking call (database, HTTP) in a parallel stream blocks it for everyone. For I/O, use an executor or virtual threads (see [Java](JAVA.md)).
- Order: `forEach` doesn't respect it in parallel (`forEachOrdered` does); `findFirst`, `limit`, `sorted` keep it but cost more.
- Side effects on a shared structure (`list.add` from a lambda) are race conditions: use `collect`.
- Measure before: thread coordination often makes it slower on small collections.

## Pitfalls and good practices

- **Side effects**: lambdas should not modify external state. `forEach(list::add)` instead of `collect(toList())` is wrong (and unsafe in parallel).
- **Reuse**: a stream stored in a variable can only be consumed once; store the collection, or a `Supplier<Stream<T>>`.
- **Checked exceptions**: a `Function` can't throw an `IOException`; wrap it into an unchecked exception (`UncheckedIOException`), or use a loop.
- **Readability**: a long pipeline with nested lambdas is harder to read than a loop. Extract methods (`.filter(this::isEligible)`), and keep a `for` loop when it's clearer (early `return`, several accumulators, checked exceptions).
- **Closing**: streams on I/O resources (`Files.lines`, `Files.list`) hold a file handle: use try-with-resources. Collection streams don't need closing.
- **Performance**: a stream is slightly slower than a loop on simple cases, negligible in business code. What matters more: avoid `sorted` or `distinct` when not needed, filter early, and use primitive streams for numbers.

## Evolutions since Java 8

| Version | Addition |
|---|---|
| Java 9 | `takeWhile`, `dropWhile`, `iterate` with a condition, `ofNullable`; collectors `filtering`, `flatMapping` |
| Java 10 | `Collectors.toUnmodifiableList`, `toUnmodifiableSet`, `toUnmodifiableMap` |
| Java 12 | `Collectors.teeing` |
| Java 16 | `Stream.toList()`, `mapMulti` |
| Java 24 | **Stream gatherers** (`gather`, JEP 485): custom intermediate operations (see below) |

## Stream gatherers (Java 24)

A **gatherer** (JEP 485, final in Java 24) is a custom intermediate operation, passed to `stream.gather(...)`. It does for intermediate operations what a `Collector` does for terminal ones: before, an operation missing from the API (batches, sliding windows, distinct by key) meant leaving the stream, or a hack with `collect`.

**Built-in gatherers** (`java.util.stream.Gatherers`):

```java
Stream.of(1, 2, 3, 4, 5).gather(Gatherers.windowFixed(2)).toList();         // [[1, 2], [3, 4], [5]]
Stream.of(1, 2, 3, 4).gather(Gatherers.windowSliding(2)).toList();          // [[1, 2], [2, 3], [3, 4]]
Stream.of(1, 2, 3).gather(Gatherers.scan(() -> 0, Integer::sum)).toList();  // running sum: [1, 3, 6]
Stream.of(1, 2, 3).gather(Gatherers.fold(() -> 0, Integer::sum)).toList();  // single result: [6]
urls.stream().gather(Gatherers.mapConcurrent(10, this::download)).toList(); // see below
```

`mapConcurrent(n, function)` applies the function on virtual threads, at most `n` at a time, keeping the order: an alternative to parallel streams for I/O calls (HTTP, database).

**How it works**: a `Gatherer<T, A, R>` receives elements of type `T`, keeps an optional state of type `A`, and pushes elements of type `R` downstream (0, 1 or n per input element). It is made of up to 4 functions:

| Function | Type | Role |
|---|---|---|
| **initializer** | `Supplier<A>` | creates the state (a list, a counter, a set), once per stream |
| **integrator** | `Integrator<A, T, R>` | called for each element: `integrate(state, element, downstream)`, updates the state, pushes 0..n elements with `downstream.push(r)`, returns `false` to stop the stream (short-circuit) |
| **finisher** | `BiConsumer<A, Downstream<R>>` | called at the end of the input, to push what remains in the state |
| **combiner** | `BinaryOperator<A>` | merges two states, for parallel streams only |

- `downstream.push(r)` returns `false` when the rest of the pipeline wants no more elements (e.g. a `limit` or a `findFirst` further down): the integrator should then return `false` too.
- `Integrator.ofGreedy(...)`: an integrator that never stops the stream by itself, which allows some optimizations.
- Factory methods: `Gatherer.of(integrator)` (stateless), `Gatherer.ofSequential(initializer, integrator[, finisher])` (stateful, run sequentially), `Gatherer.of(initializer, integrator, combiner, finisher)` (parallelizable).
- Composition: `gatherer1.andThen(gatherer2)`, and `stream.gather(a).gather(b)`.

**Creating one: batches** of `size` elements, with a state and a finisher:

```java
static <T> Gatherer<T, ?, List<T>> batch(int size) {
  return Gatherer.ofSequential(
      () -> new ArrayList<T>(size),                                 // initializer: the current batch
      Gatherer.Integrator.ofGreedy((batch, element, downstream) -> { // integrator
        batch.add(element);
        if (batch.size() < size) {
          return true;                                             // nothing to push yet
        }
        List<T> full = List.copyOf(batch);
        batch.clear();
        return downstream.push(full);
      }),
      (batch, downstream) -> {                                     // finisher: the last, incomplete batch
        if (!batch.isEmpty()) {
          downstream.push(List.copyOf(batch));
        }
      });
}

orders.stream()
    .gather(batch(500))
    .forEach(orderRepository::saveAll); // inserts by batches of 500
```

**Distinct by key**: a state, no finisher:

```java
static <T, K> Gatherer<T, ?, T> distinctBy(Function<? super T, K> keyExtractor) {
  return Gatherer.ofSequential(
      HashSet<K>::new,
      Gatherer.Integrator.ofGreedy((seen, element, downstream) ->
          !seen.add(keyExtractor.apply(element)) || downstream.push(element)));
}

authors.stream().gather(distinctBy(Author::getCountry)).toList(); // first author of each country
```

**Short-circuit, stateless**: takes elements up to and including the first one matching a condition (`takeWhile` excludes it):

```java
static <T> Gatherer<T, ?, T> takeUntil(Predicate<? super T> stop) {
  return Gatherer.of((_, element, downstream) ->
      downstream.push(element) && !stop.test(element)); // false: stops the stream
}

Stream.of(1, 3, 8, 2, 5).gather(takeUntil(n -> n > 5)).toList(); // [1, 3, 8]
```

- No state: `Gatherer.of(integrator)`, the state type is `Void` (here the unnamed variable `_`, Java 22).
- Returning `false` stops reading the source: works on infinite streams.

**Short-circuit with a state**: a non-greedy integrator (`Gatherer.Integrator.of(...)`, or a plain lambda) returns `false` to stop the stream:

```java
// takes books while the total number of pages stays within maxPages
static Gatherer<Book, ?, Book> takeWithinPages(int maxPages) {
  return Gatherer.ofSequential(
      () -> new int[1],                                  // state: the running total (array: mutable)
      Gatherer.Integrator.of((total, book, downstream) -> {
        total[0] += book.getPages();
        if (total[0] > maxPages) {
          return false;                                  // stops the stream: the source isn't read anymore
        }
        return downstream.push(book);                    // false too if the rest of the pipeline wants no more
      }));
}

books.stream().gather(takeWithinPages(1000)).toList();
```

| | `Integrator.of(...)` / lambda | `Integrator.ofGreedy(...)` |
|---|---|---|
| Can stop the stream by itself | yes, returns `false` | no: processes the whole input |
| Return value | `false` to stop, `true` to continue | `true`, or the result of `downstream.push()` |
| Use | short-circuit (take while, limit, search) | the JDK knows the stream won't be interrupted: optimizations |

- After a `false`: the integrator isn't called anymore, the source isn't read anymore (works on infinite streams), but the **finisher is still called**.
- A greedy integrator returning `false` on its own breaks its contract. Returning the result of `downstream.push()` is fine in both cases: the stop then comes from the rest of the pipeline.

**In parallel**: a gatherer built with `ofSequential` runs on a single thread, even in a parallel stream. To be parallelizable, it needs a combiner, and its result must not depend on the order of the merges (a sum, a count), which is rare for windows or batches.
