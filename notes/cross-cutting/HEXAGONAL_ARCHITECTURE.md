## Why hexagonal architecture

> **In short**: In a classic layered architecture, the business logic depends on the database and the framework: JPA entities double as the domain model, and nothing can be tested without them. Hexagonal architecture (Alistair Cockburn, also called Ports and Adapters) puts the business core at the center, independent of any technology, and connects the outside world to it through interfaces. The core is testable alone, and the technical choices can change without touching it.

**The problem with the classic layered architecture**:

```text
OrderController ──> OrderService ──> OrderRepository extends JpaRepository<OrderEntity, Long> ──> DB
```

- Dependencies go from top to bottom, so the business logic (`OrderService`) depends on the persistence layer.
- The `@Entity` is often used as the domain model: its shape is dictated by the table and by JPA (no-arg constructor, setters, lazy loading), not by the business rules.
- Testing a business rule needs a database or mocks of Spring Data; changing the persistence technology touches the business code.

**The idea**: invert the dependency. The business core defines what it needs as interfaces (an `OrderRepository` port in plain Java), and the technical code implements them (a JPA adapter). The technical code depends on the core, never the reverse.

Cockburn's original goal: an application that can be driven equally by users, other programs, automated tests or batch scripts, and that can be developed and tested in isolation from its databases and external systems. The hexagon shape has no meaning in itself (six isn't a rule): it's just a way to draw several entry and exit points around the core.

What it brings:

- **Testability**: the business rules are tested with plain unit tests, without Spring or a database.
- **Technology independence**: replacing PostgreSQL with MongoDB, or REST with Kafka, means writing a new adapter, not rewriting the business code.
- **Several entry points**: a REST controller, a Kafka consumer and a batch job call the same use cases.
- **Readable business code**: the core only expresses business rules, without technical noise.

When not to use it: a simple CRUD application, where the "business logic" is mostly reading and writing tables. The extra interfaces and mapping cost more than they bring.

## The layers: domain, application, ports and adapters

> **In short**: The core contains the domain (entities, value objects, business rules) and the application layer (use cases). The core exposes inbound ports (what it offers) and defines outbound ports (what it needs). Around it, driving adapters (REST, Kafka consumer) call the inbound ports, and driven adapters (JPA, HTTP client, Kafka producer) implement the outbound ports.

```text
driving adapters                     CORE                        driven adapters
                    ┌──────────────────────────────────────┐
                    │ in port             out ports        │
OrderController ──> │ PlaceOrderUseCase   OrderRepository  │ <── OrderJpaAdapter
OrderConsumer   ──> │        │            PaymentGateway   │ <── StripePaymentAdapter
                    │        ▼                             │
                    │ PlaceOrderService   (application)    │
                    │        │                             │
                    │        ▼                             │
                    │ Order, Money        (domain)         │
                    └──────────────────────────────────────┘
```

The horizontal arrows show dependencies: both kinds of adapters depend on the core (a driving adapter calls an inbound port, a driven adapter implements an outbound port). Inside the core, `PlaceOrderService` implements `PlaceOrderUseCase` and uses the domain.

**Domain**: the business model and its rules, in plain Java (no Spring, JPA or Jackson annotations).

```java
public class Order {
  private final OrderId id;
  private final List<OrderLine> lines;
  private OrderStatus status;

  public void ship() {
    if (status != OrderStatus.PAID) {
      throw new OrderNotPaidException(id); // business rule
    }
    status = OrderStatus.SHIPPED;
  }
}
```

- Entities (`Order`, with an identity), value objects (`Money`, `OrderId`, immutable and compared by value, often `record`s), domain exceptions.
- The rules live in the objects (`order.ship()`), not in a service that changes fields through setters.

**Application**: the use cases, one per business action. A use case loads the domain objects through the outbound ports, calls the domain, and saves the result. It contains no business rule itself, only orchestration.

**Ports**: interfaces that belong to the core.

```java
// inbound port: what the application offers, implemented by the application layer
public interface PlaceOrderUseCase {
  OrderId placeOrder(PlaceOrderCommand command);
}

// outbound ports: what the application needs, implemented by adapters
public interface OrderRepository {
  Optional<Order> findById(OrderId id);
  void save(Order order);
}

public interface PaymentGateway {
  PaymentResult charge(CustomerId customer, Money amount);
}
```

