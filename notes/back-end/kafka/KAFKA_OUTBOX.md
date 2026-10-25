## The dual write problem

> **In short**: Writing to the database and publishing to Kafka are two writes on two systems, with no shared transaction: a rollback after the send publishes a phantom event, and a failure after the commit loses the event.

A service that updates its database and publishes an event to Kafka performs two writes on two systems, which can't be in the same transaction:

```java
@Transactional
public void createOrder(CreateOrderRequest request) {
  Order order = orderRepository.save(new Order(request));
  kafkaTemplate.send("orders", order.getId().toString(), new OrderCreated(order.getId())); // not part of the DB transaction
}
```

| Failure                                                         | Result                                                                |
| --------------------------------------------------------------- | --------------------------------------------------------------------- |
| Kafka send OK, then the DB transaction rolls back               | event published for an order that doesn't exist                       |
| DB commit OK, then the Kafka send fails (broker down, crash)    | order created, event lost: other services never know                  |
| Send after commit (`@TransactionalEventListener(AFTER_COMMIT)`) | still lost if the application crashes between the commit and the send |

Kafka transactions don't help: they make Kafka writes atomic with each other, not with a database.

## The outbox pattern

> **In short**: The event is written in an `outbox` table in the same database transaction as the business data; a relay then publishes it to Kafka. Atomic by design, delivered at least once (consumers must be idempotent), and ordered per aggregate. The relay is either a polling job or CDC with Debezium.

The event is written **in the database, in the same transaction** as the business data, in an `outbox` table. A separate process reads this table and publishes the events to Kafka.

```text
Service ──(1 DB transaction)──> orders + outbox
                                      │
                         (2) relay reads the outbox
                                      ▼
                                    Kafka ──> consumers
```

- Atomicity: the order and its event are committed together, or not at all.
- Delivery: **at least once**. The relay can publish an event, crash before recording it as published, and publish it again. Consumers must be idempotent (event id).
- Ordering: events of the same aggregate are published in commit order, with the aggregate id as Kafka key (same partition).

Two ways to relay:

|                | Polling publisher                                                                  | Change Data Capture (Debezium)                            |
| -------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------- |
| How            | a scheduled job: `SELECT ... WHERE published = false`, send, `UPDATE`              | reads the database's transaction log (PostgreSQL WAL)     |
| Latency        | polling interval                                                                   | near real time                                            |
| Load on the DB | repeated queries                                                                   | low: reads the log                                        |
| Infrastructure | none (inside the application)                                                      | Kafka Connect + Debezium connector                        |
| Pitfalls       | several instances: lock rows (`FOR UPDATE SKIP LOCKED`); cleanup of published rows | replication slot and WAL retention to monitor (see below) |

## Step 1: the outbox table and the application side

> **In short**: An `outbox` table (`id`, `aggregatetype`, `aggregateid`, `type`, `payload`), filled in the same `@Transactional` method as the business write. Since Debezium reads the WAL, the row can be deleted right after the insert.

The table expected by default by Debezium's Outbox Event Router:

```sql
CREATE TABLE outbox (
  id            uuid         PRIMARY KEY,  -- event id: Kafka header "id", for idempotent consumers
  aggregatetype varchar(255) NOT NULL,     -- routes to the topic: "order" → order.events
  aggregateid   varchar(255) NOT NULL,     -- Kafka key: ordering per aggregate
  type          varchar(255) NOT NULL,     -- event type: "OrderCreated"
  payload       jsonb,                     -- the event itself
  created_at    timestamptz  NOT NULL DEFAULT now()
);
```

The business write and the event in the same Spring transaction:

```java
@Entity
@Table(name = "outbox")
public class OutboxEvent {
  @Id private UUID id;
  private String aggregatetype;
  private String aggregateid;
  private String type;
  @JdbcTypeCode(SqlTypes.JSON) private String payload; // jsonb
  // constructor, getters
}

@Service
public class OrderService {

  @Transactional
  public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(new Order(request));
    outboxRepository.save(new OutboxEvent(
        UUID.randomUUID(), "order", order.getId().toString(), "OrderCreated",
        jsonMapper.writeValueAsString(new OrderCreated(order.getId(), order.getCustomer(), order.getTotal()))));
    return order;
  }
}
```

Keeping the table empty: Debezium reads the **WAL**, not the table. An `INSERT` followed by a `DELETE` in the same transaction still produces the insert event in the WAL, and the Outbox Event Router ignores `DELETE` operations. So the row can be deleted right away (`outboxRepository.delete(event)` after `save`), or a scheduled job purges old rows.

## Step 2: enable logical decoding (WAL) in PostgreSQL

> **In short**: `wal_level=logical` (restart required) makes the WAL contain row-level changes that Debezium can decode, with enough `max_replication_slots` and `max_wal_senders`; the `pgoutput` plugin is built into PostgreSQL.

