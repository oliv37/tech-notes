## Spring Data JPA repositories

> **In short**: A repository is an interface extending `JpaRepository<Entity, Id>`; Spring Data generates its implementation at startup (a proxy over `SimpleJpaRepository`), with CRUD methods, pagination and queries derived from method names. `save()` calls `persist` for a new entity and `merge` for an existing one.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
  List<Order> findByCustomer(String customer); // derived query (see below)
}
```

| Interface | Adds |
|---|---|
| `Repository<T, ID>` | nothing: a marker, only the declared methods exist |
| `CrudRepository` / `ListCrudRepository` | `save`, `findById`, `findAll`, `count`, `deleteById`... (`Iterable` / `List` results) |
| `PagingAndSortingRepository` / `ListPagingAndSortingRepository` | `findAll(Pageable)`, `findAll(Sort)` |
| `JpaRepository` | all of the above, plus JPA specifics: `flush`, `saveAndFlush`, `deleteAllInBatch`, `getReferenceById` |

- No implementation to write: the proxy is generated from the interface, and a missing or invalid derived query fails at startup.
- **`save()`**: `persist()` if the entity is new (null id, or null `@Version`), otherwise `merge()`, which first runs a `SELECT` to load the current state. With an id assigned by the application (UUID), the entity looks existing: implement `Persistable.isNew()`, or use a `@Version` field.
- **Transactions**: each repository method runs in its own transaction (`SimpleJpaRepository` is `@Transactional(readOnly = true)`, its write methods `@Transactional`). A `@Transactional` service method is required to make several writes atomic; for reads it's optional, but needed for lazy loading after the repository call (unless Open Session In View, enabled by default in Spring Boot but discouraged) and for `Stream` results.
- `findById()` runs a `SELECT` and returns an `Optional`; `getReferenceById()` returns a lazy proxy without a query, enough to set an association (`order.setCustomer(customerRepository.getReferenceById(7L))`).
- Custom code: an interface `OrderRepositoryCustom` + a class `OrderRepositoryCustomImpl` (with an `EntityManager`), and `OrderRepository extends JpaRepository<...>, OrderRepositoryCustom`.

## Derived query methods

> **In short**: Spring Data builds the query from the method name: `findByCustomerAndStatus(String customer, Status status)` becomes `where customer = ? and status = ?`. Keywords cover comparisons, `Like`, `In`, `OrderBy`, `First` / `Top`; the return type can be an entity, an `Optional`, a `List`, a `Stream`, a count or a boolean.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
  List<Order> findByCustomerAndStatus(String customer, OrderStatus status);
  List<Order> findByTotalGreaterThanOrderByCreatedAtDesc(BigDecimal total);
  List<Order> findByStatusIn(Collection<OrderStatus> statuses);
  List<Order> findByCustomerContainingIgnoreCase(String part); // like %part%, case-insensitive
  Optional<Order> findFirstByCustomerOrderByCreatedAtDesc(String customer); // the latest one
  List<Order> findTop10ByStatus(OrderStatus status);
  List<Order> findByShippingAddressCity(String city); // nested property: shippingAddress.city
  boolean existsByReference(String reference);
  long countByStatus(OrderStatus status);
  long deleteByStatus(OrderStatus status);  // loads then deletes each entity (needs a transaction)
}
```

| Keyword | Example | SQL |
|---|---|---|
| `And`, `Or` | `findByCustomerAndStatus` | `customer = ? and status = ?` |
| `Between`, `LessThan`, `GreaterThanEqual` | `findByTotalBetween(min, max)` | `total between ? and ?` |
| `IsNull`, `IsNotNull` | `findByShippedAtIsNull()` | `shipped_at is null` |
| `Like`, `StartingWith`, `Containing` | `findByCustomerStartingWith("A")` | `customer like 'A%'` |
| `In`, `NotIn` | `findByStatusIn(list)` | `status in (...)` |
| `True`, `False` | `findByPaidTrue()` | `paid = true` |
| `IgnoreCase` | `findByCustomerIgnoreCase` | `upper(customer) = upper(?)` |
| `OrderBy...Asc / Desc` | `findByStatusOrderByCreatedAtDesc` | `order by created_at desc` |

