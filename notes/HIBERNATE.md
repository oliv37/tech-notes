## Hibernate creation and update timestamps

> **In short**: `@CreationTimestamp` and `@UpdateTimestamp` (Hibernate annotations) fill timestamp columns automatically, on insert and on each update detected by dirty checking. They are bypassed by bulk JPQL updates, native SQL and JDBC; Spring Data auditing is the alternative that also records the user.

`@CreationTimestamp` and `@UpdateTimestamp` (Hibernate annotations, not JPA) fill timestamp columns automatically:

```java
@Entity
public class Order {

  @CreationTimestamp
  @Column(updatable = false) // never modified afterwards
  private Instant createdAt; // set on INSERT

  @UpdateTimestamp
  private Instant updatedAt; // set on INSERT, then on each UPDATE
}
```

- Types: `Instant` (safest, no time zone issue), `LocalDateTime`, `OffsetDateTime`...
- JVM time by default; `@CreationTimestamp(source = SourceType.DB)` uses the database's `current_timestamp` (consistent across instances with slightly different clocks). Both are shortcuts for `@CurrentTimestamp(event = ...)`.
- `updatedAt` only changes when Hibernate runs an `UPDATE`, i.e. when dirty checking finds a modification.
- Bypassed by anything not going through the entity: bulk JPQL `UPDATE`, native SQL, JDBC.

Standard JPA alternative, without Hibernate annotations: callbacks on the entity.

```java
@PrePersist
void onCreate() { createdAt = updatedAt = Instant.now(); }

@PreUpdate
void onUpdate() { updatedAt = Instant.now(); }
```

In a Spring application, Spring Data auditing (`@CreatedDate`, `@CreatedBy`...) is often preferred, see [Spring Data](SPRING_DATA.md).

## Hibernate fetch types: LAZY vs EAGER

> **In short**: `EAGER` loads an association together with the entity, `LAZY` on first access, with an extra query. To-one associations are `EAGER` by default and to-many ones `LAZY`: make everything `LAZY` and fetch what each use case needs explicitly.

The fetch type defines **when** an association is loaded:

- **`EAGER`**: loaded at the same time as the owning entity, whether it's used or not.
- **`LAZY`**: loaded on first access (e.g. `book.getAuthor().getName()`, `author.getBooks().size()`), with an extra query.

Defaults depend on the association type:

| Association                 | Default |
| --------------------------- | ------- |
| `@ManyToOne`, `@OneToOne`   | `EAGER` |
| `@OneToMany`, `@ManyToMany` | `LAZY`  |

```java
@ManyToOne(fetch = FetchType.LAZY)
private Author author;
```

How `LAZY` works: Hibernate doesn't put the real object in the field, but a placeholder that runs the query when accessed: a proxy (generated subclass of `Author`) for a single entity, a `PersistentBag` / `PersistentSet` for a collection. Calling `getId()` on a proxy doesn't trigger the query, since the id is already known.

The catch: the query needs an open session. Accessing a lazy association after the transaction is closed (e.g. in a controller, or when serializing to JSON) throws a `LazyInitializationException`. Fix: load what's needed inside the transaction (`JOIN FETCH`, `@EntityGraph`), or map to a DTO there. Spring Boot hides the problem by default with Open Session in View (`spring.jpa.open-in-view=true`, logs a warning), which keeps the session open until the view is rendered, at the cost of queries running outside any transaction.

Recommendation: map everything as `LAZY` (`@ManyToOne(fetch = FetchType.LAZY)`) and decide per query what to load. `EAGER` is set on the mapping, so it applies to every query and can't be turned off afterwards (except with a fetch graph, see below), and it causes N+1 queries with JPQL (see below).

Caveat: `LAZY` is only a hint in the JPA spec. Hibernate ignores it on the inverse side of a `@OneToOne` (`mappedBy`, see below), unless bytecode enhancement is enabled: it has to query the other table anyway to know whether to put `null` or a proxy in the field.