The WAL (write-ahead log) records every change before it's applied to the tables, for crash recovery and replication. With `wal_level=logical`, it contains enough information to rebuild row-level changes (logical decoding), which Debezium reads.

```ini
# postgresql.conf (restart required)
wal_level = logical
max_replication_slots = 4   # at least 1 per Debezium connector
max_wal_senders = 4         # at least 1 per connector
```

```sql
SHOW wal_level; -- logical
```

- Docker: `postgres -c wal_level=logical`. Managed databases: a parameter (`rds.logical_replication=1` on AWS RDS, `cloudsql.logical_decoding` on Cloud SQL).
- The output plugin is **`pgoutput`**, built into PostgreSQL 10+: nothing to install.

## Step 3: the replication user

> **In short**: A dedicated role with `LOGIN REPLICATION`, and `SELECT` on the outbox table for the initial snapshot.

```sql
CREATE ROLE debezium WITH LOGIN REPLICATION PASSWORD 'secret';
GRANT CONNECT ON DATABASE shop TO debezium;
GRANT USAGE ON SCHEMA public TO debezium;
GRANT SELECT ON public.outbox TO debezium; -- for the initial snapshot
```

- `REPLICATION`: needed to open a replication connection and create a replication slot.
- `pg_hba.conf` must allow the user's connections (managed databases handle it).

## Step 4: the publication

> **In short**: A publication defines which tables are streamed: create it manually for the outbox table only (`CREATE PUBLICATION ... FOR TABLE public.outbox`), and set `publication.autocreate.mode=disabled` in the connector.

With `pgoutput`, a **publication** defines which tables are sent through logical replication. Create it manually, for the outbox table only:

```sql
CREATE PUBLICATION outbox_publication FOR TABLE public.outbox;
-- check
SELECT * FROM pg_publication_tables WHERE pubname = 'outbox_publication';
```

- Creating a publication requires owning the table (or superuser): done by the application's owner role (e.g. in a Flyway migration), not by the Debezium user.
- In the connector: `publication.name=outbox_publication` and `publication.autocreate.mode=disabled`. Otherwise Debezium tries to create it itself (default `all_tables`: a publication for **all** tables, which needs more privileges; `filtered`: only the tables of `table.include.list`).

## Step 5: the replication slot

> **In short**: The replication slot records how far the connector has read the WAL, so nothing is lost on restart; but while the connector is stopped, PostgreSQL keeps the WAL, which grows on disk: monitor it and drop unused slots.

A **replication slot** records how far the connector has read the WAL (its LSN). PostgreSQL keeps the WAL from that position: after a connector restart, nothing is lost.

- Created automatically by Debezium at first start (`slot.name`, one unique slot per connector), with the `pgoutput` plugin.
- Manual alternative: `SELECT pg_create_logical_replication_slot('outbox_slot', 'pgoutput');`
- **Pitfall**: while the connector is stopped, the slot retains the WAL, which **grows on disk** until the database is full. Monitor it, and drop the slot when the connector is removed:

```sql
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;

SELECT pg_drop_replication_slot('outbox_slot');
```

## Step 6: Kafka Connect and the Debezium connector

> **In short**: Debezium runs as a source connector in Kafka Connect, a separate process storing its configuration and offsets in Kafka topics; the connector is registered through the Kafka Connect REST API with the database, publication, slot and transformation settings.

**Kafka Connect** is the Kafka framework to run connectors (sources and sinks) outside the application: a separate process (cluster of workers) storing its configuration and offsets in Kafka topics. Debezium is a source connector running in it.

```yaml
# docker-compose.yml (excerpt)
connect:
  image: quay.io/debezium/connect:3.7
  ports: ["8083:8083"]
  environment:
    BOOTSTRAP_SERVERS: kafka:9092
    GROUP_ID: connect-cluster
    CONFIG_STORAGE_TOPIC: connect_configs # connector configurations
    OFFSET_STORAGE_TOPIC: connect_offsets # position of each connector (here: the LSN)
    STATUS_STORAGE_TOPIC: connect_statuses
```

The connector is registered through the Kafka Connect REST API (`POST http://localhost:8083/connectors`):

```json
{
  "name": "shop-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "shop",
    "topic.prefix": "shop",
    "plugin.name": "pgoutput",
    "publication.name": "outbox_publication",
    "publication.autocreate.mode": "disabled",
    "slot.name": "outbox_slot",
    "table.include.list": "public.outbox",
    "tombstones.on.delete": "false",
    "heartbeat.interval.ms": "10000",

    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.route.by.field": "aggregatetype",
    "transforms.outbox.route.topic.replacement": "${routedByValue}.events",
    "transforms.outbox.table.field.event.key": "aggregateid",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.table.expand.json.payload": "true",
    "transforms.outbox.table.fields.additional.placement": "type:header:eventType",

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
  }
}
```

