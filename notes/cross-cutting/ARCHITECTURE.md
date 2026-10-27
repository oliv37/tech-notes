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

Details (layers, communication between them, rules): [Hexagonal architecture](HEXAGONAL_ARCHITECTURE.md).