- Return types: `Order` (null if none, exception if several), `Optional<Order>`, `List`, `Stream<Order>` (to close, in a transaction), `Page` / `Slice` with a `Pageable` parameter (see below).
- Good for simple queries; beyond two or three criteria, the name becomes unreadable: use `@Query` or a specification (see below).
- `deleteBy...` loads the entities and deletes them one by one (lifecycle callbacks, cascades); for a bulk delete, a `@Modifying @Query`.

## @Query: JPQL and native queries

> **In short**: `@Query` declares the query explicitly on a repository method: JPQL by default (on entities and fields), or native SQL with `nativeQuery = true`. Parameters are bound by name (`:customer`); update and delete queries need `@Modifying`, and run directly in the database, bypassing the persistence context.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

  @Query("select o from Order o where o.customer = :customer and o.total > :min")
  List<Order> findBigOrders(@Param("customer") String customer, @Param("min") BigDecimal min);

  @Query("select o from Order o join fetch o.lines where o.id = :id") // loads the lines in the same query
  Optional<Order> findWithLines(Long id);

  @Query(value = "select * from orders where created_at > now() - interval '7 days'", nativeQuery = true)
  List<Order> findLastWeek();

  @Modifying(clearAutomatically = true)
  @Transactional
  @Query("update Order o set o.status = :status where o.createdAt < :before")
  int expireOld(OrderStatus status, Instant before); // number of rows updated
}
```

- JPQL uses entity and field names (`Order`, `o.customer`), checked at startup; native SQL uses tables and columns, PostgreSQL-specific features included, but isn't validated nor portable.
- `@Param` is optional when compiling with `-parameters` (set by Spring Boot): the parameter name is used.
- **`@Modifying`**: required for `update` / `delete`. The query goes straight to the database: entities already loaded in the persistence context keep their old values. `clearAutomatically = true` clears it after the query, `flushAutomatically = true` flushes pending changes before. Entity callbacks, auditing and `@Version` aren't applied.
- Pagination with `@Query`: add a `Pageable` parameter; for a complex query, provide the `countQuery`.

## Pagination and sorting

> **In short**: A `Pageable` parameter (`PageRequest.of(page, size, sort)`) adds `limit` / `offset` and an `order by` to the query. `Page` also runs a count query to know the total; `Slice` doesn't, it only knows whether there's a next page. For deep pagination on large tables, keyset scrolling (`Window`) avoids the cost of the offset.

```java
Page<Order> findByStatus(OrderStatus status, Pageable pageable);
Slice<Order> findByCustomer(String customer, Pageable pageable);

Page<Order> page = orderRepository.findByStatus(OrderStatus.PAID,
    PageRequest.of(0, 20, Sort.by("createdAt").descending()));
page.getContent();       // the 20 orders
page.getTotalElements(); // total count: an extra "select count(*)"
page.getTotalPages();
```

In a controller, Spring MVC builds the `Pageable` from the request: `GET /api/orders?page=0&size=20&sort=createdAt,desc`.

```java
@GetMapping("/api/orders")
public PagedModel<OrderDto> list(@PageableDefault(size = 20, sort = "createdAt") Pageable pageable) {
  return new PagedModel<>(orderRepository.findAll(pageable).map(OrderDto::from));
}
```

| | `Page` | `Slice` | `Window` (scroll API) |
|---|---|---|---|
| Queries | content + count | content only (fetches `size + 1` rows) | content only |
| Knows | total elements and pages | whether there's a next slice | whether there's more, and the position to continue from |
| Pagination | offset | offset | offset or **keyset** |
| Suited for | numbered pages | "load more", infinite scroll | large tables, deep pages |

- Pagination with `join fetch` on a collection: Hibernate fetches all rows and paginates in memory (warning `HHH000104`); paginate the ids first, or use `@BatchSize` (see [Hibernate](../hibernate/HIBERNATE.md)).
- Serializing a `Page` directly as JSON exposes an unstable structure (`PageImpl`, warning since Spring Data 3.3): return a `PagedModel`, or `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)`, or a custom DTO.
- Always sort on a unique column (or add the id): otherwise the order between equal values isn't stable, and rows can appear on two pages or none.

**Keyset pagination** (scroll API, Spring Data 3.1+): `offset 100000` makes the database read and skip 100,000 rows; keyset pagination continues from the last row seen instead, at constant cost.

| | Offset | Keyset |
|---|---|---|
| Cost of a distant page | grows: the database reads and skips every row before the offset | constant: `where (created_at, id) < (?, ?)` on the index |
| Navigation | any page (N → N+10) | next / previous only |
| Total count | possible (`count`) | no |
| Rows inserted while browsing | shifts: duplicates or skipped rows | stable: continues from the last row seen |
| Use | numbered pages, small tables | infinite scroll, large volumes, batch processing |

The sort must be unique as a whole: a business column (`created_at`) plus a unique tie-breaker (`id`), with an index covering them in the sort order.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
  Window<Order> findFirst20ByStatusOrderByCreatedAtDesc(OrderStatus status, KeysetScrollPosition position);
}

WindowIterator<Order> orders = WindowIterator
    .of(position -> orderRepository.findFirst20ByStatusOrderByCreatedAtDesc(
        OrderStatus.PAID, (KeysetScrollPosition) position))
    .startingAt(ScrollPosition.keyset()); // from the beginning

while (orders.hasNext()) {
  process(orders.next()); // fetches the next window when needed
}
```