## Hibernate owning side of a relationship (mappedBy)

> **In short**: In a bidirectional relationship, the owning side (the `@ManyToOne`, holding the foreign key) is the only one read to write the FK; the other side declares `mappedBy` and is a mirror for navigation. Set both sides in Java (helper method); cascades are a separate mechanism that works from either side.

A bidirectional relationship has two Java fields but a single foreign key in the database. JPA needs to know which field drives that column: the **owning side**.

```java
@Entity
public class Book {
  @ManyToOne
  @JoinColumn(name = "author_id") // owning side: the FK is in the book table
  private Author author;
}

@Entity
public class Author {
  @OneToMany(mappedBy = "author") // inverse side: "see the author field in Book"
  private List<Book> books;
}
```

- **Owning side**: the field mapped to the FK column, read by Hibernate to write it on `INSERT` / `UPDATE`. In a `@OneToMany` / `@ManyToOne` pair, always the `@ManyToOne` (the FK lives in the "many" table).
- **Inverse side** (`mappedBy`): a mirror, useful for navigation in Java, ignored when writing the FK. `mappedBy` references the **Java field name** in the other entity, not a column name.

Consequence: only changes on the owning side update the FK.

```java
author.getBooks().add(book); // ignored for the FK: author_id stays null
book.setAuthor(author);      // this is what writes author_id
```

Hence the common helper method (`addBook(book)`) that sets both sides, keeping the database and the in-memory objects consistent.

Not to be confused with cascading: with `@OneToMany(mappedBy = "author", cascade = CascadeType.PERSIST)`, persisting an author also persists the books in its list (`INSERT` into `book`). Cascading propagates entity operations (persist, merge, remove) and works from either side, but the FK value still comes from `book.getAuthor()`: without `book.setAuthor(author)`, the books are inserted with `author_id` null. Same for `orphanRemoval = true`: removing a book from the list deletes it, even from the inverse side.

Without `mappedBy`, Hibernate sees two independent relationships: on a `@OneToMany`, it creates an extra join table (`author_books`), usually not what's wanted.

A unidirectional `@OneToMany` (no `author` field in `Book`) is possible: with no field on the other side there's no `mappedBy`, so the `@OneToMany` becomes the owning side.

- Without `@JoinColumn`: a join table (`author_books`), the JPA default.
- With `@JoinColumn(name = "author_id")`: the FK stays in `book`, but is written from the collection. Hibernate may insert the book with `author_id` null, then run an extra `UPDATE` per element (`@JoinColumn(nullable = false)` puts the FK in the `INSERT`). Removing a book from the list sets `author_id` to null (or deletes it with `orphanRemoval = true`).

Preferred options: a `@ManyToOne` alone (simplest and most efficient, the books of an author come from a query like `findByAuthor(author)`), or a bidirectional relationship with `mappedBy` when navigation from `Author` is needed.

Other relationships:

- `@ManyToMany`: the owning side is the one without `mappedBy`, it manages the join table.
- `@OneToOne`: the owning side is the one whose table holds the FK.

## Hibernate N+1 problem

> **In short**: Loading N entities with 1 query, then accessing a lazy association, triggers 1 extra query per entity. Fix it with `JOIN FETCH`, an entity graph, or `@BatchSize` (which also works with pagination); switching to `EAGER` doesn't fix it.

The N+1 problem happens when loading a list of N entities triggers 1 query for the list, then 1 extra query per entity to load a lazy association.

```java
@Entity
public class Author {
  @OneToMany(mappedBy = "author") // LAZY by default
  private List<Book> books;
}

List<Author> authors = authorRepository.findAll(); // 1 query
for (Author author : authors) {
  author.getBooks().size(); // 1 query per author → N queries
}
```

It often goes unnoticed in development (small datasets) and becomes a performance issue in production. Enabling SQL logs (`spring.jpa.show-sql=true`) makes it visible.

