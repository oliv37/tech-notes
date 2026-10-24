## First-level cache (persistence context)

The first-level cache is the persistence context itself: every `EntityManager` (Hibernate `Session`) keeps a map of the entities it has loaded, keyed by type and id. It's always active and can't be disabled. In Spring, an `EntityManager` lives as long as the transaction, so the cache only lasts for one transaction and is never shared between requests or users.

```java
@Transactional
public void example() {
  Author a1 = em.find(Author.class, 1L); // SELECT
  Author a2 = em.find(Author.class, 1L); // no query: taken from the persistence context
  assert a1 == a2;                       // same Java instance
}
```

- **Identity guarantee**: within a transaction, a given row is always represented by the same Java object, whatever the way it was loaded (`find()`, a query, a lazy association).
- **Only lookups by id** skip the database (`find()`, `getReference()`, loading a `@ManyToOne`). A JPQL query always runs its SQL; Hibernate then returns the instances already in the context for the rows it finds, without overwriting their in-memory state.
- **Dirty checking**: the context keeps a snapshot of each loaded entity. At flush (before commit, or before a query that could be affected), Hibernate compares each entity with its snapshot and sends an `UPDATE` for the modified ones. This is why changing a managed entity is enough to update it, without calling `save()`.

Pitfalls: in a batch processing thousands of entities in one transaction, the context grows until it runs out of memory and dirty checking gets slower; call `flush()` then `clear()` regularly. In a long transaction, entities can also become stale, since the context doesn't see changes committed by others.

## Second-level cache (L2)

The second-level cache is shared by all `EntityManager`s of the application (it belongs to the `EntityManagerFactory`), so an entity loaded by one request can be reused by the next ones without hitting the database. It's inactive by default: it needs a cache provider (JCache with Ehcache, Caffeine, Infinispan, Hazelcast...) and must be enabled per entity.

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=jcache
```

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country { ... }
```

When loading by id, Hibernate first looks in the persistence context, then in the L2 cache, and only then queries the database. The L2 cache doesn't store Java objects but their "dehydrated" state (an array of column values), from which Hibernate builds a new instance for each persistence context. Like the first-level cache, it's only used for lookups by id, not for JPQL queries.

The concurrency strategy defines how the cache stays consistent with writes:

- **`READ_ONLY`**: the entity is never modified (an update throws an exception). The fastest, for reference data (countries, currencies).
- **`NONSTRICT_READ_WRITE`**: the cache entry is invalidated after the commit; a short window where another transaction can read stale data. For rarely modified data where that's acceptable.
- **`READ_WRITE`**: soft locks on the entry during the update guarantee a transaction never reads stale data. The usual choice for modified entities.
- **`TRANSACTIONAL`**: the cache takes part in the transaction (JTA); only with providers supporting it, rarely used.

Collections (`@OneToMany`) are cached separately with `@Cache` on the field, and only store the ids of the elements: the target entity must be cached too, otherwise each element is loaded with a query (N+1).

## Query cache

The query cache stores the result of a JPQL query for given parameters. It's disabled by default and enabled per query:

```properties
spring.jpa.properties.hibernate.cache.use_query_cache=true
```

```java
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Country> findByContinent(String continent);
```

It only stores the ids of the results; the entities are then read from the L2 cache, so they must be cached too (otherwise one query per result). Its invalidation is coarse: Hibernate records the last update time of each table, and any insert, update or delete on a table invalidates **all** cached queries reading it. It's only worth it for frequently run queries on tables that rarely change.

## When to use the L2 cache, and its pitfalls

The L2 cache pays off for data read often and modified rarely: reference data, configuration, catalogs. For frequently modified entities, invalidations and locks cost more than they save.

- **Several instances**: with a local cache (Ehcache, Caffeine), each instance has its own copy, and an update on one instance isn't seen by the others. Requires a distributed or replicated cache (Infinispan, Hazelcast, Redis via Redisson) or accepting stale data with a short TTL.
- **Changes outside Hibernate**: another application, a SQL script or a database trigger modifying the tables is invisible to the cache, which keeps serving the old values until expiration.
- **Bulk queries**: a JPQL `UPDATE` / `DELETE` invalidates the whole region of the entity, and a native SQL update invalidates the entire L2 cache, since Hibernate can't know which tables it touched (unless declared with `addSynchronizedEntityClass()`).
- **Memory and TTL**: size limits and expiration are configured in the provider (e.g. `ehcache.xml`), not in Hibernate.

Alternative: Spring's `@Cacheable` on a service method caches the final result (often a DTO), with explicit keys and eviction. Simpler to reason about than the L2 cache, and independent of Hibernate.
