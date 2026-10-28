## What Elasticsearch is

> **In short**: Elasticsearch is a distributed search and analytics engine built on Apache Lucene. It stores JSON documents, exposes a REST API, and makes them searchable in near real time (about 1 second after indexing), with full-text search ranked by relevance. It's usually a secondary store fed from the main database, not the source of truth.

Elasticsearch stores **JSON documents** in **indices**, and is queried over **HTTP** (port `9200`) with a JSON query language. Each index is split into shards spread across the nodes of a cluster (see below), so it scales horizontally.

What it does that a relational database does badly:

- **Full-text search**: `WHERE name LIKE '%wireless headphones%'` scans the whole table, finds no "headphone" (singular) and ranks nothing. Elasticsearch finds documents containing the words in any order, handles plurals, synonyms and typos (`hedphones`), and sorts the results by relevance.
- **Speed on large volumes**: an inverted index (see below) finds the documents containing a word without reading them all.
- **Aggregations**: counts and statistics computed on the search results (facets of an e-commerce site: "Brand: Sony (12), Bose (8)", price ranges).

Typical uses:

- Search in an application: products, articles, autocomplete.
- Logs and observability: the **Elastic Stack** (formerly ELK: Elasticsearch, Logstash to ingest, Kibana to visualize, plus Beats/Elastic Agent to collect).
- Analytics, geo search, and vector search (semantic search, kNN).

Vocabulary compared with SQL:

| SQL                | Elasticsearch                  |
| ------------------ | ------------------------------ |
| table              | index                          |
| row                | document (JSON, with an `_id`) |
| column             | field                          |
| schema             | mapping                        |
| `SELECT ... WHERE` | search with the Query DSL      |

Why it isn't the main database: no multi-document transactions, no joins, writes visible only after a refresh, and some operations (changing a field's type) require rebuilding the index. The usual setup keeps the data in PostgreSQL and copies it into Elasticsearch for search: a CQRS read model (see [CQRS](../../cross-cutting/CQRS.md)).

OpenSearch is a fork created by AWS in 2021 after Elastic changed the license; it has the same concepts and a very close API.

## Inverted index and analysis

> **In short**: Elasticsearch doesn't search the documents, it searches an inverted index: for each term, the list of documents containing it, like the index at the end of a book. Before indexing, a text field goes through an analyzer (tokenizer + filters: lowercase, stemming, synonyms) that turns it into terms, and the search text goes through the same analyzer. A `text` field is analyzed for full-text search; a `keyword` field keeps the exact value for filters, sorting and aggregations.

**The inverted index**: three product names, analyzed in lowercase with plurals reduced (stemming):

```text
1: "Wireless Headphones Sony"
2: "Sony Wireless Speaker"
3: "Bose headphone"
```

| Term       | Documents |
| ---------- | --------- |
| `bose`     | 3         |
| `headphon` | 1, 3      |
| `sony`     | 1, 2      |
| `speaker`  | 2         |
| `wireless` | 1, 2      |

A search for "wireless headphones" is analyzed into `wireless` and `headphon`, looked up in the table, then the documents are ranked: document 1 contains both terms, so it comes first. No document is read during the search.

**The analyzer**, applied to each `text` field at indexing time and to the query text at search time:

1. **Character filters**: clean the raw text (strip HTML tags).
2. **Tokenizer**: split into tokens (`standard`: on spaces and punctuation).
3. **Token filters**: transform the tokens: `lowercase`, stop words (`the`, `a`), stemming (`headphones` → `headphon`), synonyms (`tv` = `television`), `asciifolding` (`café` → `cafe`).

```http
POST /_analyze
{ "analyzer": "english", "text": "The Wireless Headphones" }
```

Result: `wireless`, `headphon` (`the` removed as a stop word, plural reduced). The default `standard` analyzer only splits and lowercases; language analyzers (`english`, `french`) add stop words and stemming.

**`text` vs `keyword`**:

|            | `text`                                  | `keyword`                                     |
| ---------- | --------------------------------------- | --------------------------------------------- |
| Indexed as | analyzed terms (`wireless`, `headphon`) | the exact value (`Wireless Headphones Sony`)  |
| For        | full-text search (`match`)              | exact filters (`term`), sorting, aggregations |
| Examples   | product name, description, article body | status, email, id, tag, brand for facets      |

- A `term` query (exact, not analyzed) on a `text` field usually finds nothing: `term: { "name": "Sony" }` looks for `Sony`, but the index contains `sony`. Use `match` on `text`, `term` on `keyword`.
- Sorting or aggregating on a `text` field is refused by default; it needs a `keyword` field. A field often has both (multi-fields, see below).
- `keyword` is case-sensitive: `SHIPPED` and `shipped` are two different values.

