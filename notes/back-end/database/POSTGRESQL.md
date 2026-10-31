## What PostgreSQL brings

> **In short**: PostgreSQL is an open-source relational database, ACID and transactional, known for its standard SQL and its extensibility. Beyond tables and joins, it has rich types (`jsonb`, arrays, ranges, `uuid`), upsert with `ON CONFLICT`, `RETURNING`, CTEs and window functions, full-text search, and extensions (PostGIS, `pg_trgm`, `pgvector`). It often replaces a second store: a JSON document store, a search engine for simple cases, a queue.

| Feature                            | Example                                                                                                            |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `jsonb` (binary JSON, indexable)   | `where attributes ->> 'color' = 'red'`, `where attributes @> '{"color": "red"}'`                                   |
| Arrays                             | `tags text[]`, `where 'sale' = any(tags)`                                                                          |
| Upsert                             | `insert ... on conflict (email) do update set name = excluded.name`                                                |
| `RETURNING`                        | `insert into orders (...) values (...) returning id, created_at`: no second query                                  |
| CTE and window functions           | `with ... as (...)`, `row_number() over (partition by customer_id order by created_at desc)`                       |
| Full-text search                   | `to_tsvector('english', title) @@ to_tsquery('headphones')`                                                        |
| Identity columns                   | `id bigint generated always as identity` (standard, replaces `serial`)                                             |
| `uuid`                             | `uuidv7()` since PostgreSQL 18: time-ordered, friendlier to B-tree indexes than random v4                          |
| Extensions                         | PostGIS (geo), `pg_trgm` (fuzzy search, `like '%x%'` with an index), `pgvector` (embeddings), `pg_stat_statements` |
| `LISTEN` / `NOTIFY`, `SKIP LOCKED` | simple pub/sub; a job queue with `select ... for update skip locked`                                               |

- **`jsonb` vs `json`**: `json` stores the text as is, `jsonb` a parsed binary form: slower to write, faster to query, indexable with GIN. Use `jsonb`, for attributes that vary, not to avoid designing columns.
- **One process per connection**: each connection is a server process (a few MB), so thousands of connections are expensive: use a pool (HikariCP in the application, PgBouncer in front of the database) with a small size.

**ACID**: the four guarantees of a transaction, e.g. a transfer of 100 from account A to account B (two `update` in one transaction).

| Guarantee | Meaning | For the transfer | In PostgreSQL |
|---|---|---|---|
| **Atomicity** | all or nothing | never a debit without the credit, even after a crash between the two updates | an error or a `rollback` cancels everything; the new row versions never become visible |
| **Consistency** | from one valid state to another, respecting the constraints | a balance that can't go negative, an account B that must exist | `check`, `not null`, `unique`, foreign keys: the transaction fails if one is violated |
| **Isolation** | concurrent transactions don't see each other's intermediate states | no other transaction sees "A debited, B not credited yet" | MVCC snapshots, with the isolation levels (see below) |
| **Durability** | once committed, the data survives a crash | the confirmed transfer isn't lost if the server goes down a second later | the WAL (write-ahead log) is flushed to disk before the `commit` returns, and replayed after a crash |

Consistency only covers the constraints declared in the database; business rules not expressed as constraints stay the application's job. The opposite model, BASE (eventually consistent), is the one of many distributed NoSQL stores and of Elasticsearch.

## MVCC, VACUUM and transactions

> **In short**: PostgreSQL uses MVCC: an `UPDATE` doesn't modify the row in place, it writes a new version and marks the old one dead; each transaction sees a snapshot, so readers never block writers. Dead rows are cleaned by `VACUUM` (autovacuum), which a long-running transaction prevents: tables and indexes then bloat. The default isolation level is `READ COMMITTED`.

- **Row versions**: an `UPDATE` is an insert of the new version plus a delete of the old one. A table updated very often (a counter, a status) accumulates dead rows.
- **Autovacuum** removes dead rows that no transaction can still see, and updates the planner statistics (`ANALYZE`). It runs in the background; on very active tables, its thresholds are tuned per table.
- **Long transactions are the main enemy**: a transaction open for hours (a forgotten `idle in transaction` connection, a long batch, a stream left open) keeps every row version created since its start alive, for every table. Watch `pg_stat_activity`, set `idle_in_transaction_session_timeout`.
- **HOT updates**: an update that touches no indexed column and fits in the same page doesn't update the indexes: another reason not to index everything.

| Isolation level            | Behavior in PostgreSQL                                                                                                                         |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `READ COMMITTED` (default) | each statement sees the data committed before it starts                                                                                        |
| `REPEATABLE READ`          | the whole transaction sees the snapshot of its first statement; a concurrent update of the same row fails with a serialization error, to retry |
| `SERIALIZABLE`             | as if the transactions ran one after another (SSI): more serialization errors, to retry                                                        |

`READ UNCOMMITTED` behaves like `READ COMMITTED`: PostgreSQL never shows uncommitted data.

## Indexes

