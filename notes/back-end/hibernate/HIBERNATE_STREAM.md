## Why stream large results

> **In short**: `getResultList()` / `findAll()` load every row into memory before returning: a `List` of 1,000,000 entities before the first one is processed. Streaming reads the results row by row through a database cursor, fetched by chunks (the fetch size), so the memory stays constant whatever the number of rows. It works at every level: JDBC, JPA / Hibernate and Spring Data.

Typical uses: exporting a table to a CSV file, recomputing a value for every row, migrating or reindexing data (e.g. into Elasticsearch).

|                     | `List`                           | Stream                             |
| ------------------- | -------------------------------- | ---------------------------------- |
| Memory              | all rows at once                 | one chunk (fetch size) at a time   |
| First row processed | after the whole result is loaded | as soon as the first chunk arrives |
| Connection          | released after the query         | held until the stream is closed    |
| Transaction         | optional for a read              | required in practice (cursor)      |

## Streaming with plain JDBC

> **In short**: The JDBC `ResultSet` is already a cursor: `next()` moves row by row. But by default, drivers like PostgreSQL's load the whole result into memory during `executeQuery()`. With `setFetchSize(500)`, a forward-only result set and auto-commit off, the driver uses a server-side cursor and fetches 500 rows per round trip. In Spring, `JdbcTemplate.queryForStream()` does the same, inside a transaction.

```java
try (Connection connection = dataSource.getConnection()) {
  connection.setAutoCommit(false); // PostgreSQL: the cursor only lives inside a transaction

  try (PreparedStatement statement = connection.prepareStatement(
      "select id, customer, total from orders where status = ?",
      ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY)) {
    statement.setFetchSize(500); // rows per round trip
    statement.setString(1, "PAID");

    try (ResultSet rs = statement.executeQuery()) {
      while (rs.next()) { // fetches the next 500 rows when the current chunk is consumed
        csvWriter.write(out, rs.getLong("id"), rs.getString("customer"), rs.getBigDecimal("total"));
      }
    }
  }
  connection.commit();
} // the pool (HikariCP) restores auto-commit when the connection is returned
```

- **PostgreSQL** (pgJDBC): streams only with a fetch size > 0, auto-commit off and a forward-only result set; otherwise the fetch size is ignored and the whole result is loaded.
- **MySQL** (Connector/J): streams row by row with `setFetchSize(Integer.MIN_VALUE)`, or by chunks with `useCursorFetch=true` on the URL and a positive fetch size.
- No entities and no persistence context: the lightest option, at the cost of mapping by hand.

With Spring's `JdbcTemplate`:

```java
@Transactional(readOnly = true) // auto-commit off, so the PostgreSQL cursor works
public void exportPaidOrders(Writer out) {
  jdbcTemplate.setFetchSize(500); // better set once when creating the JdbcTemplate bean
  try (Stream<OrderRow> rows = jdbcTemplate.queryForStream(
      "select id, customer, total from orders where status = ?",
      (rs, i) -> new OrderRow(rs.getLong("id"), rs.getString("customer"), rs.getBigDecimal("total")),
      "PAID")) {
    rows.forEach(row -> csvWriter.write(out, row));
  }
}
```

`jdbcTemplate.query(sql, rowCallbackHandler, args)` streams too, with a callback per row instead of a `Stream`.

## Streaming with JPA / Hibernate and Spring Data

> **In short**: `getResultStream()` (JPA 2.2) returns a `Stream` backed by Hibernate's `ScrollableResults`, a JDBC cursor. Spring Data supports it with a `Stream<T>` return type on a repository method, inside a transaction and closed with try-with-resources. The fetch size is set with a query hint.

```java
// plain JPA
try (Stream<Order> orders = entityManager
    .createQuery("select o from Order o where o.status = :status", Order.class)
    .setParameter("status", OrderStatus.PAID)
    .setHint(HibernateHints.HINT_FETCH_SIZE, 500)
    .getResultStream()) {
  orders.forEach(order -> csvWriter.write(out, order));
}
```