## Components: cluster, node, index, shard, segment

> **In short**: A cluster is a set of nodes (Elasticsearch instances), one of which is the elected master that manages the cluster state. An index is split into primary shards, each a Lucene index, with replica copies on other nodes for availability and read throughput. A document goes to one shard, chosen by a hash of its `_id`, which is why the number of primary shards is fixed at creation. Each shard is made of immutable segments.

```text
cluster "shop" ── index "products": 3 primary shards (P), 1 replica each (R)

 node-1            node-2            node-3
┌──────────┐      ┌──────────┐      ┌──────────┐
│ P0   R1  │      │ P1   R2  │      │ P2   R0  │
└──────────┘      └──────────┘      └──────────┘
```

- **Cluster**: the nodes sharing the same `cluster.name`. One master-eligible node is elected **master**: it manages the cluster state (index creation, mappings, which shard lives on which node), not the search traffic.
- **Node**: one Elasticsearch process (a JVM). Its roles: `master` (eligible to be elected), `data` (holds shards; can be split into hot/warm/cold tiers for logs), `ingest` (transforms documents before indexing). Every node can coordinate a request: receive it, send it to the shards, merge the results.
- **Index**: a logical collection of documents with one mapping, physically split into shards.
- **Primary shard**: a part of the index, a complete Lucene index in itself. Each document lives in exactly one primary shard: `shard = hash(_routing) % number_of_primary_shards`, with `_routing` = `_id` by default. Changing the number of primaries would move every document, hence it's fixed at creation (changing it means `_split`, `_shrink` or a reindex).
- **Replica shard**: a copy of a primary, never on the same node as its primary. It takes over if the node of the primary fails, and serves searches too. The number of replicas can be changed at any time.
- **Segment**: a shard is made of immutable segments. New documents go into a new segment at each refresh; segments are merged in the background, and a deleted document is only marked as deleted until the merge (see writing below).