Main fixes:

- `JOIN FETCH` in JPQL: `select a from Author a join fetch a.books`, loads authors and books in a single query
- `@EntityGraph(attributePaths = "books")` on a Spring Data repository method, same result without writing the query
- `@BatchSize(size = 50)` on the association: not a single query, but loads the associations by batches of 50 (`where author_id in (...)`), turning N queries into N/50

Caveat: fetching a collection with pagination (`Pageable`) makes Hibernate load all rows and paginate in memory (warning `HHH000104`); `@BatchSize` is the better option in that case.

Switching to `FetchType.EAGER` is not a fix: the association is then always loaded, even when not needed, and `findAll()` can still trigger N+1 queries.

The problem also happens with `@ManyToOne`, which is `EAGER` by default (like `@OneToOne`). A JPQL query such as `findAll()` doesn't join the association, so Hibernate runs one extra query per author right after loading the books, even if `getAuthor()` is never called. With `FetchType.LAZY` (recommended), the queries are triggered when accessing `book.getAuthor()`. In both cases, the number of extra queries is the number of distinct authors, since an already loaded author is reused from the persistence context. Same fixes (`JOIN FETCH`, `@EntityGraph`), except that `@BatchSize` goes on the target entity class (`Author`), not on the field.

## JPA entity graph

> **In short**: An entity graph lists the associations to load with the entity in the same query, without changing the `LAZY` mapping: `@EntityGraph(attributePaths = "books")` on a Spring Data method, or a reusable `@NamedEntityGraph` on the entity.

An entity graph lists the associations to load with the entity, in the same query (`LEFT JOIN`), without changing the mapping (which stays `LAZY`). In Spring Data, it's declared with `@EntityGraph` on a repository method:

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @EntityGraph(attributePaths = "books")
  List<Author> findByCountry(String country); // authors + their books, one query

  @EntityGraph(attributePaths = {"books", "books.publisher"}) // nested path
  Optional<Author> findWithBooksById(Long id);

  @EntityGraph(attributePaths = "books")
  @Query("select a from Author a where a.name like :prefix%")
  List<Author> search(String prefix); // also works with @Query

  @Override
  @EntityGraph(attributePaths = "books")
  List<Author> findAll(); // overrides the inherited method
}
```

- Each method chooses what it loads: `findById()` stays lazy, `findWithBooksById()` loads the books (`With...` is ignored by the derived query parser, it just names the variant).
- Same pagination caveat as `JOIN FETCH` when the graph contains a collection (see N+1 above).
To reuse a graph on several methods without repeating its attributes, declare it once on the entity with `@NamedEntityGraph`, and reference it by name:

```java
@Entity
@NamedEntityGraph(
    name = "Author.booksAndPublisher",
    attributeNodes = @NamedAttributeNode(value = "books", subgraph = "books"),
    subgraphs = @NamedSubgraph(name = "books", attributeNodes = @NamedAttributeNode("publisher")))
public class Author { ... }

public interface AuthorRepository extends JpaRepository<Author, Long> {
  @EntityGraph("Author.booksAndPublisher")
  List<Author> findByCountry(String country);