- An outbound port is expressed in business terms (`save(Order)`, `charge(...)`), not technical ones (`executeQuery`, `postJson`).
- Inbound ports are sometimes skipped: the controller calls the use case class directly. The core is still isolated, since the dependency already points inwards.

**Adapters**: the technical code, outside the core.

|                      | Driving (primary) adapters                                | Driven (secondary) adapters                               |
| -------------------- | --------------------------------------------------------- | --------------------------------------------------------- |
| Role                 | call the application                                      | are called by the application                             |
| Relation to the port | call an inbound port                                      | implement an outbound port                                |
| Examples             | REST controller, Kafka consumer, scheduled job, CLI, test | JPA repository, Kafka producer, HTTP client, email sender |

A test is a driving adapter too: it calls the use case directly, with in-memory implementations of the outbound ports.

## How the layers communicate

> **In short**: A driving adapter converts its input (REST DTO, Kafka message) into a command and calls the inbound port. The use case loads the domain objects through the outbound ports, calls the business methods, and saves the result; the driven adapters map the domain to their technical model (`@Entity`). Each boundary has its own model, and the mapping is done by the adapters, so the core never sees a DTO or an entity.

**The path of a request**, `POST /orders` (here the arrows show the flow of the call, not the dependencies):

```text
OrderRequest ──> OrderController ──> PlaceOrderCommand ──> PlaceOrderUseCase
  (JSON DTO)     (driving adapter)                           │ implemented by
                                                             ▼
                                                     PlaceOrderService ──> Order.place(...)   (domain)
                                                             │
                                                             ▼ OrderRepository.save(order)    (out port)
                                                     OrderJpaAdapter ──> OrderEntity ──> DB
```

```java
// driving adapter: REST DTO → command
@RestController
class OrderController {
  private final PlaceOrderUseCase placeOrder;

  @PostMapping("/orders")
  ResponseEntity<Void> place(@Valid @RequestBody OrderRequest request) {
    OrderId id = placeOrder.placeOrder(request.toCommand());
    return ResponseEntity.created(URI.create("/orders/" + id.value())).build();
  }
}

// application: orchestration only, the rules are in the domain
class PlaceOrderService implements PlaceOrderUseCase {
  private final OrderRepository orders;          // out port
  private final OrderEventPublisher events;      // out port

  @Transactional
  public OrderId placeOrder(PlaceOrderCommand command) {
    List<OrderLine> lines = command.lines().stream()
        .map(line -> new OrderLine(line.productId(), line.quantity()))
        .toList();
    Order order = Order.place(command.customerId(), lines); // business rules
    orders.save(order);
    events.publish(new OrderPlaced(order.id()));
    return order.id();
  }
}

// driven adapter: domain ↔ JPA entity
@Component
class OrderJpaAdapter implements OrderRepository {
  private final OrderJpaRepository jpa; // Spring Data: JpaRepository<OrderEntity, UUID>

  public void save(Order order) {
    jpa.save(OrderEntity.fromDomain(order));
  }

  public Optional<Order> findById(OrderId id) {
    return jpa.findById(id.value()).map(OrderEntity::toDomain);
  }
}
```

**One model per boundary**:

| Model                                  | Lives in                                        | Converted by                                |
| -------------------------------------- | ----------------------------------------------- | ------------------------------------------- |
| `OrderRequest` / `OrderResponse` (DTO) | the web adapter                                 | the controller                              |
| `PlaceOrderCommand`                    | the application layer, next to the inbound port | each driving adapter, from its own format   |
| `Order`, `OrderLine`, `Money`          | the domain                                      | —                                           |
| `OrderEntity` (`@Entity`)              | the JPA adapter                                 | the JPA adapter (`fromDomain` / `toDomain`) |

- **Commands in, domain or dedicated result out**: the inbound port takes a command, not a REST DTO (the core would depend on the web format) nor a domain object (the adapter would contain the creation logic). It returns an id, a domain object mapped by the controller, or a read-only view (`OrderView`) when the adapter shouldn't be able to call business methods.
- **Validation at two levels**: format in the adapter (`@Valid`, `@NotNull` on the DTO), business rules in the domain (`Order.place` refuses an empty order).
- **The use case is the transaction boundary**: `@Transactional` on the use case covers all the port calls of one business action. It's a Spring annotation in the core, a common compromise (see the rules below).
- **Events** go through an outbound port too (`OrderEventPublisher`), implemented by a Kafka adapter; the core doesn't know Kafka.
- **At runtime**, Spring injects the adapter into the use case through the port: `PlaceOrderService` only knows `OrderRepository`, and Spring finds the `OrderJpaAdapter` bean that implements it.

