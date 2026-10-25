## Spring Kafka setup

> **In short**: Spring Kafka provides `KafkaTemplate` to produce and `@KafkaListener` to consume, configured by Spring Boot through `spring.kafka.*` properties: bootstrap servers, serializers (`JacksonJsonSerializer` with Jackson 3), consumer group. `ErrorHandlingDeserializer` keeps an unreadable message from blocking the consumer.

Spring for Apache Kafka provides `KafkaTemplate` to produce and `@KafkaListener` to consume (see [Kafka](KAFKA.md) for the concepts). With Spring Boot, the configuration is done with `spring.kafka.*` properties:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JacksonJsonSerializer
      acks: all
    consumer:
      group-id: billing
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JacksonJsonDeserializer
        spring.json.trusted.packages: "com.example.shop.events"
```

- JSON with Jackson 3: `JacksonJsonSerializer` / `JacksonJsonDeserializer` (Spring Kafka 4.0, Spring Boot 4). `JsonSerializer` / `JsonDeserializer` are the deprecated Jackson 2 versions.
- `ErrorHandlingDeserializer` wraps the real deserializer: an unreadable message is passed to the error handler instead of failing on every `poll()` (see below).
- `spring.json.trusted.packages`: the deserializer only instantiates classes from these packages (the type comes from a header written by the producer).

Topics can be created at startup by declaring `NewTopic` beans (`KafkaAdmin`, auto-configured by Spring Boot):

```java
@Bean
NewTopic ordersTopic() {
  return TopicBuilder.name("orders").partitions(6).replicas(3).build();
}
```

## Producing messages: KafkaTemplate

> **In short**: `kafkaTemplate.send(topic, key, value)` is asynchronous and returns a `CompletableFuture`, which must be checked, otherwise failures go unnoticed. The key (the entity id) decides the partition and so the ordering; the database and Kafka aren't atomic, hence the outbox pattern.

```java
public record OrderCreated(String orderId, String customer, BigDecimal amount) {}

@Service
public class OrderEventPublisher {

  private final KafkaTemplate<String, OrderCreated> kafkaTemplate;

  public OrderEventPublisher(KafkaTemplate<String, OrderCreated> kafkaTemplate) {
    this.kafkaTemplate = kafkaTemplate;
  }

  public void publish(OrderCreated event) {
    kafkaTemplate.send("orders", event.orderId(), event) // key = orderId: same order → same partition
        .whenComplete((result, ex) -> {
          if (ex != null) {
            log.error("Failed to publish {}", event.orderId(), ex);
          } else {
            log.info("Published to {}-{} at offset {}", result.getRecordMetadata().topic(),
                result.getRecordMetadata().partition(), result.getRecordMetadata().offset());
          }
        });
  }
}
```

- `send()` is **asynchronous**: it adds the message to a batch and returns a `CompletableFuture<SendResult>`. A failure (broker unavailable, timeout) is only visible in the future: not checking it loses messages silently. `send(...).get()` blocks until the acknowledgment, simpler but slower.
- The key decides the partition, hence the ordering: use the business id of the entity (`orderId`).
- Headers: send a `ProducerRecord` or a `Message<?>` built with `MessageBuilder.withPayload(event).setHeader(KafkaHeaders.TOPIC, "orders")`.

**Database + Kafka**: in a `@Transactional` method, the database commit and the Kafka send aren't atomic: the transaction can roll back after the message was sent, or the send can fail after the commit. Reliable solution: the **outbox pattern**: the event is written in an `outbox` table in the same transaction, and a separate process (scheduler, Debezium) publishes it to Kafka (see [Kafka outbox](KAFKA_OUTBOX.md)).

## Consuming messages: @KafkaListener

> **In short**: `@KafkaListener(topics, groupId)` methods run in a listener container that polls Kafka and calls them for each record, sequentially per partition; `concurrency` adds consumers to the group, useful up to the number of partitions.

```java
@Component
public class BillingListener {

