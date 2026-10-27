## What CQRS is

> **In short**: CQRS (Command Query Responsibility Segregation) separates the write side (commands that change state, through the domain model and its rules) from the read side (queries that return data shaped for the screens, without the domain). The domain model is made to enforce rules, not to read efficiently: each side gets the model that suits it.

- **Command**: changes state, returns nothing or an id (`PlaceOrderCommand`, `ShipOrderCommand`). Goes through the domain: load the aggregate, apply the business rules, save.
- **Query**: returns data, changes nothing (`GetOrderSummariesQuery`). Reads a model shaped for the screen: only the needed columns, joins across aggregates, pagination.

```java
// write side: the domain model and its rules
class ShipOrderService implements ShipOrderUseCase {
  public void shipOrder(OrderId id) {
    Order order = orders.findById(id).orElseThrow();
    order.ship(); // business rule: must be paid
    orders.save(order);
  }
}

// read side: a view read directly, without loading Order
class GetOrderSummariesService implements GetOrderSummariesUseCase {
  public List<OrderSummary> getSummaries(CustomerId customer) {
    return orderSummaries.findByCustomer(customer); // projection or SQL, with the product names
  }
}
```

It comes from CQS (Command Query Separation, Bertrand Meyer): a method either changes state or returns a value, never both. CQRS (Greg Young) applies the same idea at the architecture level, to the models.

Not to be confused with event sourcing: CQRS is often combined with it, but works with a plain database. In a hexagonal architecture, the read side is just a dedicated outbound port returning views (see [Hexagonal architecture](HEXAGONAL_ARCHITECTURE.md#common-pitfalls)).

## CQRS levels and eventual consistency

> **In short**: CQRS ranges from two models on the same tables (no extra infrastructure) to a separate read database updated by events. A separate store scales reads and fits search screens, but the read side becomes eventually consistent: just after a command, a query may still return the old data.

| Level                            | Read model                                                                     | Consistency                                            | Cost                                      |
| -------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------ | ----------------------------------------- |
| Same tables                      | projections / SQL queries on the write tables                                  | immediate                                              | none: just separate code paths            |
| Read tables in the same database | views, materialized views, denormalized tables updated in the same transaction | immediate (or at refresh time for a materialized view) | low                                       |
| Separate store                   | Elasticsearch, Redis, another database, updated from events                    | eventual                                               | high: synchronization, replay, monitoring |

With a separate store, the write side publishes an event (`OrderPlaced`, `OrderShipped`), and a projector updates the read model:

```text
command ──> domain ──> write DB ──(event, e.g. Kafka)──> projector ──> read DB <── queries
```

- The event must be published reliably with the write: same problem as any database + broker write (see [Kafka outbox pattern](../back-end/kafka/KAFKA_OUTBOX.md)).
- The projector must be idempotent (an event can be delivered twice), and the read model can be rebuilt by replaying the events.
- The UI must handle stale reads: show the result of the command instead of reloading, or poll until the view is updated.

When to use it: reads and writes with very different needs (complex search screens, dashboards, read traffic far above write traffic). The first level costs almost nothing and is often worth it; a separate store is only justified by real performance or search needs, since eventual consistency complicates the whole application.
