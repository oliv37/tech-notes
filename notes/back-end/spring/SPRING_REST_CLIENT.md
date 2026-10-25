## RestClient: calling a REST API

> **In short**: `RestClient` (Spring 6.1+) is Spring's synchronous HTTP client with a fluent API, replacing `RestTemplate`. Build it from the `RestClient.Builder` auto-configured by Spring Boot; 4xx / 5xx responses throw exceptions, customizable with `onStatus`.

`RestClient` (Spring Framework 6.1+) is the synchronous HTTP client of Spring, with a fluent API. It replaces `RestTemplate` (in maintenance, planned for deprecation) for blocking code; `WebClient` stays the reactive client.

Spring Boot auto-configures a `RestClient.Builder` (message converters, Jackson, timeouts from the properties, observability): inject the builder rather than calling `RestClient.create()`, which skips all of it.

```java
@Component
public class PaymentClient {

  private final RestClient restClient;

  public PaymentClient(RestClient.Builder builder) {
    this.restClient = builder.baseUrl("https://payment.example.com").build();
  }

  public Payment findById(long id) {
    return restClient.get()
        .uri("/payments/{id}", id)
        .retrieve()
        .body(Payment.class);
  }

  public Payment create(PaymentRequest request) {
    return restClient.post()
        .uri("/payments")
        .body(request)                      // serialized to JSON
        .retrieve()
        .body(Payment.class);
  }
}
```

- `retrieve().body(...)` for the body only, `toEntity(...)` for status and headers too, `toBodilessEntity()` when there's no body. A generic type uses `body(new ParameterizedTypeReference<List<Payment>>() {})`.
- Errors: a 4xx / 5xx response throws `HttpClientErrorException` / `HttpServerErrorException` (both `RestClientResponseException`). `onStatus` replaces it with a custom exception:

```java
.retrieve()
.onStatus(status -> status.value() == 404, (request, response) -> {
  throw new PaymentNotFoundException(id);
})
.body(Payment.class);
```

- The builder bean is a prototype: each injection point gets its own. It's mutable, so within one class, a change affects every client built afterwards from it: `builder.clone()` to build several different clients.

## HTTP service interfaces (@HttpExchange)

> **In short**: A declarative client: an interface annotated with `@GetExchange`, `@PostExchange`..., implemented at runtime by a proxy over `RestClient`. Since Spring Framework 7 / Boot 4, `@ImportHttpServices` registers these interfaces as beans.

A declarative client: an interface with annotated methods, implemented at runtime by a proxy that delegates to a `RestClient` (same idea as Feign).

```java
public interface PaymentService {

  @GetExchange("/payments/{id}")
  Payment findById(@PathVariable long id);

  @PostExchange("/payments")
  Payment create(@RequestBody PaymentRequest request);

  @GetExchange("/payments")
  List<Payment> search(@RequestParam String status);
}
```

Since Spring Framework 7 / Spring Boot 4, `@ImportHttpServices` registers the proxies as beans, without building an `HttpServiceProxyFactory` by hand:

```java
@SpringBootApplication
@ImportHttpServices(basePackages = "com.example.shop.client") // every @HttpExchange interface of the package
public class ShopApplication { }
```

```java
@Service
public class OrderService {
  private final PaymentService paymentService; // injected like any bean
  // ...
}
```

The base URL can be written on the interface (`@HttpExchange(url = "https://payment.example.com")`), but it's better taken from the configuration (see below).

## Several clients with a different URL each

> **In short**: With HTTP service interfaces, each remote API is a group (`@ImportHttpServices(group = "payment", ...)`), configured with `spring.http.serviceclient.<group>.base-url`. With plain `RestClient`, declare one bean per API, each with its URL from `@ConfigurationProperties`.

With HTTP service interfaces, each remote API is a **group**: the interfaces of a group share the same `RestClient`, configured under `spring.http.serviceclient.<group>` (Spring Boot 4).

```java
@SpringBootApplication
@ImportHttpServices(group = "payment", types = PaymentService.class)
@ImportHttpServices(group = "stock", types = {StockService.class, WarehouseService.class})
public class ShopApplication { }
```

```yaml
spring:
  http:
    clients: # applies to every client
      connect-timeout: 2s
      read-timeout: 5s
    serviceclient:
      payment:
        base-url: https://payment.example.com
        read-timeout: 10s # overrides the global value for this group
      stock:
        base-url: https://stock.example.com
```

- One URL per environment comes for free: `application-prod.yml`, or an environment variable (`SPRING_HTTP_SERVICECLIENT_PAYMENT_BASEURL`).
- An interface without a group goes to the `default` group.
- What isn't a property (an auth header, an interceptor) is set per group in code:

```java
@Bean
RestClientHttpServiceGroupConfigurer groupConfigurer() {
  return groups -> groups
      .filterByName("payment")
      .forEachClient((group, builder) -> builder.defaultHeader("X-Api-Key", "..."));
}
```

With plain `RestClient` (no interface), declare one bean per API, each with its own URL from a `@ConfigurationProperties` class:

```java
@ConfigurationProperties("shop.clients")
public record ClientsProperties(String paymentUrl, String stockUrl) { }
```

```java
@Configuration
@EnableConfigurationProperties(ClientsProperties.class)
public class RestClientConfig {

  @Bean
  RestClient paymentRestClient(RestClient.Builder builder, ClientsProperties props) {
    return builder.baseUrl(props.paymentUrl()).build();
  }

  @Bean
  RestClient stockRestClient(RestClient.Builder builder, ClientsProperties props) {
    return builder.baseUrl(props.stockUrl()).build();
  }
}
```

```yaml
shop:
  clients:
    payment-url: https://payment.example.com
    stock-url: https://stock.example.com
```

Each `@Bean` method receives its own builder (prototype), so the two URLs don't interfere. Two beans of the same type: inject by name, `@Qualifier("paymentRestClient") RestClient restClient`.
