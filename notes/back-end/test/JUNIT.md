## JUnit: what it is and how it works

> **In short**: JUnit is Java's standard testing framework: `@Test` methods, assertions, a new test class instance per test. The JUnit Platform runs the tests for the tools (IDE, Maven, Gradle) and delegates to engines, Jupiter being the engine of JUnit 5 / 6.

JUnit is the standard testing framework for Java: tests are methods annotated with `@Test`, run by JUnit, which reports which ones pass or fail. A test fails when an assertion fails or an exception is thrown. Current versions: JUnit 5 (2017) introduced the **Jupiter** API (`org.junit.jupiter.api`); JUnit 6 (2025) keeps the same API, with Java 17 as minimum.

```java
class PriceCalculatorTest {

  private final PriceCalculator calculator = new PriceCalculator();

  @Test
  void appliesTaxToBasePrice() {
    // given
    Product product = new Product("Book", 10.0);
    // when
    double price = calculator.priceWithTax(product);
    // then
    assertEquals(12.0, price);
  }

  @Test
  void rejectsNegativePrice() {
    var exception = assertThrows(IllegalArgumentException.class,
        () -> calculator.priceWithTax(new Product("Book", -1)));
    assertEquals("Price must be positive", exception.getMessage());
  }
}
```

- Architecture: the **JUnit Platform** launches tests for the tools (IDE, Maven, Gradle) and delegates to **engines**, Jupiter for JUnit 5/6 tests (see Platform and engines below).
- **A new instance of the test class per test method**: fields are reset between tests, so tests stay independent. Test classes and methods don't need to be `public`.
- Structure each test in three steps: given (arrange), when (act), then (assert).
- Main assertions: `assertEquals`, `assertTrue` / `assertFalse`, `assertNull`, `assertThrows` (returns the exception, to check its message), `assertAll` (runs every assertion and reports all failures, not only the first). AssertJ (`assertThat(price).isEqualTo(12.0)`) is often preferred for readability, and comes with `spring-boot-starter-test`.

Typical use cases: unit tests of business logic (a service with mocked dependencies, via Mockito), checking that invalid input throws the right exception, regression tests reproducing a fixed bug, integration tests with Spring (`@SpringBootTest`, `@DataJpaTest`), which are JUnit tests extended by Spring.

## Lifecycle and organization

> **In short**: `@BeforeEach` / `@AfterEach` run around each test, `@BeforeAll` / `@AfterAll` once per class (`static`, since each test gets a new instance); `@DisplayName`, `@Nested`, `@Tag` and `@Disabled` organize the tests.

```java
class OrderServiceTest {

  @BeforeAll
  static void startOnce() { ... }       // once before all tests (static)

  @BeforeEach
  void setUp() { ... }                  // before each test: fresh test data

  @AfterEach
  void tearDown() { ... }               // after each test: cleanup

  @AfterAll
  static void stopOnce() { ... }        // once after all tests

  @Test
  @DisplayName("an empty cart can't be ordered")
  void emptyCart() { ... }

  @Nested
  class WhenOrderIsPaid {               // groups tests sharing a context
    @BeforeEach
    void pay() { ... }                  // runs after the outer @BeforeEach

    @Test
    void canBeShipped() { ... }
  }
}
```

- `@BeforeAll` / `@AfterAll` are `static`, since there's a new instance per test. `@TestInstance(Lifecycle.PER_CLASS)` uses a single instance for the whole class, making them non-static (and fields shared between tests).
- `@Disabled("reason")` skips a test, `@Tag("slow")` labels it to include or exclude it in the build (e.g. run only fast tests locally).
- Test order isn't guaranteed (deterministic but not obvious), by design: tests must not depend on each other. `@TestMethodOrder` exists for rare cases.

## Parameterized tests

> **In short**: `@ParameterizedTest` runs the same test with several inputs, each reported separately: `@ValueSource`, `@CsvSource`, `@EnumSource`, `@MethodSource`.