> **In short**: B-tree is the default index (equality, ranges, sorting). A composite index serves the queries that filter on its first columns. Partial indexes (`WHERE`) index only the useful rows, expression indexes index a computed value (`lower(email)`), `INCLUDE` enables index-only scans. GIN indexes `jsonb`, arrays and full text, BRIN very large tables ordered by time. Foreign keys aren't indexed automatically, and every index slows down the writes.

```sql
-- composite: serves "where customer_id = ?" and "where customer_id = ? and created_at > ?",
-- not "where created_at > ?" alone
create index orders_customer_created_idx on orders (customer_id, created_at);

-- partial: only the pending orders, a small fraction of the table
create index orders_pending_idx on orders (created_at) where status = 'PENDING';

-- expression: the query must use the same expression
create unique index users_email_lower_idx on users (lower(email));
select * from users where lower(email) = lower(?);

-- covering: the query reads only the index (index-only scan)
create index orders_customer_total_idx on orders (customer_id) include (total);

-- GIN for jsonb: serves "attributes @> '{"color": "red"}'"
create index products_attributes_idx on products using gin (attributes);

-- created without blocking writes on a production table
create index concurrently orders_status_idx on orders (status);
```

| Type             | For                                                                                     | Example                            |
| ---------------- | --------------------------------------------------------------------------------------- | ---------------------------------- |
| B-tree (default) | `=`, `<`, `between`, `order by`, `like 'abc%'`                                          | ids, dates, emails                 |
| GIN              | "contains": `jsonb` (`@>`), arrays, full text, trigrams (`like '%abc%'` with `pg_trgm`) | attributes, tags, search           |
| BRIN             | very large tables whose order follows a column                                          | logs, events by date: a tiny index |
| Hash             | equality only                                                                           | rarely better than B-tree          |

Pitfalls:

- **Foreign keys aren't indexed**: `orders.customer_id` referencing `customers` has no index unless created; joins and deletes on `customers` (checking the references) then scan `orders`.
- **An index the query can't use**: a function on the column (`where date(created_at) = ...` with an index on `created_at`), a leading wildcard (`like '%abc'`), a type mismatch, or the second column of a composite index alone.
- **A low-selectivity column** (`status` with 3 values) is rarely worth a full index: the planner prefers a sequential scan; a partial index on the rare value is better.
- **Unused indexes** cost on every write and in memory: `pg_stat_user_indexes.idx_scan = 0` lists them.
- **Index-only scans** need an up-to-date visibility map, i.e. a recently vacuumed table.

## EXPLAIN ANALYZE

> **In short**: `EXPLAIN` shows the plan the planner chose, `EXPLAIN ANALYZE` runs the query and adds the actual times and row counts. The key reading: which scan on each table (sequential, index, index-only, bitmap), which join algorithm, and the gap between estimated and actual rows, which means stale statistics. `pg_stat_statements` finds the queries worth analyzing.

```sql
explain (analyze, buffers)
select * from orders where customer_id = 42 and created_at > now() - interval '30 days';
```

```text
Index Scan using orders_customer_created_idx on orders  (cost=0.43..12.50 rows=8 width=64)
                                                        (actual time=0.031..0.054 rows=11 loops=1)
  Index Cond: ((customer_id = 42) AND (created_at > (now() - '30 days'::interval)))
  Buffers: shared hit=5
Planning Time: 0.120 ms
Execution Time: 0.081 ms
```

- **`cost`**: the planner's estimate (arbitrary units, startup..total); **`actual time`**: measured, in ms; **`rows`**: estimated vs actual; **`loops`**: how many times the node ran (multiply the time by it).
- **`Buffers`**: pages read from the cache (`hit`) or from disk (`read`): included by default with `ANALYZE` since PostgreSQL 18.
- `EXPLAIN ANALYZE` really runs the query: wrap an `UPDATE` / `DELETE` in `begin; ... rollback;`.

| Node                                     | Meaning                                                                                                |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `Seq Scan`                               | reads the whole table: fine for small tables or when most rows match                                   |
| `Index Scan`                             | walks the index, then reads each matching row in the table                                             |
| `Index Only Scan`                        | everything comes from the index (`INCLUDE`, visibility map)                                            |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | collects the matching rows from one or more indexes, then reads the pages in order: medium selectivity |
| `Nested Loop`                            | for each row on one side, a lookup on the other: good with few rows and an index                       |
| `Hash Join`                              | builds a hash table of one side: large joins on equality                                               |
| `Merge Join`                             | merges two sorted inputs                                                                               |
| `Sort` with `external merge Disk`        | the sort didn't fit in `work_mem` and spilled to disk                                                  |

What to look for: a `Seq Scan` on a large table with few matching rows (a missing or unusable index), estimates far from reality (`rows=10` vs `actual rows=500000`: run `ANALYZE`, or raise the statistics target), a `Nested Loop` with many `loops`, a sort or hash spilling to disk.

**`pg_stat_statements`** (extension): aggregates every query by shape, with its number of calls, total and mean time. Sorting by `total_exec_time` gives the queries that cost the most overall, often not the slowest single one.

## Partitioning and other performance techniques