  @EntityGraph("Author.booksAndPublisher")
  Optional<Author> findWithBooksById(Long id);
}
```

`attributePaths` is shorter for a one-off graph; `@NamedEntityGraph` keeps a shared graph in one place (subgraphs handle nested paths like `books.publisher`).

An entity graph can be applied in two modes, which only differ for the attributes **not** listed in the graph:

- **Fetch graph** (`jakarta.persistence.fetchgraph`): listed attributes are loaded, all others are treated as `LAZY`, even if mapped as `EAGER`.
- **Load graph** (`jakarta.persistence.loadgraph`): listed attributes are loaded, others keep their mapped fetch type (so an `EAGER` `@ManyToOne` is still loaded).

```java
@EntityGraph(attributePaths = "books", type = EntityGraphType.LOAD)
List<Author> findAll();
```

In Spring Data, `@EntityGraph` uses `FETCH` by default. In Hibernate, basic attributes (columns) are always loaded whatever the mode (unless bytecode enhancement is enabled): in practice, the difference only applies to associations.

## Hibernate JDBC batching

> **In short**: With `hibernate.jdbc.batch_size`, Hibernate groups statements of the same type into JDBC batches, one round trip per batch instead of one per statement. Order inserts and updates so they can be grouped; `GenerationType.IDENTITY` disables batching for inserts, prefer a sequence.

By default, Hibernate sends one SQL statement per round trip: inserting 1,000 entities means 1,000 calls to the database. JDBC batching groups the statements of the same type and sends them together (`PreparedStatement.addBatch()` / `executeBatch()`), one round trip per batch. Disabled by default:

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
# group statements by table, so they can be batched together
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

`order_inserts` matters as soon as several tables are involved: without it, alternating `INSERT author`, `INSERT book`, `INSERT author`... breaks the batch at each change of statement.

```java
@Transactional
public void importBooks(List<Book> books) {
  for (int i = 0; i < books.size(); i++) {
    entityManager.persist(books.get(i));
    if (i % 50 == 49) {        // same value as batch_size
      entityManager.flush();   // sends the batch
      entityManager.clear();   // empties the persistence context to keep memory flat
    }
  }
}
```

Pitfalls:

- **`GenerationType.IDENTITY` disables insert batching**: Hibernate needs the id generated by the database right after each `INSERT`, so it has to run them one by one. Use `SEQUENCE` with an `allocationSize` (e.g. 50): Hibernate reserves 50 ids in a single sequence call and assigns them in memory.
- **Spring Data `save()` with an assigned id** (a UUID set in the constructor, a business key) considers the entity as existing, and calls `merge()`, which runs a `SELECT` before each insert. Fix: a `@Version` field (`null` means new) or implementing `Persistable.isNew()`.
- **Check it really works**: SQL logs show each statement even when batched. Use `hibernate.generate_statistics=true` (number of JDBC batches) or a proxy like datasource-proxy. With PostgreSQL, `reWriteBatchedInserts=true` on the JDBC URL also rewrites a batch into a single multi-row `INSERT`.

## Hibernate StatelessSession

> **In short**: A `StatelessSession` has no persistence context: no first-level cache, dirty checking, lazy loading or cascades, each call runs its SQL immediately. Suited for bulk processing of large volumes without filling memory; Hibernate 7 adds `insertMultiple()` for batched inserts.

A `StatelessSession` is a Hibernate session **without persistence context**: no first-level cache, no dirty checking, no lazy loading, no cascades (see [Hibernate cache](HIBERNATE_CACHE.md)). Each operation is an explicit command that runs its SQL immediately: `insert()`, `update()`, `delete()`, `upsert()`, `get()`. Entities returned are plain detached objects: modifying one does nothing until `update()` is called. Associations that aren't fetched in the query stay unloaded, unless loaded explicitly with `fetch()`.

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);

try (StatelessSession session = sessionFactory.openStatelessSession()) {
  Transaction tx = session.beginTransaction();
  session.insertMultiple(lines); // Hibernate 7: inserts in JDBC batches, nothing kept in memory
  tx.commit();
}
```

Since Hibernate 7, `hibernate.jdbc.batch_size` is ignored by a stateless session (`insert()` in a loop runs one statement at a time): batching is explicit with `insertMultiple()`, `updateMultiple()`, `deleteMultiple()`. A stateless session also uses the L2 cache by default since Hibernate 7 (`CacheMode.IGNORE` to bypass it).

Main use: batch processing and imports of large volumes, where a regular session would grow until it runs out of memory and slow down with dirty checking (instead of the `flush()` / `clear()` workaround). Opened this way, it manages its own connection and transaction, outside Spring's `@Transactional`.
