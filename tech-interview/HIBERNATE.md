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
