## What design patterns are

> **In short**: Design patterns are proven, named solutions to recurring design problems. The reference is the "Gang of Four" book (1994, 23 patterns), in three families: creational (how objects are created), structural (how objects are composed) and behavioral (how objects interact). In interviews, the expected answer is the problem a pattern solves and where it appears in the JDK or Spring.

| Family     | Question               | Main patterns                                                         |
| ---------- | ---------------------- | --------------------------------------------------------------------- |
| Creational | how to create objects? | Singleton, Factory Method, Builder                                    |
| Structural | how to compose them?   | Adapter, Decorator, Proxy, Facade, Bridge                             |
| Behavioral | how do they interact?  | Strategy, Observer, Template Method, Chain of Responsibility, Command |

A pattern is a vocabulary as much as a solution: saying "a decorator around the repository" describes a design in a few words. Applied without a real need, it only adds indirection.

## Creational patterns: Singleton, Factory Method, Builder

> **In short**: Singleton guarantees a single instance (in Spring, beans already are). Factory Method hides which class is instantiated behind a method: a static method replacing the constructor (`List.of`), or, in the GoF pattern, a creation method overridden by subclasses. Builder builds an object with many optional parameters step by step, with a readable call and an immutable result.

| Pattern        | Problem solved                                       | In the JDK / Spring                                                                                                |
| -------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Singleton      | one shared instance                                  | `Runtime.getRuntime()`, Spring beans (singleton scope); details in [Singleton](JAVA.md#singleton)                  |
| Factory Method | create an object without exposing its concrete class | static factory methods: `List.of()`, `Optional.empty()`, `Integer.valueOf()`; GoF pattern: `Collection.iterator()` |
| Builder        | many parameters, some optional                       | `StringBuilder`, `HttpRequest.newBuilder()`, `UriComponentsBuilder`, Lombok `@Builder`                             |

**Factory Method** has two meanings:

- **Static factory method** (_Effective Java_, item 1): a `static` method instead of a constructor. It has a name (`Optional.of` / `Optional.empty`), can return a cached instance (`Integer.valueOf(42)`), or a hidden subclass chosen from the parameters (`EnumSet.noneOf` returns a `RegularEnumSet` or a `JumboEnumSet` depending on the enum size).
- **GoF pattern**: a base class uses an object in its algorithm but lets subclasses decide which concrete class to create, by overriding a creation method (`ReportExporter.createWriter()` returning a `PdfWriter` or a `CsvWriter`). In the JDK, each collection implements `iterator()` with its own `Iterator` class.

A "simple factory" (a class with a `switch` on a type) isn't a GoF pattern; in Spring, it's usually replaced by injecting every implementation, as in the Strategy example below.

**Builder**: avoids constructors with 6 parameters in a row (which one is the email?) and telescoping constructors.

```java
public final class User {
  private final String name;
  private final String email;
  private final int age;

  private User(Builder builder) {
    this.name = Objects.requireNonNull(builder.name);
    this.email = builder.email;
    this.age = builder.age;
  }

  public static Builder builder() { return new Builder(); }

  public static class Builder {
    private String name;
    private String email;
    private int age;

    public Builder name(String name) { this.name = name; return this; }
    public Builder email(String email) { this.email = email; return this; }
    public Builder age(int age) { this.age = age; return this; }
    public User build() { return new User(this); }
  }
}

User user = User.builder().name("Alice").email("alice@example.com").build();
```

Validation goes in `build()` (or the private constructor), so an invalid `User` can't exist. With a `record` and few parameters, a plain constructor is often enough.

## Structural patterns: Adapter, Decorator, Proxy, Facade, Bridge

> **In short**: Adapter, Decorator and Proxy wrap an object, with different intents. Adapter converts one interface into another that the client expects. Decorator adds behavior while keeping the same interface, and can be stacked. Proxy keeps the same interface but controls access (lazy loading, transactions, security): Spring AOP is built on it. Facade offers a simple interface over a complex subsystem. Bridge separates an abstraction from its implementation so that both vary independently (the JDBC API and its drivers).

| Pattern   | Intent                                                                    | In the JDK / Spring                                                                       |
| --------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Adapter   | make an incompatible interface usable                                     | `InputStreamReader` (bytes → chars), `Arrays.asList()`, Spring MVC `HandlerAdapter`       |
| Decorator | add behavior, same interface, stackable                                   | `BufferedInputStream`, `Collections.unmodifiableList()`                                   |
| Proxy     | control access to the object, same interface                              | `@Transactional` / `@Cacheable` beans, Hibernate lazy entities, `java.lang.reflect.Proxy` |
| Facade    | simplify a complex subsystem                                              | `JdbcTemplate` over JDBC, SLF4J over logging frameworks                                   |
| Bridge    | two dimensions that vary independently, without one class per combination | JDBC: the API (`Connection`, `Statement`) and the drivers                                 |

**Decorator**: the wrapper implements the same interface as the wrapped object and delegates to it.

```java
interface Notifier { void send(String message); }

class EmailNotifier implements Notifier {
  public void send(String message) { /* sends the email */ }
}

class RetryingNotifier implements Notifier {
  private final Notifier delegate;

  RetryingNotifier(Notifier delegate) { this.delegate = delegate; }

  public void send(String message) {
    try {
      delegate.send(message);
    } catch (RuntimeException e) {
      delegate.send(message); // one retry
    }
  }
}

Notifier notifier = new RetryingNotifier(new EmailNotifier());
```

The JDK combines several of them in one line: `new BufferedReader(new InputStreamReader(new FileInputStream("data.txt")))`: `InputStreamReader` adapts a byte stream into a `Reader`, `BufferedReader` decorates it with buffering.

**Decorator vs Proxy**: same structure, different intent. A decorator is composed explicitly by the caller to add features; a proxy is usually created by a framework and stands in for the object to control access to it, often without the caller knowing (Spring injects the proxy instead of the bean).

**Bridge**: with notification types (alert, reminder) and channels (email, SMS), inheritance gives one class per combination (`EmailAlert`, `SmsAlert`...). Bridge links the two hierarchies by composition: 2 + 2 classes instead of 4, and a new channel is one class.

```java
interface MessageSender { void send(String to, String text); } // implementation: EmailSender, SmsSender

abstract class Notification {                                  // abstraction: AlertNotification, ReminderNotification
  protected final MessageSender sender;                        // the bridge
  Notification(MessageSender sender) { this.sender = sender; }
  abstract void notify(User user);
}

new AlertNotification(new SmsSender()).notify(user);
```

**Bridge vs Adapter**: both use composition, but an adapter makes two existing, incompatible interfaces work together after the fact, while a bridge is designed upfront so that two hierarchies can evolve separately.

## Behavioral patterns: Strategy, Observer, Template Method, Chain of Responsibility, Command

> **In short**: Strategy makes an algorithm interchangeable behind an interface (`Comparator`), replacing `if`/`switch` chains. Observer notifies subscribers of an event without knowing them (Spring events). Template Method fixes the steps of an algorithm in a base class and lets subclasses fill some of them. Chain of Responsibility passes a request along a chain of handlers, each one handling it or passing it on (servlet filters). Command turns an action into an object, so it can be queued, scheduled, logged or undone (`Runnable`).

| Pattern                 | Problem solved                                       | In the JDK / Spring                                                                                                              |
| ----------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Strategy                | choose an algorithm at runtime                       | `Comparator`, `PasswordEncoder`, any interface with several beans                                                                |
| Observer                | notify several objects of an event, without coupling | `ApplicationEventPublisher` + `@EventListener`, RxJS, `PropertyChangeListener` (`java.util.Observer` is deprecated since Java 9) |
| Template Method         | fixed steps, some customizable                       | `HttpServlet.service()` calling `doGet()` / `doPost()`, `AbstractList`                                                           |
| Chain of Responsibility | several handlers, each may handle or pass on         | servlet `Filter` / `FilterChain`, Spring Security filter chain, `HandlerInterceptor`                                             |
| Command                 | an action as an object: queue, schedule, undo        | `Runnable` / `Callable` submitted to an `ExecutorService`, undo/redo in editors                                                  |

Also often cited: **Iterator** (`Iterator`, for-each).

**Strategy with Spring**: each strategy is a bean, and the service picks one at runtime instead of a `switch`.

```java
public interface PaymentMethod {
  String type();
  void pay(Order order);
}

@Component class CardPayment implements PaymentMethod { ... }    // type() = "CARD"
@Component class PaypalPayment implements PaymentMethod { ... }  // type() = "PAYPAL"

@Service
class PaymentService {
  private final Map<String, PaymentMethod> methods;

  PaymentService(List<PaymentMethod> all) { // Spring injects every implementation
    this.methods = all.stream().collect(Collectors.toMap(PaymentMethod::type, m -> m));
  }

  void pay(Order order, String type) {
    methods.get(type).pay(order);
  }
}
```

Adding a payment method means adding a class, without touching `PaymentService`: the Open/Closed principle (see [SOLID](../../cross-cutting/ARCHITECTURE.md#solid-principles)).

**Template Method vs Strategy**: Template Method fixes the skeleton of an algorithm in a base class and lets subclasses implement some steps (inheritance, chosen at compile time). Strategy delegates the whole behavior to an object implementing an interface, interchangeable at runtime (composition). Composition is usually preferred.

**Observer**: the subject publishes an event, the subscribed observers are notified; the subject only knows the observer interface, so adding a reaction doesn't touch it. In Spring, with application events:

```java
@Service
class OrderService {
  private final ApplicationEventPublisher events;

  @Transactional
  void placeOrder(Order order) {
    orderRepository.save(order);
    events.publishEvent(new OrderPlaced(order.getId())); // doesn't know who listens
  }
}

@Component
class ConfirmationEmailListener {
  @TransactionalEventListener // runs after the commit
  void on(OrderPlaced event) { ... }
}
```

- `@EventListener` is synchronous: same thread, same transaction, and an exception in the listener reaches the publisher. `@TransactionalEventListener` runs after the commit (no email for an order that was rolled back), `@Async` in another thread.
- Across applications, the same idea is publish/subscribe through a broker (Kafka).

**Chain of Responsibility**: the request goes from handler to handler; each one handles it and stops the chain, or passes it on. Classic example: an expense approved by the manager up to 1,000, otherwise passed to the director, then to the CEO. Servlet filters are a variant where each handler does its part and passes on:

```java
public class ApiKeyFilter implements Filter {
  public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    if (((HttpServletRequest) req).getHeader("X-Api-Key") == null) {
      ((HttpServletResponse) res).sendError(401); // stops the chain: the controller is never reached
      return;
    }
    chain.doFilter(req, res); // next filter, then the servlet
  }
}
```

- The `FilterChain` holds the position of the current filter, so the container creates one per request (Tomcat's `ApplicationFilterChain`, Spring Security's `VirtualFilterChain`). The filters themselves are single instances shared by all threads: they must be stateless.
- Unlike a decorator, which always delegates, a link of the chain can stop.

|              | Observer                                     | Chain of Responsibility                            |
| ------------ | -------------------------------------------- | -------------------------------------------------- |
| Who receives | every subscriber                             | the handlers in turn; one can stop the chain       |
| Shape        | broadcast: one event, independent reactions  | sequence: each handler decides what happens next   |
| Example      | `OrderPlaced` → email, stock, loyalty points | servlet filters, Spring Security, expense approval |

**Command**: once an action is an object, it can be stored, queued, scheduled, retried or undone.

```java
interface Command {
  void execute();
  void undo();
}

class AddTextCommand implements Command {
  private final Document document;
  private final String text;

  AddTextCommand(Document document, String text) { this.document = document; this.text = text; }

  public void execute() { document.append(text); }
  public void undo() { document.removeLast(text.length()); }
}

Deque<Command> history = new ArrayDeque<>();
Command command = new AddTextCommand(document, "Hello");
command.execute();
history.push(command);
history.pop().undo(); // Ctrl+Z
```

`executor.submit(() -> sendEmail(user))` is the everyday form: the action is passed as an object (`Runnable`) and run later, in another thread. Not to be confused with the commands of CQRS and hexagonal architecture (`PlaceOrderCommand`): plain data objects describing an intent, executed by a use case, without an `execute()` method.