```sql
-- first window
select ... where status = 'PAID' order by created_at desc, id desc limit 21
-- next ones: after the (created_at, id) of the last row seen
select ... where status = 'PAID' and (created_at < ? or (created_at = ? and id < ?))
order by created_at desc, id desc limit 21
```

- Spring Data adds the primary key to the sort (unique order), and fetches `limit + 1` rows to know whether there's more.
- Manual loop: `window.positionAt(window.size() - 1)` gives the position after the last row, `window.isLast()` tells whether it's over.
- REST API: return the keys of the last row as an opaque cursor (`((KeysetScrollPosition) position).getKeys()`, e.g. Base64-encoded), and rebuild the position on the next call with `ScrollPosition.forward(keys)`.
- Requirements: non-nullable sort columns, sort columns included in the result (also in projections), an index matching the sort (`status, created_at, id`).
- Next / previous only: no jump to page 50 and no total, so suited for infinite scroll and batch processing, not numbered pages.

## Projections

> **In short**: A projection loads only some columns instead of the whole entity: an interface with getters (interface projection), or a record / DTO class (class projection), declared as the return type of a repository method. Lighter queries, no dirty checking, and no `LazyInitializationException` when the result leaves the transaction.

```java
// interface projection: only id, customer and total are selected
public interface OrderSummary {
  Long getId();
  String getCustomer();
  BigDecimal getTotal();
}

// class projection: a record, filled through its constructor
public record OrderDto(Long id, String customer, BigDecimal total) {}

public interface OrderRepository extends JpaRepository<Order, Long> {
  List<OrderSummary> findByStatus(OrderStatus status);
  List<OrderDto> findByCustomer(String customer);
  <T> List<T> findByCreatedAtAfter(Instant date, Class<T> type); // dynamic: the caller picks the projection

  @Query("select new com.example.shop.OrderDto(o.id, o.customer, o.total) from Order o where o.total > :min")
  List<OrderDto> findBig(BigDecimal min); // JPQL constructor expression
}
```

- **Closed interface projection** (getters matching fields, `getCustomerName()` for a nested `customer.name`): Spring Data selects only those columns.
- **Open projection** (`@Value("#{target.firstName + ' ' + target.lastName}")`): computed in Java, the whole entity is loaded, no gain on the query.
- **Class projection**: constructor parameters matching the field names; with `@Query`, a JPQL `select new ...` (full class name), or a native query mapped to an interface projection.
- Read-only: projections aren't managed entities, changing them updates nothing.
- Typical use: lists and search results returned by an API, where loading the full entity with its associations would be wasteful.

## Spring Data JPA auditing

> **In short**: `@EnableJpaAuditing` and an entity listener fill `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy` and `@LastModifiedBy` automatically, the user coming from an `AuditorAware` bean (e.g. from Spring Security). Unlike Hibernate's `@CreationTimestamp`, it also tracks who made the change.

