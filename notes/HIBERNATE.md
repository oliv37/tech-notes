## Hibernate fetch types: LAZY vs EAGER

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

## JPA entity graph: fetch graph vs load graph

An entity graph can be applied in two modes, which only differ for the attributes **not** listed in the graph:

- **Fetch graph** (`jakarta.persistence.fetchgraph`): listed attributes are loaded, all others are treated as `LAZY`, even if mapped as `EAGER`.
- **Load graph** (`jakarta.persistence.loadgraph`): listed attributes are loaded, others keep their mapped fetch type (so an `EAGER` `@ManyToOne` is still loaded).

```java
@EntityGraph(attributePaths = "books", type = EntityGraphType.LOAD)
List<Author> findAll();
```

In Spring Data, `@EntityGraph` uses `FETCH` by default. In Hibernate, basic attributes (columns) are always loaded whatever the mode (unless bytecode enhancement is enabled): in practice, the difference only applies to associations.