The same test run with several inputs, each reported as a separate test:

```java
@ParameterizedTest
@CsvSource({
    "10.0, 12.0",
    "0.0,  0.0",
    "99.9, 119.88"
})
void appliesTax(double basePrice, double expected) {
  assertEquals(expected, calculator.priceWithTax(basePrice), 0.001);
}

@ParameterizedTest
@ValueSource(strings = {"", " ", "not-an-email"})
void rejectsInvalidEmails(String input) {
  assertThrows(IllegalArgumentException.class, () -> new Email(input));
}

@ParameterizedTest
@MethodSource("orders") // a static method returning a Stream of arguments
void computesTotal(Order order, double expectedTotal) { ... }

static Stream<Arguments> orders() {
  return Stream.of(
      Arguments.of(new Order(List.of(line(10, 2))), 20.0),
      Arguments.of(new Order(List.of()), 0.0));
}
```

Other sources: `@EnumSource` (each value of an enum), `@NullAndEmptySource` (adds `null` and `""`), `@CsvFileSource` (from a file).

## JUnit Platform and engines

> **In short**: The Platform is a facade: tools only talk to its Launcher API, and each engine (Jupiter, Vintage, Spock, Cucumber) implements the TestEngine API to discover and run the tests written with its own programming model.

JUnit is split in two layers. The **Platform** is a facade between the tools and the tests: IDEs, Maven Surefire and Gradle only talk to its `Launcher`. An **engine** understands one programming model (its annotations, its rules) and runs the tests written with it: Jupiter (JUnit 5/6), Vintage (JUnit 4, deprecated in JUnit 6), or third-party engines (Spock, Cucumber, Kotest). Like JDBC: one API for the tools, one driver per framework.

The Platform itself has two sides:

- **Launcher API** (`junit-platform-launcher`), used by the tools: takes a request ("run this class"), asks the engines, builds the test tree, runs it and sends events to a listener.
- **TestEngine API** (`junit-platform-engine`), implemented by each engine: mainly `discover()` ("which tests do you find?") and `execute()` ("run them").

```java
// roughly what an IDE or Surefire does
LauncherDiscoveryRequest request = LauncherDiscoveryRequestBuilder.request()
    .selectors(selectClass(OrderServiceTest.class))
    .build();

Launcher launcher = LauncherFactory.create();
launcher.execute(request, new SummaryGeneratingListener()); // the listener gets the results
```

What happens when clicking "Run test" in the IDE:

1. The IDE starts a JVM and calls the Platform's `Launcher` with a selector (a class, a method).
2. The Platform finds the engines on the classpath (each jar declares its engine in `META-INF/services/org.junit.platform.engine.TestEngine`).
3. **Discovery**: the Platform asks **every** engine which tests it finds. It doesn't choose an engine: each one only returns what it recognizes, and an engine finding nothing returns an empty result. The result is a tree of tests, each with a unique id prefixed by its engine: `[engine:junit-jupiter]/[class:com.shop.OrderServiceTest]/[method:emptyCart()]`.
4. **Execution**: each engine runs its own part of the tree (Jupiter creates the instance, calls `@BeforeEach`, runs the extensions and the method).
5. The engine reports events (started, passed, failed) to the Platform, which forwards them to the IDE (green / red). `mvn test` works the same way, with Surefire instead of the IDE.

The import of the annotation decides which engine finds a test: `org.junit.jupiter.api.Test` is found by Jupiter, `org.junit.Test` (JUnit 4) by Vintage. Classic pitfall: importing `org.junit.Test` in a project with only Jupiter, no engine recognizes the method, and the test **silently doesn't run** ("no tests found"). Engines can be restricted (Surefire `<includeEngines>`), but by default every engine on the classpath is asked.