Pitfall with JPA: the domain `Order` isn't a managed entity, so dirty checking no longer saves the changes automatically. The use case must call `orders.save(order)` explicitly after a change, and the adapter maps it back onto the entity (loaded by id, or merged).

## Rules to respect

> **In short**: Dependencies point inwards: adapters → application → domain, never the reverse. The domain has no framework dependency, ports belong to the core and speak business terms, no DTO or `@Entity` crosses the core, and adapters only talk to each other through the core. Spring annotations in the application layer (`@Transactional`, `@Service`) are a common compromise; the strict version adds them from the outside. ArchUnit tests or separate modules enforce the rules.

```text
com.shop.order
├── domain        # Order, Money, OrderNotPaidException: depends on nothing
├── application
│   ├── port/in   # PlaceOrderUseCase, PlaceOrderCommand
│   ├── port/out  # OrderRepository, PaymentGateway
│   └── service   # PlaceOrderService: depends on domain and ports only
├── adapter
│   ├── in/web    # OrderController, OrderRequest
│   └── out/jpa   # OrderJpaAdapter, OrderEntity, OrderJpaRepository
└── config        # @Configuration: wires the beans
```

**The rules**:

1. **Dependencies point inwards.** The domain depends on nothing, the application only on the domain, the adapters on the application and the domain. No `import ...adapter...` in the core.
2. **No framework in the domain**: no `@Entity`, `@JsonProperty`, `@Component`, `@Autowired`. Plain Java, testable with `new`.
3. **Ports belong to the core and speak business terms**: `OrderRepository.save(Order)`, designed from the needs of the use cases, not a copy of `JpaRepository` (`flush()`, `Pageable`, `Specification` have nothing to do there).
4. **No technical model crosses the core**: no DTO or `@Entity` in a port signature. Technical exceptions don't leak either: the adapter catches a `DataAccessException` or an HTTP `5xx` and throws an exception defined by the port (`PaymentFailedException`).
5. **Adapters don't call each other**: the controller doesn't call `OrderJpaRepository` directly, even for a simple read; it goes through a use case.
6. **Business rules live in the domain**, not in the controller, the use case or the JPA adapter. A use case usually avoids calling another use case's port (nested transactions, hidden coupling): shared logic goes into the domain.

**Spring in the application layer**: strictly, the application layer doesn't know Spring either. Two approaches:

- **Pragmatic**: accept `@Service` and `@Transactional` on the use cases. Annotations are only metadata: unit tests without Spring ignore them. The cost is a compile-time dependency on Spring in the core. The domain stays pure in both cases.
- **Strict**: the use case is plain Java, and the transaction is added from the outside, in one of two ways.

_Decorate the bean when it's created_: the `@Configuration` (outside the core) declares the use case and wraps it in a transaction, so no `@Service` either.

```java
@Configuration
class OrderConfig {
  @Bean
  PlaceOrderUseCase placeOrderUseCase(OrderRepository orders, OrderEventPublisher events,
                                      PlatformTransactionManager txManager) {
    PlaceOrderService service = new PlaceOrderService(orders, events);
    TransactionTemplate tx = new TransactionTemplate(txManager);
    return command -> tx.execute(status -> service.placeOrder(command));
  }
}
```

_An outbound port for transactions_: the core defines a `TransactionRunner` port, implemented by an adapter with Spring's `TransactionTemplate`.

```java
// application: outbound port
public interface TransactionRunner {
  <T> T inTransaction(Supplier<T> action);
}

// adapter
@Component
class SpringTransactionRunner implements TransactionRunner {
  private final TransactionTemplate tx;

  SpringTransactionRunner(PlatformTransactionManager txManager) {
    this.tx = new TransactionTemplate(txManager);
  }

  public <T> T inTransaction(Supplier<T> action) {
    return tx.execute(status -> action.get());
  }
}

// use case
public OrderId placeOrder(PlaceOrderCommand command) {
  return transaction.inTransaction(() -> {
    Order order = Order.place(command.customerId(), toLines(command));
    orders.save(order);
    return order.id();
  });
}
```

