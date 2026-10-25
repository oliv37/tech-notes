## SOLID principles

> **In short**: Five design principles for code that is easy to change: one reason to change per class (S), extend without modifying (O), subtypes usable as their parent (L), small focused interfaces (I), depend on abstractions rather than implementations (D). Guidelines to apply where change is expected, not rules to apply blindly.

Five object-oriented design principles (Robert C. Martin) aimed at code that is easier to change, test and extend.

**S — Single Responsibility**: a class should have only one reason to change. An `InvoiceService` that computes totals, generates the PDF and sends the email changes for three different reasons; split it into `InvoiceCalculator`, `InvoicePdfGenerator`, `InvoiceMailer`.

**O — Open/Closed**: open for extension, closed for modification. Adding a behavior should mean adding code, not editing existing code.

```java
// ❌ every new payment method means editing this switch
switch (type) { case CARD -> ...; case PAYPAL -> ...; }

// ✅ a new payment method is a new implementation
interface PaymentMethod { void pay(Amount amount); }
class CardPayment implements PaymentMethod { ... }
class PaypalPayment implements PaymentMethod { ... }
```

**L — Liskov Substitution**: a subtype must be usable anywhere its parent type is expected, without breaking the behavior. Classic counter-example: `Square extends Rectangle`, where `setWidth()` also changes the height, so code that sets width and height on a `Rectangle` gets unexpected results with a `Square`. Throwing `UnsupportedOperationException` in an overridden method is another warning sign.

**I — Interface Segregation**: clients shouldn't depend on methods they don't use. Prefer several small interfaces (`Readable`, `Writable`) over a big one forcing implementations to stub half of the methods.

**D — Dependency Inversion**: high-level code depends on abstractions, not on concrete implementations. `OrderService` depends on an `OrderRepository` interface, not on `JpaOrderRepository`; the implementation is injected (this is what Spring's dependency injection enables), which also makes it easy to replace with a mock in tests.

```java
class OrderService {
  private final OrderRepository repository; // interface

  OrderService(OrderRepository repository) { // implementation injected
    this.repository = repository;
  }
}
```

These are guidelines, not strict rules: applied blindly, they lead to over-engineering (an interface for every class, dozens of tiny classes). Apply them where change is actually expected.

## Hexagonal architecture

> **In short**: Ports and Adapters: the business core (domain + use cases) defines interfaces (ports), and the technical code (REST, JPA, Kafka) implements or calls them (adapters). Dependencies point inwards, so the core is testable alone and technologies can be swapped; worth it for a rich domain, overkill for simple CRUD.

Also called **Ports and Adapters** (Alistair Cockburn). The goal: isolate the business logic from technical details (framework, database, messaging, UI), so it can be tested alone and the technology can change without touching it.

```
 REST controller ──┐                               ┌──> JPA repository
 Kafka consumer  ──┼──> [in port] CORE [out port] ─┼──> Kafka producer
 Scheduled job   ──┘     (domain + application)    └──> HTTP client

 driving adapters                                  driven adapters
```

The layers, from the center outwards:

- **Domain**: entities, value objects and business rules (`Order`, `Money`, "an order can't be shipped before payment"). Plain Java, no Spring, JPA or Jackson annotations.
- **Application**: the use cases (`PlaceOrderService`), which orchestrate the domain and call the outbound ports. Transactions are usually handled here.
- **Ports**: interfaces owned by the core.
  - **Inbound (driving) ports**: what the application offers, e.g. `PlaceOrderUseCase`, implemented by the application layer.
  - **Outbound (driven) ports**: what the application needs, e.g. `OrderRepository`, `PaymentGateway`, implemented by adapters.
- **Adapters**: the technical code around the core.
  - **Driving adapters** call an inbound port: REST controller, Kafka consumer, scheduled job, CLI.
  - **Driven adapters** implement an outbound port: JPA repository, Kafka producer, HTTP client to an external API.

The key rule: **dependencies point inwards**. Adapters depend on the core, never the reverse. The core defines the `OrderRepository` interface, the infrastructure implements it with JPA: this is Dependency Inversion (the D of SOLID) applied to the whole application.

```
com.shop.order
├── domain        # Order, Money, business rules
├── application   # PlaceOrderService, ports (in/ out/)
└── adapter
    ├── in/web    # OrderController
    └── out/jpa   # OrderJpaAdapter, OrderEntity
```

Benefits: the domain and use cases are tested with plain unit tests (ports replaced by mocks or in-memory implementations), and swapping a technology (PostgreSQL to MongoDB, REST to Kafka) only means writing a new adapter.

Cost: more interfaces and mapping code (a JPA `OrderEntity` separate from the domain `Order`, DTOs for the REST layer). Worth it for a rich business domain; overkill for a simple CRUD application.