Why this split: in JUnit 4, IDEs and build tools relied on JUnit's internals, so JUnit could hardly evolve without breaking them, and every other framework needed its own integration in every tool. Now tools integrate once with the Platform, any framework providing an engine works everywhere, and several engines can run in the same project, which allowed migrating from JUnit 4 to 5 progressively (Vintage and Jupiter side by side).

## Extensions

> **In short**: An extension (`@ExtendWith`) hooks into the test lifecycle to share a behavior across tests, by composition rather than inheritance: callbacks, parameter injection (`ParameterResolver`), test instance post-processing, conditions. It's how Mockito and Spring integrate with JUnit.

An extension shares a behavior across many tests, declared once instead of being called manually in each test. It's JUnit's plugin mechanism, hooking into the test lifecycle like a servlet filter or an aspect for tests.

- **Composition instead of inheritance**: a shared `AbstractDatabaseTest` base class limits a test to one parent, while a test class can combine several extensions: `@ExtendWith({MockitoExtension.class, TimingExtension.class})`.
- **More than a shared setup**: an extension acts inside JUnit itself, and can do what a test can't do by hand: decide whether a test runs, inject parameters, prepare the test instance after its creation (how Mockito fills the `@Mock` fields and Spring the `@Autowired` ones), handle exceptions.

This is how Mockito and Spring integrate with JUnit:

```java
@ExtendWith(MockitoExtension.class) // creates the @Mock and injects them in @InjectMocks
class OrderServiceTest {
  @Mock OrderRepository repository;
  @InjectMocks OrderService service;
}

@SpringBootTest // meta-annotated with @ExtendWith(SpringExtension.class): starts the Spring context
class OrderIntegrationTest {
  @Autowired OrderService service; // injected by the extension
}
```

An extension implements one or more extension points, interfaces called by JUnit at a given moment:

- **Lifecycle callbacks**: `BeforeAllCallback`, `BeforeEachCallback`, `AfterEachCallback`, `AfterAllCallback`, `BeforeTestExecutionCallback`...
- **`ParameterResolver`**: provides the parameters of test methods or constructors (this is how `@Autowired` works on test method parameters with Spring, or `TestInfo` / `@TempDir` are injected).
- **`TestExecutionExceptionHandler`**: handles exceptions thrown by tests.
- **`ExecutionCondition`**: decides whether a test runs (`@EnabledOnOs`, `@EnabledIfEnvironmentVariable` are built this way).

```java
public class TimingExtension implements BeforeEachCallback, AfterEachCallback {

  @Override
  public void beforeEach(ExtensionContext context) {
    store(context).put("start", System.currentTimeMillis());
  }

  @Override
  public void afterEach(ExtensionContext context) {
    long start = store(context).get("start", Long.class);
    System.out.printf("%s: %d ms%n", context.getDisplayName(), System.currentTimeMillis() - start);
  }

  private ExtensionContext.Store store(ExtensionContext context) {
    return context.getStore(ExtensionContext.Namespace.create(getClass()));
  }
}

@ExtendWith(TimingExtension.class)
class SlowServiceTest { ... }
```

A `ParameterResolver` injects values into test methods, constructors and lifecycle methods (`@BeforeEach`...). JUnit asks each registered resolver if it supports a parameter, then calls the one that does. Example: injecting a fixed `Clock`, to test time-dependent code with a predictable date:

```java
public class FixedClockResolver implements ParameterResolver {

  @Override
  public boolean supportsParameter(ParameterContext parameter, ExtensionContext context) {
    return parameter.getParameter().getType() == Clock.class;
  }

  @Override
  public Object resolveParameter(ParameterContext parameter, ExtensionContext context) {
    return Clock.fixed(Instant.parse("2026-01-01T10:00:00Z"), ZoneOffset.UTC);
  }
}

@ExtendWith(FixedClockResolver.class)
class InvoiceServiceTest {

  @Test
  void dueDateIs30DaysLater(Clock clock) { // injected by the resolver
    InvoiceService service = new InvoiceService(clock);
    assertEquals(LocalDate.of(2026, 1, 31), service.createInvoice().dueDate());
  }
}
```