  @KafkaListener(topics = "orders", groupId = "billing", concurrency = "3")
  public void onOrderCreated(OrderCreated event,
                             @Header(KafkaHeaders.RECEIVED_KEY) String key,
                             @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                             @Header(KafkaHeaders.OFFSET) long offset) {
    billingService.createInvoice(event);
  }
}
```

- The listener runs in a **listener container**, which polls Kafka in a loop and calls the method for each record. `concurrency = "3"`: 3 consumers (3 threads) in the group, useful only with at least 3 partitions.
- Parameters: the payload (converted from JSON), `@Header`s, the whole `ConsumerRecord<String, OrderCreated>`, an `Acknowledgment` (manual commit), `@Payload @Valid` for validation.
- Within a partition, records are processed **one after the other**, in order: a slow listener slows down its partition.
- `groupId` overrides `spring.kafka.consumer.group-id`. Placeholders work: `topics = "${shop.topics.orders}"`.
- **Batch listener**: `@KafkaListener(topics = "orders", batch = "true")` with a `List<OrderCreated>` parameter, to process several records at once (bulk insert).

## Offset commit: AckMode

> **In short**: Spring commits the offsets itself after the listener returns (`AckMode.BATCH` by default), or on `ack.acknowledge()` in manual mode: at-least-once delivery, so listeners must be idempotent.

Spring disables Kafka's auto-commit (`enable.auto.commit=false`) and commits the offsets itself, **after** the listener returns: at-least-once delivery.

| `AckMode`          | Commit                                                                |
| ------------------ | --------------------------------------------------------------------- |
| `BATCH` (default)  | after all the records of a `poll()` have been processed               |
| `RECORD`           | after each record                                                     |
| `TIME` / `COUNT`   | after a delay / a number of records                                   |
| `MANUAL`           | when the listener calls `ack.acknowledge()`, committed with the batch |
| `MANUAL_IMMEDIATE` | when the listener calls `ack.acknowledge()`, committed immediately    |

```yaml
spring.kafka.listener.ack-mode: manual
```

```java
@KafkaListener(topics = "orders")
public void listen(OrderCreated event, Acknowledgment ack) {
  billingService.createInvoice(event);
  ack.acknowledge();
}
```

Whatever the mode, a crash before the commit replays the messages: the listener must be **idempotent** (check whether the `orderId` was already processed, upsert).

## Error handling: retries and dead letter topic

> **In short**: The `DefaultErrorHandler` retries a failed record (blocking its partition), then hands it to a recoverer, typically `DeadLetterPublishingRecoverer` towards a `.DLT` topic. `@RetryableTopic` retries through separate retry topics instead, without blocking the partition but losing the order.

When the listener throws an exception, the **`DefaultErrorHandler`** retries the record: by default 9 retries without delay (10 attempts), then the record is logged and skipped, and the partition moves on. The retries are **blocking**: the partition waits meanwhile.

Usual configuration: retries with a backoff, then the record is sent to a **dead letter topic** (`orders.DLT`) to be analysed or replayed:

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
  var recoverer = new DeadLetterPublishingRecoverer(template); // → topic "orders.DLT", same partition
  var handler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3)); // 3 retries, 1 s apart
  handler.addNotRetryableExceptions(ValidationException.class); // straight to the DLT
  return handler;
}
```

- Spring Boot plugs a `CommonErrorHandler` bean into the auto-configured listener container factory.
- Not retryable: errors that will fail every time (invalid message, business rule). By default, deserialization errors (`DeserializationException`, with `ErrorHandlingDeserializer`) and a few others go directly to the recoverer.
- The DLT record keeps the original payload, with headers describing the failure (exception, original topic, partition, offset).

**Non-blocking retries: `@RetryableTopic`**: a failed record is moved to retry topics (`orders-retry-0`, `orders-retry-1`...) consumed with a delay, so the main partition isn't blocked. The price: the **order is lost** for retried records.

```java
@RetryableTopic(attempts = "4", backOff = @BackOff(delayString = "1s", multiplier = 2)) // Spring Kafka 4.0 syntax
@KafkaListener(topics = "orders", groupId = "billing")
public void listen(OrderCreated event) {
  billingService.createInvoice(event); // retried after 1 s, 2 s, 4 s, then sent to orders-dlt
}

@DltHandler
public void onDlt(OrderCreated event, @Header(KafkaHeaders.EXCEPTION_MESSAGE) String error) {
  log.error("Order {} sent to DLT: {}", event.orderId(), error);
}
```

|                          | `DefaultErrorHandler` (blocking)               | `@RetryableTopic` (non-blocking)       |
| ------------------------ | ---------------------------------------------- | -------------------------------------- |
| Partition during retries | blocked                                        | keeps going                            |
| Order of the records     | kept                                           | lost for retried records               |
| Suited for               | short retries (a few seconds), strict ordering | long retries (an API down for minutes) |

## Testing with Kafka

> **In short**: An integration test with a real broker (Testcontainers + `@ServiceConnection`), sending a message and waiting for the listener's effect with Awaitility, never `Thread.sleep()`; `@EmbeddedKafka` is a lighter, in-memory alternative.

An integration test with a real broker, using Testcontainers and `@ServiceConnection` (see [Spring Test](SPRING_TEST.md)): `spring.kafka.bootstrap-servers` is set automatically.

```java
@TestConfiguration(proxyBeanMethods = false)
class KafkaTestConfig {
  @Bean
  @ServiceConnection
  KafkaContainer kafka() {
    return new KafkaContainer("apache/kafka-native:4.0.0");
  }
}

@SpringBootTest
@Import(KafkaTestConfig.class)
class BillingListenerIT {

  @Autowired KafkaTemplate<String, OrderCreated> kafkaTemplate;
  @Autowired InvoiceRepository invoiceRepository;

  @Test
  void createsAnInvoiceForAnOrder() {
    kafkaTemplate.send("orders", "42", new OrderCreated("42", "alice", new BigDecimal("100")));

    await().atMost(Duration.ofSeconds(10)) // Awaitility: the listener runs asynchronously
        .untilAsserted(() -> assertThat(invoiceRepository.findByOrderId("42")).isPresent());
  }
}
```

- The consumption is asynchronous: wait with **Awaitility** (`await().untilAsserted(...)`), never `Thread.sleep()`.
- Lighter alternative: `@EmbeddedKafka` (spring-kafka-test), an in-memory KRaft broker, without Docker.
- `auto-offset-reset: earliest` in tests: otherwise a listener starting after the message was sent may not see it.