- `topic.prefix`: logical name of the database, prefix of the raw change topics (`shop.public.outbox`), replaced here by the SMT routing.
- `heartbeat.interval.ms`: lets the connector confirm its position regularly even without outbox writes, so PostgreSQL can release the WAL (low-traffic databases).
- `tombstones.on.delete=false`: no tombstone record for the deletes of the outbox rows.
- `GET /connectors/shop-outbox-connector/status` checks that the connector and its task are `RUNNING`.

## Step 7: the Outbox Event Router transformation

> **In short**: The Outbox Event Router SMT turns a raw change event of the outbox table into a clean business event: topic from `aggregatetype`, key from `aggregateid` (ordering), value from `payload`, event id and type as headers.

Without transformation, Debezium publishes a **change event** for each row change: an envelope with `before`, `after`, `source` (LSN, transaction, table), `op` (`c`, `u`, `d`), to a topic per table. Not what consumers want.

A **SMT** (Single Message Transform) modifies each record inside Kafka Connect before it's written to Kafka. The **Outbox Event Router** turns an outbox insert into a clean business event:

| Outbox column             | Kafka record                                                                | Option (default)                                                                                |
| ------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `aggregatetype` = `order` | topic `order.events`                                                        | `route.by.field` (`aggregatetype`), `route.topic.replacement` (`outbox.event.${routedByValue}`) |
| `aggregateid`             | key: same order → same partition → ordered                                  | `table.field.event.key` (`aggregateid`)                                                         |
| `payload`                 | value, as JSON (`table.expand.json.payload=true`, with the `JsonConverter`) | `table.field.event.payload` (`payload`)                                                         |
| `id`                      | header `id`                                                                 | `table.field.event.id` (`id`)                                                                   |
| `type`                    | header `eventType`                                                          | `table.fields.additional.placement` (none by default)                                           |

```text
Topic: order.events   Key: "42"   Headers: id=3f1c..., eventType=OrderCreated
Value: {"orderId": 42, "customer": "alice", "total": 100.0}
```

- `DELETE` operations are ignored; updates of outbox rows are unexpected (warning by default).
- Without `expand.json.payload`, the payload is a JSON string inside the value (`"{\"orderId\":42}"`).

## Step 8: the consumer side

> **In short**: A normal Kafka listener, made idempotent with the event id header: already-processed ids are recorded in a table, in the same transaction as the business write.

A normal Spring Kafka listener (see [Spring Kafka](../spring/SPRING_KAFKA.md)), idempotent because the relay is at-least-once:

```java
@KafkaListener(topics = "order.events", groupId = "billing")
public void onOrderEvent(OrderCreated event,
                         @Header("id") String eventId,
                         @Header("eventType") String eventType) {
  if (processedEventRepository.existsById(eventId)) {
    return; // duplicate: already processed
  }
  billingService.createInvoice(event);
  processedEventRepository.save(new ProcessedEvent(eventId)); // same transaction as the invoice
}
```

- The `processed_event` table (inbox) and the business write in the same transaction: a duplicate is detected even after a crash.
- `eventType` header: dispatch several event types published to the same topic (`OrderCreated`, `OrderCancelled`).

## Pitfalls

> **In short**: WAL growth when the connector is down, slots lost on failover, the event schema as a public contract, the initial snapshot republishing old rows, and publishing only events other services need, never the internal tables directly.

- **WAL growth**: a stopped or failing connector keeps its slot, PostgreSQL keeps all the WAL since: monitor `pg_replication_slots`, alert on retained size, drop unused slots. PostgreSQL 13+: `max_slot_wal_keep_size` caps the retained WAL (the slot is invalidated beyond, the connector then needs a new snapshot).
- **Failover**: a logical slot exists only on the primary; after a failover to a replica, it's lost unless failover slots are used (PostgreSQL 17+).
- **Schema evolution**: the payload is a public contract: add optional fields, don't rename or remove; a schema registry (Avro, JSON Schema) enforces it.
- **Snapshot**: at first start, Debezium snapshots the existing rows of `public.outbox` (`snapshot.mode=initial`): old events are published. `no_data` skips it when starting on a table with history.
- **Only events other services need**: not every database change deserves an event.
- **Why not Debezium on the business tables directly**: consumers would receive the rows of the internal tables (column names, internal codes, `before` / `after`), which become a public API: renaming a column or splitting a table breaks them. The outbox payload is an event with its **own schema**, designed as a contract and independent of the database: it can combine data from several tables (an `OrderCreated` with the order, its lines and the customer), or come from no table at all (an event built by the application, like `PasswordResetRequested`, without any JPA entity behind it).