> **In short**: Declarative partitioning splits a large table into partitions by range (by month), list or hash; queries filtering on the partition key only read the relevant partitions (partition pruning), and old data is removed by dropping a partition instead of a huge `DELETE`. It's for very large tables (hundreds of millions of rows), not a default. Other levers: connection pooling, keyset pagination, batching and `COPY` for bulk loads.

```sql
create table events (
  id bigint generated always as identity,
  created_at timestamptz not null,
  payload jsonb,
  primary key (id, created_at)              -- the primary key must include the partition key
) partition by range (created_at);

create table events_2026_10 partition of events
  for values from ('2026-10-01') to ('2026-11-01');

-- reads only events_2026_10 (partition pruning)
select count(*) from events where created_at >= '2026-10-15' and created_at < '2026-10-16';

-- removes a whole month instantly, without bloat
drop table events_2025_10;
```

- **Benefits**: pruning, smaller indexes per partition, instant purge (`drop` / `detach partition`), vacuum per partition.
- **Costs**: primary keys and unique constraints must include the partition key; a query without the partition key reads every partition; partitions must be created in advance (a job, or `pg_partman`); too many partitions slow down planning.

Other techniques:

| Technique                                               | Why                                                                                      |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Connection pool sized small (HikariCP, PgBouncer)       | a process per connection: dozens of connections perform better than hundreds             |
| Keyset pagination (`where id > ? order by id limit 20`) | `offset 100000` reads and discards 100,000 rows                                          |
| Batch inserts, `COPY` for bulk loads                    | one round trip and one transaction for many rows; `COPY` is the fastest way to load data |
| `ANALYZE` after a large load                            | the planner uses statistics; stale ones give bad plans                                   |
| Materialized views                                      | precomputed aggregates, refreshed with `refresh materialized view concurrently`          |
| Read replicas                                           | streaming replication offloads reads (with a slight replication lag)                     |

## PostgreSQL pitfalls

> **In short**: `count(*)` reads the whole table, `NOT IN` with a `NULL` returns nothing, unquoted identifiers are lowercased, `timestamp` has no time zone (use `timestamptz`), some `ALTER TABLE` lock the table, and sequences have gaps. Most surprises come from MVCC and long transactions (see above).

- **`count(*)` is slow on large tables**: no stored row count, because of MVCC each transaction may see a different number. For an approximate count: `reltuples` in `pg_class`; for a UI, avoid exact totals (`Slice` instead of `Page`).
- **`NOT IN` with a `NULL`**: `where id not in (select customer_id from orders)` returns no row if one `customer_id` is `NULL`. Use `NOT EXISTS`.
- **Identifiers**: `create table Orders` creates `orders`; `"Orders"` with quotes is case-sensitive, and must then always be quoted.
- **`timestamp` vs `timestamptz`**: `timestamp` stores a date without time zone, interpreted differently by each client; `timestamptz` stores an instant (converted to UTC). Use `timestamptz`.
- **`varchar(n)` vs `text`**: same storage and performance; `varchar(n)` only adds a length check.
- **Locking DDL**: most `ALTER TABLE` take an exclusive lock, and wait behind running queries while blocking the new ones. Adding a column with a constant default is instant (since PostgreSQL 11), but changing a column type rewrites the table. Set `lock_timeout` for migrations, create indexes `concurrently`, add constraints `not valid` then `validate constraint`.
- **Sequences have gaps**: a rolled-back insert consumes its value; never rely on ids being consecutive.
- **`ON CONFLICT`** needs a unique index or constraint on the target columns.

**Unique constraints and `NULL`**: by default, a unique constraint accepts several `NULL`, since `NULL = NULL` is unknown, not true. On several columns, one `NULL` in the tuple is enough for the row to never conflict.

```sql
-- intent: one active account per email; but (email, NULL) never equals (email, NULL)
unique (email, deleted_at)                     -- ❌ two active accounts with the same email are accepted

-- fix: a partial unique index on the active rows only
create unique index users_active_email_idx on users (email) where deleted_at is null;
```

To forbid duplicates whether a column is filled or `NULL`, **`NULLS NOT DISTINCT`** (PostgreSQL 15+) treats `NULL` as an ordinary value, equal to the other `NULL`s:

```sql
create table settings (
  tenant_id bigint,                            -- NULL = global setting
  key text not null,
  value text,
  unique nulls not distinct (tenant_id, key)
);

insert into settings values (1, 'theme', 'dark');
insert into settings values (1, 'theme', 'light');    -- ❌ duplicate (1, theme)
insert into settings values (null, 'theme', 'light'); -- ✅ the global setting
insert into settings values (null, 'theme', 'blue');  -- ❌ duplicate (NULL, theme)
```

- `ON CONFLICT (tenant_id, key)` then also detects the conflicts on the `NULL` rows, so the upsert of a global setting works.
- Before PostgreSQL 15: two partial unique indexes, `(tenant_id, key) where tenant_id is not null` and `(key) where tenant_id is null`; `ON CONFLICT` must then repeat the predicate (`on conflict (key) where tenant_id is null`).
- An index on `coalesce(tenant_id, -1)` also works, but needs a sentinel value that is never a real one, and the same expression in every query.
- Other databases differ: SQL Server allows a single `NULL` in a unique constraint by default.