A parameter no resolver supports fails the test with a `ParameterResolutionException` (same if two resolvers support it). JUnit's built-in resolvers inject `TestInfo` (current test name, tags), `TestReporter` and `@TempDir`.

- Extensions are stateless: state is kept in the `ExtensionContext.Store`, not in fields (see ExtensionContext below).
- `@ExtendWith` can be put on a custom annotation, which then activates the extension (`@SpringBootTest` does exactly that).
- `@RegisterExtension` on a field registers an extension instance configured in code (e.g. a WireMock server with a given port).

## ExtensionContext

> **In short**: The `ExtensionContext` passed to every extension method describes where the execution is (class, method, display name, tags) and gives access to a `Store`, where extensions keep their state, scoped by a `Namespace` to avoid collisions.

The `ExtensionContext` is the object JUnit passes to every extension method (`beforeEach(context)`, `resolveParameter(..., context)`...). It describes where the execution currently is, and gives access to everything attached to it:

```java
context.getDisplayName();         // "dueDateIs30DaysLater(Clock)"
context.getRequiredTestClass();   // InvoiceServiceTest.class
context.getRequiredTestMethod();  // the current Method (throws at class level)
context.getTestInstance();        // Optional: the test class instance
context.getTags();                // the test's @Tag values
context.getExecutionException();  // Optional: the exception thrown by the test (in afterEach)
context.getConfigurationParameter("my.param"); // from junit-platform.properties
context.getStore(namespace);      // storage for the extension
```

Contexts form a hierarchy, one per node of the test tree, each knowing its parent (`getParent()`, `getRoot()`):

```
root context (engine)                         ← the whole run
└── class context (InvoiceServiceTest)        ← received in beforeAll / afterAll
    └── method context (dueDateIs30DaysLater) ← received in beforeEach / afterEach
```

**The `Store`**: since an extension instance can serve several tests (possibly in parallel), it keeps its state in the store of the context, not in fields.

- Each context has its own store, discarded with it: a method store after the method, a class store after the class. A child context sees the values of its parents.
- A stored value implementing `AutoCloseable` is closed automatically when its context ends.
- The store is shared by all extensions of the context, so values are identified by **(namespace, key)**, not by the key alone: two extensions both using the key `"start"` would otherwise overwrite each other. Each extension uses its own namespace, usually its class (`Namespace.create(TimingExtension.class)`), and `Namespace.GLOBAL` exists to share data on purpose.

Typical use: a costly resource started **once** for the whole run, then reused and closed at the end (how extensions sharing a Testcontainers container or a WireMock server work):

```java
@Override
public void beforeAll(ExtensionContext context) {
  context.getRoot() // root context: shared by all test classes
      .getStore(Namespace.create(MyServerExtension.class))
      .getOrComputeIfAbsent("server", key -> MyServer.start(), MyServer.class);
}
```

## Other useful features

> **In short**: `@TempDir`, `@Timeout`, assumptions to skip a test when a precondition isn't met, dynamic tests generated at runtime, and parallel execution.

- **`@TempDir`**: injects a temporary directory, deleted after the test.
- **`@Timeout(2)`**: fails the test if it takes more than 2 seconds; `assertTimeout()` for a block of code.
- **Assumptions**: `assumeTrue(condition)` skips the test (instead of failing it) when a precondition isn't met, e.g. Docker not available.
- **Dynamic tests**: a `@TestFactory` method returns a `Stream<DynamicTest>` generated at runtime (e.g. one test per file in a folder).
- **Parallel execution**: `junit.jupiter.execution.parallel.enabled=true` in `junit-platform.properties`, with `@Execution(CONCURRENT)` / `@ResourceLock` to control what can run in parallel. Only safe if tests don't share state.
