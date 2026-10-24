## Spring bean scopes

The scope defines how many instances of a bean the container creates, and how long they live. Set with `@Scope` (or shortcuts like `@RequestScope`) on the class or the `@Bean` method.

| Scope         | Instances                                      | Typical use                                 |
| ------------- | ---------------------------------------------- | ------------------------------------------- |
| `singleton`   | one per container (**default**)                | services, repositories, controllers         |
| `prototype`   | a new one each time it's injected or requested | stateful, short-lived objects               |
| `request`     | one per HTTP request                           | request-specific data (current user, trace) |
| `session`     | one per HTTP session                           | user-specific data (shopping cart)          |
| `application` | one per `ServletContext`                       | rarely used, close to `singleton`           |

- A singleton is shared by all threads (one per request): it must be stateless, or its fields thread-safe.
- Spring doesn't manage the end of a prototype: `@PreDestroy` is never called on it.

Pitfall: a shorter-lived bean injected into a singleton is injected **only once**, when the singleton is created, so the singleton keeps the same instance forever.

- **`request` / `session`**: nothing to do with `@RequestScope` / `@SessionScope`, which inject a proxy by default, forwarding each call to the bean of the current request or session. Only a plain `@Scope("request")` needs `proxyMode = ScopedProxyMode.TARGET_CLASS`.
- **`prototype`**: inject an `ObjectProvider<T>` and call `getObject()` when a new instance is needed. A proxy would create a new instance on **every method call**, so two consecutive calls wouldn't hit the same object.

```java
@Component
@Scope("prototype")
public class ReportBuilder { // stateful: accumulates lines
  private final List<String> lines = new ArrayList<>();
  private final MoneyFormatter formatter; // injected by Spring

  public ReportBuilder(MoneyFormatter formatter) {
    this.formatter = formatter;
  }

  public void addLine(Order order) { lines.add(order.id() + ": " + formatter.format(order.total())); }
  public String build() { return String.join("\n", lines); }
}

@Service
public class ReportService { // singleton
  private final ObjectProvider<ReportBuilder> builderProvider;

  public ReportService(ObjectProvider<ReportBuilder> builderProvider) {
    this.builderProvider = builderProvider;
  }

  public String monthlyReport(List<Order> orders) {
    ReportBuilder builder = builderProvider.getObject(); // new instance for this report
    orders.forEach(builder::addLine);
    return builder.build();
  }
}
```

Injecting `ReportBuilder` directly would share a single builder between all calls (and threads), mixing the lines of every report.

A prototype bean is only worth it when the object needs Spring: injected dependencies (`MoneyFormatter` here), `@Value`, or AOP (`@Transactional`...). Without any of that, a plain `new ReportBuilder()` is simpler.

## Spring retry (@Retryable)

Since Spring Framework 7 (Spring Boot 4), retry is built into the core (`org.springframework.resilience`): a failing method is called again automatically, for transient errors (timeout, `503`, lost connection).

```java
@Configuration
@EnableResilientMethods // enables @Retryable and @ConcurrencyLimit
class AppConfig {}

@Service
public class NotificationService {

  @Retryable(
      includes = MessageDeliveryException.class, // only these exceptions (and their causes)
      maxRetries = 4,  // 1 initial call + 4 retries
      delay = 100,     // 100 ms before the 1st retry
      multiplier = 2,  // exponential backoff: 100, 200, 400, 800 ms
      maxDelay = 1000,
      jitter = 10)     // random ± 10 ms, so clients don't all retry at the same time
  public void send(Notification notification) { ... }
}
```

- Defaults: every exception is retried, 3 retries, 1 second between attempts. `excludes` and `predicate` refine which exceptions are retried.
- When all attempts fail, the last exception is propagated to the caller.
- Also works on reactive methods (`Mono` / `Flux`).

Programmatic version, without annotation:

```java
var retryTemplate = new RetryTemplate(RetryPolicy.builder()
    .includes(MessageDeliveryException.class)
    .maxRetries(4)
    .delay(Duration.ofMillis(100))
    .multiplier(2)
    .build());

retryTemplate.invoke(() -> client.send(notification));
```

Pitfalls:

- **Proxy-based**: a call from inside the same class (`this.send()`) isn't retried, like `@Transactional`.
- **Idempotence**: only retry operations that are safe to run twice (a payment whose first call may have succeeded isn't).
- **Only transient errors**: a `4xx` or a business exception fails the same way every time.
- **With `@Transactional`**: the retry must wrap the transaction, so each attempt runs in a new one (see below).

Retry and transaction: Hibernate sends the `UPDATE`s at flush time, usually at commit, which the `@Transactional` proxy runs **after** the method returns. A retry inside the transaction never sees the error: an optimistic locking failure (`@Version`, the typical case) is thrown at commit, outside the retry. And even when the error is thrown inside (explicit `flush()`), the transaction is already lost: marked rollback-only, aborted by the database (PostgreSQL), and the persistence context still holds the stale entity. Each attempt needs a new transaction that reloads the data. Simplest setup: the retry in one bean, the transaction in another, so each attempt goes through the transactional proxy.

```java
@Service
public class OrderFacade { // must NOT be @Transactional, or the attempts would join its transaction
  private final OrderService orderService;

  @Retryable(includes = ObjectOptimisticLockingFailureException.class, maxRetries = 3, delay = 50)
  public void confirm(long orderId) {
    orderService.confirm(orderId); // new transaction for each attempt
  }
}

@Service
public class OrderService {
  @Transactional
  public void confirm(long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow(); // reloaded on each attempt
    order.confirm();
  }
}
```

Programmatic alternative in a single method: `retryTemplate.invoke(() -> transactionTemplate.executeWithoutResult(status -> ...))`.

Related: `@ConcurrencyLimit(10)` limits concurrent calls to a method, e.g. to protect a fragile API (useful with virtual threads).