**Cluster health**: `green` (all shards assigned), `yellow` (all primaries assigned, but some replicas aren't: the classic case of a single-node cluster with 1 replica, since a replica can't be on its primary's node), `red` (a primary is missing: part of the data is unavailable).

**The path of a request**:

- **Write**: the coordinating node computes the shard, sends the document to the primary, which indexes it and forwards it to its replicas before acknowledging.
- **Search** (query then fetch): the coordinating node sends the query to one copy (primary or replica) of every shard; each returns the ids and scores of its best results; the coordinator merges them into the global top N, then fetches those documents from the shards.

Sizing: one primary shard per index by default (since 7.0). Too many small shards cost memory and coordination; a common guideline is shards of about 10 to 50 GB.

## Mapping

> **In short**: The mapping is the schema of an index: the type of each field (`text`, `keyword`, `long`, `date`...) and how it's analyzed. Elasticsearch can guess it from the first document (dynamic mapping), but an explicit mapping avoids wrong guesses. The type of an existing field can't be changed: the data must be reindexed into a new index, and an alias lets the application switch to it without downtime.

**Explicit mapping** for a product index:

```http
PUT /products_v1
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name":      { "type": "text", "analyzer": "english",
                     "fields": { "keyword": { "type": "keyword" } } },
      "brand":     { "type": "keyword" },
      "price":     { "type": "scaled_float", "scaling_factor": 100 },
      "inStock":   { "type": "boolean" },
      "createdAt": { "type": "date" }
    }
  }
}
```

- **Common types**: `text`, `keyword`, `long` / `integer` / `double`, `scaled_float` (a price stored as an integer number of cents), `boolean`, `date`, `object`, `nested` (see the pitfalls below), `geo_point`, `dense_vector`.
- **Multi-fields**: `name` is indexed twice, as `text` for the search (`match` on `name`) and as `keyword` for sorting and aggregations (`name.keyword`).
- **`_source`**: the original JSON is stored as is and returned in the results. The mapping only defines how fields are indexed for the search.

**Dynamic mapping**: without a mapping, the first document that contains a field decides its type.

- A string becomes `text` with a `keyword` sub-field (the `name` / `name.keyword` pair, values over 256 characters not indexed as keyword).
- A number becomes `long` or `float`, a string like `"2026-10-28"` becomes `date`.
- Wrong guesses are frequent: a first price `10` gives a `long` field, and `10.5` is then indexed as `10` (while `_source` still shows `10.5`); a zip code `"37000"` becomes `text`.
- The `dynamic` setting controls unknown fields: `true` (default: added to the mapping), `false` (kept in `_source` but not searchable), `strict` (the document is rejected).

**Changing a mapping**: a new field can be added at any time, but the type or analyzer of an existing field can't change, since the data is already indexed with the old one. The procedure:

1. Create a new index with the new mapping (`products_v2`).
2. Copy the data: `POST /_reindex` from `products_v1` to `products_v2` (or re-index from the source database).
3. Switch the alias atomically.

```http
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add":    { "index": "products_v2", "alias": "products" } }
  ]
}
```

The application always uses the alias `products`, never the real index name: the switch is invisible to it. Using an alias from the start costs nothing and makes every later mapping change possible without downtime.

## Searching: Query DSL, bool, score and aggregations

> **In short**: Searches are JSON queries (Query DSL). Full-text queries (`match`) analyze the search text and compute a relevance score; term-level queries (`term`, `range`) match exact values. A `bool` query combines them: `must` and `should` contribute to the score, `filter` and `must_not` only include or exclude (no score, cached, faster). Results are ranked by a BM25 score, and aggregations compute facets and statistics on the matching documents.

```http
GET /products/_search
{
  "query": {
    "bool": {
      "must":   [ { "match": { "name": { "query": "wireless hedphones", "fuzziness": "AUTO" } } } ],
      "filter": [ { "term":  { "inStock": true } },
                  { "range": { "price": { "lte": 300 } } } ],
      "should": [ { "match_phrase": { "name": "noise cancelling" } } ]
    }
  },
  "aggs": {
    "brands": { "terms": { "field": "brand" } },
    "prices": { "range": { "field": "price",
                           "ranges": [ { "to": 100 }, { "from": 100, "to": 200 }, { "from": 200 } ] } }
  },
  "from": 0,
  "size": 20
}
```

The products whose name matches "wireless headphones" (despite the typo), in stock and at most 300, those containing "noise cancelling" ranked higher; plus the count per brand and per price range.

```json
{
  "hits": {
    "total": { "value": 42, "relation": "eq" },
    "hits": [ { "_id": "1", "_score": 7.3, "_source": { "name": "Wireless Headphones Sony", "brand": "Sony", ... } } ]
  },
  "aggregations": {
    "brands": { "buckets": [ { "key": "Sony", "doc_count": 12 }, { "key": "Bose", "doc_count": 8 } ] }
  }
}
```

**Query types**:

| | Full-text | Term-level |
|---|---|---|
| Queries | `match`, `multi_match` (several fields, `name^3` to boost one), `match_phrase` (words in order) | `term`, `terms`, `range`, `exists`, `prefix` |
| Search text | analyzed, like the field | used as is |
| Fields | `text` | `keyword`, numbers, dates, booleans |

**`bool`**:

| Clause | Must match | Affects the score |
|---|---|---|
| `must` | yes | yes |
| `filter` | yes | no (cached) |
| `should` | no if there's a `must` or `filter` (it only boosts); at least one otherwise | yes |
| `must_not` | must not match | no |

Query context vs filter context: "how well does it match?" (score) vs "does it match?" (yes/no). Exact criteria (status, category, price range, dates) go into `filter`: no score to compute, and the result is cached.

**Relevance score (BM25)**, from three factors:

- **Term frequency**: the more often the term appears in the field, the higher the score (with a saturation: 10 occurrences don't count 10 times more).
- **Inverse document frequency**: a rare term weighs more than a common one ("headphones" more than "wireless" if most products are wireless).
- **Field length**: a match in a short field counts more ("Sony headphones" beats a long description mentioning headphones once).

**Aggregations**:

- **Bucket** aggregations group documents: `terms` (by brand), `range` (price ranges), `date_histogram` (per day).
- **Metric** aggregations compute a value: `avg`, `min`, `max`, `sum`, `cardinality` (approximate count of distinct values).
- They apply to the documents matching the query; `"size": 0` returns only the aggregations, without hits.
- They need `keyword`, numeric or date fields, not `text`.

## Writing: near real time, refresh, translog and updates

> **In short**: An indexed document is written to an in-memory buffer and to the translog (durable), but only becomes searchable at the next refresh, every second by default: Elasticsearch is near real time. Segments are immutable, so an update is a delete plus a full reindex of the document. Concurrent writes are protected by optimistic concurrency (`if_seq_no` / `if_primary_term`), or by an external version taken from the database.

**Indexing**:

```http
PUT /products/_doc/1                      # create or replace the document with id 1
{ "name": "Wireless Headphones Sony", "brand": "Sony", "price": 199.99, "inStock": true }

POST /products/_update/1                  # partial update
{ "doc": { "price": 179.99 } }

DELETE /products/_doc/1
```

For many documents, the `_bulk` API sends them in one request (one action line + one document line each): much faster than one request per document.

**The write path**:

```text
document ──> in-memory buffer + translog ──(refresh, 1 s)──> new segment: searchable
                                          ──(flush)────────> segments committed to disk, translog cleared
                                          ──(merge)────────> small segments merged, deleted documents purged
```

- **Translog**: an append-only log written to disk (fsync) before the write is acknowledged, on the primary and the replicas. After a crash, it's replayed: an acknowledged document isn't lost.
- **Refresh**: turns the buffer into a new searchable segment, every second by default (`refresh_interval`). Between indexing and refresh, a search doesn't find the document. A `GET` by id is real time, though: it finds it immediately.
- **Flush**: a Lucene commit, writing the segments to disk durably, after which the translog can be cleared. Automatic.
- **Merge**: background merging of small segments into bigger ones, which also removes the deleted documents for good.

Near real time in practice:

- `?refresh=wait_for` on a write waits for the next refresh before responding: the document is searchable when the call returns. Useful in tests, or when the user must see their change immediately.
- `?refresh=true` forces an immediate refresh: costly, avoid in production (lots of tiny segments).
- Mass loading: set `refresh_interval` to `-1` and replicas to `0` during the load, then restore them.

**Updates and deletes**: segments are immutable, so a document is never modified in place.

- An update (even partial with `_update`) marks the old version as deleted and indexes the whole new document. Frequently updating the same document (a view counter) is expensive.
- A delete only marks the document; the disk space is freed at the merge.

**Concurrency**: each document has a `_seq_no` and a `_primary_term`, returned by `GET` and by searches. A conditional write fails with `409 Conflict` if the document changed in between, like a JPA `@Version`:

```http
PUT /products/_doc/1?if_seq_no=12&if_primary_term=1
```

When Elasticsearch is fed from a database, **external versioning** prevents an old event from overwriting a newer one (events arriving out of order): `PUT /products/_doc/1?version=42&version_type=external`, with the version of the database row. The write is rejected if the stored version is already greater or equal.

## Subtleties and pitfalls

> **In short**: `from` + `size` stops at 10,000 results (use `search_after`). Arrays of objects are flattened, mixing their fields (use `nested`). There are no joins: data is denormalized and must be reindexed when the copied values change. Dynamic keys make the mapping explode. Feeding Elasticsearch from the database needs a reliable, idempotent pipeline, not a dual write. Counts, aggregations and scores are partly approximate or per shard.

**Deep pagination**: with `from` + `size`, each shard must return its first `from + size` results to the coordinating node, which sorts them all. Hence a limit of 10,000 (`index.max_result_window`).

- Beyond that, use `search_after`: pass the sort values of the last hit of the previous page, with a unique tiebreaker field in the sort. Combined with a point in time (PIT), the pages stay consistent even if the index changes. The `scroll` API is the older way, no longer recommended for this.
- `hits.total` is capped at 10,000 by default (`"relation": "gte"`); `"track_total_hits": true` gives the exact count, at a cost.

**`object` vs `nested`**: an array of objects is flattened into arrays of values, losing which values belong together.

```json
{ "name": "T-shirt", "variants": [ { "color": "red", "size": "M" }, { "color": "blue", "size": "L" } ] }
```

Indexed as `variants.color: [red, blue]`, `variants.size: [M, L]`: a search for `color = red AND size = L` matches, although no red L exists. The `nested` type indexes each object as a hidden separate document and keeps the pairs, queried with a `nested` query; each nested object counts as a document, and updating one reindexes them all.

**No joins**: the document must contain everything that's searched and displayed, so data is denormalized (the category name copied into each product).

- When a copied value changes (a category is renamed), all the documents containing it must be reindexed (`_update_by_query`, or from the database).
- Alternatives when denormalizing isn't possible: `nested`, the `join` field type (parent/child in the same shard, slower), or two queries joined by the application.

**Mapping explosion**: with dynamic mapping, each new key becomes a field. Product attributes stored as keys (`"attributes": { "ram_gb": 16, "screen_inches": 15.6, ... }`) can create thousands of fields, each one in the mapping and the cluster state. Limit: 1000 fields per index by default (`index.mapping.total_fields.limit`). Fix: `nested` key/value pairs (`{ "key": "ram_gb", "value": "16" }`) or the `flattened` type.

**Synchronization with the database**:

- Dual write (save in PostgreSQL, then index in Elasticsearch in the same request): if the second write fails, the two stores diverge silently. Use an outbox or CDC (Debezium) feeding Kafka, and an indexer consuming the events (see [Kafka outbox pattern](../kafka/KAFKA_OUTBOX.md)).
- The indexer must be idempotent (an event can be delivered twice), and resistant to reordering: external versioning (see writing above).
- Keep a way to rebuild the whole index from the database: after a mapping change, a bug, or lost events.

**Approximations and per-shard statistics**:

- `terms` aggregations on several shards are approximate for counts beyond the top buckets (each shard returns its own top), and `cardinality` is an estimate.
- The score's term statistics (inverse document frequency) are computed per shard: on a small index with several shards, the ranking can look inconsistent. Not visible on large volumes; `search_type=dfs_query_then_fetch` computes global statistics, at a cost.

**Operations**: JVM heap at most half the RAM (and below ~31 GB), the rest left to the filesystem cache, which Lucene relies on. At least 3 master-eligible nodes, so that a master can still be elected by majority if one fails.

## Example: product search with Spring Data Elasticsearch

> **In short**: `spring-boot-starter-data-elasticsearch` maps a class to an index with `@Document` and `@Field`, and an `ElasticsearchRepository` gives CRUD and derived queries. Complex searches are built with `NativeQuery` (the official Java client's DSL) and run by `ElasticsearchOperations`. The index is fed from the database's events, with external versioning so that an old event can't overwrite a newer one.

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: ${ELASTIC_PASSWORD}
```

**The document**: the index and its mapping are created beforehand (`products_v1` behind the alias `products`, see the mapping above), so Spring doesn't create it.

```java
@Document(indexName = "products", createIndex = false, versionType = Document.VersionType.EXTERNAL)
public record ProductDocument(
    @Id String id,
    @MultiField(mainField = @Field(type = FieldType.Text, analyzer = "english"),
                otherFields = @InnerField(suffix = "keyword", type = FieldType.Keyword))
    String name,
    @Field(type = FieldType.Keyword) String brand,
    @Field(type = FieldType.Scaled_Float, scalingFactor = 100) double price,
    @Field(type = FieldType.Boolean) boolean inStock,
    @Version Long version) {}  // the database row's version, sent as external version
```

**The repository**: CRUD, derived queries, and JSON queries with `@Query`.

```java
public interface ProductSearchRepository extends ElasticsearchRepository<ProductDocument, String> {

  List<ProductDocument> findByBrandAndInStockTrue(String brand); // term queries on brand and inStock

  @Query("""
      { "match": { "name": { "query": "?0", "fuzziness": "AUTO" } } }
      """)
  Page<ProductDocument> searchByName(String text, Pageable pageable);
}
```

**A complex search**: the `bool` query of the search section, built with `NativeQuery`.

```java
@Service
class ProductSearchService {
  private final ElasticsearchOperations operations;

  List<ProductDocument> search(String text, double maxPrice) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(q -> q.bool(b -> b
            .must(m -> m.match(t -> t.field("name").query(text).fuzziness("AUTO")))
            .filter(f -> f.term(t -> t.field("inStock").value(true)))
            .filter(f -> f.range(r -> r.number(n -> n.field("price").lte(maxPrice))))))
        .withAggregation("brands", Aggregation.of(a -> a.terms(t -> t.field("brand"))))
        .withPageable(PageRequest.of(0, 20))
        .build();

    SearchHits<ProductDocument> hits = operations.search(query, ProductDocument.class);
    return hits.getSearchHits().stream().map(SearchHit::getContent).toList(); // also getScore(), getAggregations()
  }
}
```

**Feeding the index from the database**: the product service publishes its changes (through an outbox), an indexer consumes them.

```java
@Component
class ProductIndexer {
  private final ProductSearchRepository repository;

  @KafkaListener(topics = "product-events")
  void on(ProductChanged event) {
    try {
      repository.save(new ProductDocument(event.id(), event.name(), event.brand(),
          event.price(), event.inStock(), event.version()));
    } catch (VersionConflictException e) { // org.springframework.data.elasticsearch
      // a newer version is already indexed: this event is outdated, ignore it
    }
  }
}
```

- Replaying the same event is harmless (same id, same version rejected): the indexer is idempotent.
- Integration tests: Testcontainers' `ElasticsearchContainer`, and a refresh before asserting a search (`operations.indexOps(ProductDocument.class).refresh()`), otherwise the document just indexed isn't found yet.