|                      | Decorated bean                                   | `TransactionRunner` port                                  |
| -------------------- | ------------------------------------------------ | --------------------------------------------------------- |
| Use case code        | no transaction code at all                       | calls `inTransaction(...)`                                |
| Transaction boundary | the whole use case, decided in the configuration | explicit in the code, can cover only part of the use case |
| Unit test            | test `PlaceOrderService` directly                | a fake runner that just calls `action.get()`              |

**Enforcing the rules**: without a check, an `import` of an adapter in the core eventually slips in.

- **Separate modules** (Maven/Gradle): a `domain` module without the Spring dependency can't compile a Spring annotation. The strongest guarantee, at the cost of a heavier build.
- **ArchUnit**: tests that check the dependencies between packages.

```java
@AnalyzeClasses(packages = "com.shop.order")
class ArchitectureTest {
  @ArchTest
  static final ArchRule domainIsPure = noClasses()
      .that().resideInAPackage("..domain..")
      .should().dependOnClassesThat()
      .resideInAnyPackage("..application..", "..adapter..", "org.springframework..", "jakarta.persistence..");

  @ArchTest
  static final ArchRule coreDoesNotKnowAdapters = noClasses()
      .that().resideInAnyPackage("..domain..", "..application..")
      .should().dependOnClassesThat().resideInAPackage("..adapter..");
}
```

ArchUnit also has a predefined rule for this layout: `Architectures.onionArchitecture()` (domain models, domain services, application services, adapters).

## Hexagonal vs layered, onion and clean architecture

> **In short**: The layered architecture makes the business depend on persistence. Onion (Palermo) and Clean Architecture (Robert C. Martin) share the hexagonal idea: the domain at the center, dependencies pointing inwards. They differ mostly in vocabulary and in how much they prescribe the inside of the core; hexagonal only separates the inside from the outside through ports.

| | Layered (n-tier) | Hexagonal (Cockburn, 2005) | Onion (Palermo, 2008) | Clean (Martin, 2012) |
|---|---|---|---|---|
| Shape | stacked layers | a core with ports, adapters around | concentric circles | concentric circles |
| Dependencies | top to bottom: business → persistence | inwards | inwards | inwards ("dependency rule") |
| Inside of the core | — | not prescribed | domain model, domain services, application services | entities, use cases |
| Outside | presentation, persistence | driving and driven adapters | UI, infrastructure, tests | interface adapters, frameworks and drivers |

- The three "centric" architectures are the same idea with different words: they can be combined (the layout of this note is hexagonal on the outside, onion-like on the inside).
- In an interview, the key point is the dependency inversion: the core defines the interfaces, the technical code implements them.

## Common pitfalls

> **In short**: The most frequent mistakes: an anemic domain (all the rules in the services), ports that copy the technology (`extends JpaRepository`), mapping and interfaces everywhere "just in case", and one big hexagon for the whole application instead of one per business module. Hexagonal architecture is a cost to pay where the domain justifies it.

- **Anemic domain**: `Order` with only getters and setters, all the rules in `PlaceOrderService`. The structure is hexagonal, but the business is still procedural code in services; the rules belong in the domain objects (`order.ship()`).
- **Leaky ports**: `interface OrderRepository extends JpaRepository<OrderEntity, Long>` puts JPA in the core. A port is a plain interface with business methods.
- **Over-engineering**: an interface for every class, a port per method, mapping between identical classes. Ports are only needed at real boundaries (persistence, external systems, messaging); a mapper (MapStruct) reduces the boilerplate.
- **Reads through the domain**: loading full aggregates to display a list is slow and useless. Read use cases can use a dedicated outbound port returning views (`OrderSummaryQuery` → `List<OrderSummary>`, implemented with a JPA projection or SQL), without going through the domain model (a light form of CQRS).
- **One global hexagon**: `domain/`, `application/`, `adapter/` at the root of the whole application mixes every business area. Package by business module first (`com.shop.order`, `com.shop.payment`), each one with its own hexagon; modules talk through their inbound ports or events, not by reading each other's tables.
- **Applying it everywhere**: for a CRUD module or a thin proxy to another service, a simple controller → service → repository is enough, even inside an otherwise hexagonal application.
