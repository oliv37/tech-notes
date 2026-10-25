## What Kafka is

> **In short**: Kafka is a distributed event log: producers append messages to topics, consumers read them at their own pace. Messages stay on disk for the retention period, so several applications can read the same data and replay it; built for high throughput and horizontal scaling.

Apache Kafka is a distributed **event log**: producers append messages (records) to topics, consumers read them. Unlike a classic message queue, a message isn't deleted once read: it stays on disk for the retention period, and each consumer keeps its own position. Several applications can read the same messages, at their own pace, and replay them.

Typical uses: communication between microservices through events (`OrderCreated`, `PaymentAccepted`), data pipelines (CDC with Debezium, data lake), activity tracking, logs and metrics, stream processing (Kafka Streams).

What it provides: high throughput (sequential writes, batching, zero-copy), horizontal scalability (partitions), durability (replication), decoupling of producers and consumers in time.

## Main components

> **In short**: A cluster of brokers stores topics, split into partitions: ordered, append-only logs where each message has an offset. The message key decides the partition, so messages with the same key stay in order.

```text
Cluster (brokers 1, 2, 3)
└─ Topic "orders"
   ├─ Partition 0: [0][1][2][3][4][5] →   (offsets, append-only)
   ├─ Partition 1: [0][1][2][3] →
   └─ Partition 2: [0][1][2][3][4] →
```

| Component | Role |
|---|---|
| **Broker** | a Kafka server: stores partitions on disk, serves producers and consumers |
| **Cluster** | a set of brokers; metadata managed by KRaft controllers (ZooKeeper removed in Kafka 4.0) |
| **Topic** | a named stream of messages (`orders`), like a table or a category |
| **Partition** | a topic is split into partitions: ordered, append-only logs, spread over the brokers. The unit of parallelism and ordering |
| **Offset** | position of a message in its partition (0, 1, 2...), never reused |
| **Record** | a message: key (optional), value, headers, timestamp |
| **Producer** | writes records to a topic |
| **Consumer** | reads records from partitions, tracking its offsets |
| **Consumer group** | consumers sharing the work of reading a topic (see below) |

The key decides the partition: `hash(key) % number of partitions`. All messages with the same key (e.g. the same `orderId`) go to the same partition, so they are **read in the order they were written**. Without a key, messages are spread over the partitions (sticky partitioner: one partition per batch).

## Consumer groups

> **In short**: Within a consumer group, each partition is read by a single consumer, so the work is shared and parallelism is limited by the number of partitions; each group receives all the messages. Each group commits its offsets, and partitions are reassigned (rebalance) when consumers join or leave.

Consumers with the same `group.id` form a group: each partition of the topic is assigned to **one single consumer** of the group. The partitions are shared, each message is processed once per group.

```text
Topic "orders" (3 partitions)

Group "billing":       consumer A ← P0, P1     consumer B ← P2
Group "notification":  consumer C ← P0, P1, P2
```

- **Different groups each receive all the messages**: `billing` and `notification` both read every order (publish / subscribe). Inside a group, the messages are shared (work queue).
- **Parallelism limited by the partitions**: 3 partitions → at most 3 active consumers in a group; a 4th one stays idle. The number of partitions is chosen for the target parallelism.
- **Rebalance**: when a consumer joins, leaves or crashes (no heartbeat, or `max.poll.interval.ms` exceeded), the partitions are reassigned within the group. Kafka 4.0 makes the new consumer group protocol (KIP-848) generally available: the coordinator on the broker computes the assignment, and rebalances no longer stop the whole group.

**Offsets**: each group commits, per partition, the offset of the next message to read, stored by Kafka in the internal topic `__consumer_offsets`. After a restart or a rebalance, the consumer resumes from the committed offset.

- `auto.offset.reset`: where a new group starts when it has no committed offset: `latest` (default, only new messages) or `earliest` (from the beginning of the retention).
- Commit **after** processing: on a crash between processing and commit, the message is read again (see below).

## Producer: acks, idempotence, ordering

> **In short**: Each partition is replicated (leader + followers). With `acks=all` and `min.insync.replicas=2`, a write is acknowledged once replicated, and the idempotent producer (default) removes duplicates caused by retries while keeping the order.

```text
Producer → leader of partition 2 (broker 1) → replicated to followers (brokers 2, 3)
```

Each partition has a **leader** (receives writes and reads) and **followers** that copy it, with `replication.factor` copies in total (3 in production). The **ISR** (in-sync replicas) are the replicas up to date with the leader; if the leader's broker fails, an ISR replica becomes leader.