Spring Data fills audit fields automatically: when (`@CreatedDate`, `@LastModifiedDate`) and **who** (`@CreatedBy`, `@LastModifiedBy`).

```java
@Configuration
@EnableJpaAuditing
class AuditConfig {

  @Bean
  AuditorAware<String> auditorAware() { // who is the current user
    return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
        .map(Authentication::getName);
  }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {
  @CreatedDate @Column(updatable = false) private Instant createdAt;
  @LastModifiedDate private Instant updatedAt;
  @CreatedBy @Column(updatable = false) private String createdBy;
  @LastModifiedBy private String updatedBy;
}

@Entity
public class Order extends AuditableEntity { ... }
```

How it works: `AuditingEntityListener` is a JPA entity listener, called on `@PrePersist` and `@PreUpdate`; it sets the dates and asks `AuditorAware` for the user. By default, `@LastModifiedDate` / `@LastModifiedBy` are also set on creation.

To control the time in tests, declare a `DateTimeProvider` (e.g. based on a fixed `Clock`) and reference it: `@EnableJpaAuditing(dateTimeProviderRef = "auditingDateTimeProvider")`.

Compared to Hibernate's `@CreationTimestamp` / `@UpdateTimestamp` (see [Hibernate](../hibernate/HIBERNATE.md)):

|                       | `@CreationTimestamp` (Hibernate)            | `@CreatedDate` (Spring Data)                          |
| --------------------- | ------------------------------------------- | ----------------------------------------------------- |
| Mechanism             | Hibernate value generator, applied at flush | JPA entity listener (`@PrePersist` / `@PreUpdate`)    |
| Setup                 | none, the annotation is enough              | `@EnableJpaAuditing` + `@EntityListeners`             |
| Who made the change   | no                                          | yes, `@CreatedBy` / `@LastModifiedBy` + `AuditorAware` |
| Database time         | yes (`source = SourceType.DB`)              | no, application time                                  |
| Time controllable in tests | no, JVM clock                          | yes, `DateTimeProvider`                               |
| Other stores          | no, JPA / Hibernate only                    | yes (MongoDB, JDBC...)                                |

Same limit for both: they only work when Hibernate writes the entity (not bulk JPQL updates, native SQL or JDBC), and the modification date only changes when the entity actually changed.

## Dynamic queries: specifications and query by example

> **In short**: When the criteria depend on the request (an optional filter form), a `Specification` builds the `where` clause from composable predicates (JPA Criteria API), passed to a repository extending `JpaSpecificationExecutor`. Query by Example matches on the non-null fields of a sample entity, simpler but limited to equality and string matching.

```java
public interface OrderRepository extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> {}

public final class OrderSpecs {
  static Specification<Order> hasStatus(OrderStatus status) {
    return (root, query, cb) -> cb.equal(root.get("status"), status);
  }
  static Specification<Order> totalAbove(BigDecimal min) {
    return (root, query, cb) -> cb.greaterThan(root.get("total"), min);
  }
  static Specification<Order> customerContains(String text) {
    return (root, query, cb) -> cb.like(cb.lower(root.get("customer")), "%" + text.toLowerCase() + "%");
  }
}

// in the service: only the filters actually provided
List<Specification<Order>> specs = new ArrayList<>();
if (filter.status() != null) specs.add(hasStatus(filter.status()));
if (filter.minTotal() != null) specs.add(totalAbove(filter.minTotal()));
if (filter.customer() != null) specs.add(customerContains(filter.customer()));

Page<Order> result = orderRepository.findAll(Specification.allOf(specs), pageable); // AND of all
```

- Specifications compose with `and`, `or`, `not`, and work with pagination and sorting.
- `root.get("status")` uses strings: the JPA static metamodel (`Order_.status`, generated by `hibernate-jpamodelgen`) makes it type-safe.
- **Query by Example**: `orderRepository.findAll(Example.of(probe, ExampleMatcher.matching().withIgnoreCase().withStringMatcher(CONTAINING)))`: no code per criterion, but no ranges (`>`, `between`), no `or` between fields.
- Alternatives: Querydsl (type-safe fluent API, `QuerydslPredicateExecutor`), or a custom repository method with the Criteria API or a JPQL string built by hand.