Hibernate's native API is `query.scroll(ScrollMode.FORWARD_ONLY)`, returning a `ScrollableResults` (`next()` / `get()`), which `getResultStream()` uses underneath.

**Spring Data**: a repository method returning `Stream<T>`.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

  @QueryHints({
      @QueryHint(name = HibernateHints.HINT_FETCH_SIZE, value = "500"), // rows per round trip
      @QueryHint(name = HibernateHints.HINT_READ_ONLY, value = "true")}) // no dirty checking snapshot
  Stream<Order> streamByStatus(OrderStatus status);
}

@Transactional(readOnly = true) // required: the stream keeps the connection open
public void exportPaidOrders(Writer out) {
  try (Stream<Order> orders = orderRepository.streamByStatus(OrderStatus.PAID)) { // closes the cursor
    orders.forEach(order -> {
      csvWriter.write(out, order);
      entityManager.detach(order); // otherwise the persistence context keeps every entity
    });
  }
}
```

- Without a surrounding transaction, Spring Data throws `InvalidDataAccessApiUsageException`: the connection would go back to the pool while the cursor is still being read. So the stream can't be returned from the service to a controller after the transaction ends.
- `hibernate.jdbc.fetch_size` sets a default fetch size for all queries.

**With a `StatelessSession`**: no persistence context, so the streamed entities aren't kept anywhere and no `detach()` is needed (see [Hibernate StatelessSession](HIBERNATE.md#hibernate-statelesssession)).

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);

// opens the session and a transaction (needed for the PostgreSQL cursor),
// commits at the end, rolls back on exception, closes the session (Hibernate 6.3+)
sessionFactory.inStatelessTransaction(session -> {
  try (Stream<Order> orders = session
      .createSelectionQuery("from Order o where o.status = :status", Order.class)
      .setParameter("status", OrderStatus.PAID)
      .setFetchSize(500)
      .getResultStream()) {
    orders.forEach(order -> csvWriter.write(out, order)); // each entity can be garbage collected after use
  }
});
```

Handled by hand, the transaction needs a `try` / `catch` that calls `tx.rollback()` before rethrowing: otherwise an exception skips the `commit()`, and the transaction is only cleaned up when the connection goes back to the pool.

- The entities are plain detached objects: no dirty checking (modifying one does nothing), and lazy associations aren't loaded on access: fetch them in the query (`join fetch` on a to-one association) or explicitly with `session.fetch(...)`.
- Opened this way, the session manages its own connection and transaction, outside Spring's `@Transactional`.

## Pitfalls

> **In short**: A stream must be closed and read inside a transaction. With JPA, the streamed entities pile up in the persistence context, so memory grows anyway unless they're detached, read-only or replaced by projections. Operations that need all rows (`sorted()`, `collect()`) cancel the benefit, and a long stream holds a connection and a transaction for the whole processing.

- **Always close the stream** (try-with-resources): it holds a `ResultSet` and a connection; a forgotten stream leaks a connection from the pool.
- **The fetch size decides whether it really streams**: without it (or with auto-commit on, for PostgreSQL), the driver loads everything and the stream only iterates over a list in memory. Check with a large table and a memory profile, not with 10 rows.
- **The persistence context grows**: each streamed entity stays managed. `detach()` each one or `clear()` every N rows, use the read-only hint, stream DTO projections (not managed), or use a `StatelessSession`, which has no persistence context (see [Hibernate StatelessSession](HIBERNATE.md#hibernate-statelesssession)).
- **Stream operations that need everything** (`sorted()`, `distinct()`, `collect(toList())`) load it all again: sort and filter in the query.
- **Lazy associations** accessed in the loop trigger one query per row (N+1): fetch the needed to-one associations in the query, or use a projection.
- **A long transaction** holds a connection and a database snapshot for the whole processing, and a failure means starting over. For very long jobs, keyset pagination (see [Pagination and sorting](../spring/SPRING_DATA.md#pagination-and-sorting)) releases the connection between pages and can resume from the last key.