| `acks` | The producer waits for | Risk |
|---|---|---|
| `0` | nothing | messages lost silently |
| `1` | the leader's write | lost if the leader fails before replication |
| `all` (default since Kafka 3.0) | all the ISR | none, with `min.insync.replicas=2` |

- `acks=all` + `replication.factor=3` + `min.insync.replicas=2`: the usual durable setup, a write succeeds while at least 2 replicas are up.
- **Idempotent producer** (`enable.idempotence=true`, default since Kafka 3.0): the broker discards duplicates caused by retries (producer id + sequence number), and order is kept within a partition despite retries.
- **Batching**: messages are grouped per partition (`batch.size`, `linger.ms`) and compressed (`compression.type`), the main reason for Kafka's throughput.

## Delivery guarantees

> **In short**: Committing offsets after processing gives at-least-once delivery, so duplicates are possible. Exactly-once only covers Kafka-to-Kafka flows, so in practice: at-least-once with idempotent consumers.

| Guarantee | How | Consequence |
|---|---|---|
| **At most once** | commit the offset before processing | a crash loses the message |
| **At least once** | commit after processing (the usual choice) | a crash before the commit replays the message: **duplicates** |
| **Exactly once** | idempotent producer + transactions + consumers in `isolation.level=read_committed` | only for read-from-Kafka / write-to-Kafka flows (Kafka Streams) |

Exactly-once doesn't cover side effects outside Kafka (a database write, a call to an API). In practice: at-least-once, and **idempotent consumers** (an already-processed message id is ignored, an `upsert` instead of an `insert`).

## Retention and log compaction

> **In short**: Messages are kept for a time or size limit, whether consumed or not; log compaction instead keeps the last message of each key, turning the topic into a table of the latest state.

Messages are kept on disk, whether they were consumed or not:

- **Retention by time or size**: `retention.ms` (7 days by default), `retention.bytes`. Old segments are deleted.
- **Log compaction** (`cleanup.policy=compact`): Kafka keeps at least the **last message of each key**. The topic becomes a table of the latest state per key (e.g. the current address of each customer). A message with a `null` value (tombstone) deletes the key.

A consumer can replay a topic by resetting its offsets (`kafka-consumer-groups.sh --reset-offsets`), e.g. to rebuild a database after a bug.

## Pitfalls and design choices

> **In short**: Ordering only exists within a partition, adding partitions moves keys to other partitions, a frequent key overloads one partition, slow processing triggers rebalances, and a poison message needs a dead letter topic.

- **Ordering only within a partition**: no global order on a topic. Messages that must stay ordered (all the events of one order) need the same key.
- **Adding partitions** changes `hash(key) % partitions`: a key may move to another partition, the order between old and new messages of that key is lost. Choose the number of partitions upfront, with some margin.
- **Hot partition**: a very frequent key (a big customer) overloads one partition and its consumer.
- **Slow processing**: if `poll()` isn't called within `max.poll.interval.ms` (5 minutes by default), the consumer is excluded from the group, a rebalance follows and the messages are processed again by another consumer.
- **Poison message**: a message that always fails blocks its partition if retried forever: limited retries, then a dead letter topic (see [Spring Kafka](SPRING_KAFKA.md)).
- **Message format**: JSON, or Avro / Protobuf with a schema registry, to evolve the format without breaking the consumers (adding optional fields).

## Kafka vs RabbitMQ

> **In short**: Kafka is a replayable log read by several consumer groups, ordered per partition, built for streaming and very high throughput; RabbitMQ is a broker routing messages to queues and deleting them once acknowledged, better for task queues and complex routing.

| | Kafka | RabbitMQ |
|---|---|---|
| Model | distributed log, consumers pull | broker routing messages to queues, pushed to consumers |
| After consumption | message kept (retention), replayable | message deleted once acknowledged |
| Ordering | per partition | per queue |
| Scaling consumers | limited by the number of partitions | add consumers to the queue |
| Throughput | very high (millions of msg/s) | high, lower |
| Routing | by topic and key | rich: exchanges, routing keys, headers |
| Strengths | event streaming, replay, several readers of the same data, pipelines | task queues, complex routing, per-message acknowledgment, priorities |

Kafka 4.0 also introduces **share groups** (KIP-932, "Queues for Kafka", early access): several consumers can read the same partition, with per-message acknowledgment, like a queue.
