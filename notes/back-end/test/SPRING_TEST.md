## Which test for which layer

> **In short**: Unit tests without Spring for business logic, slice tests (`@WebMvcTest`, `@DataJpaTest`...) for each technical layer, and a few `@SpringBootTest` integration tests for the main flows.

| Tested component | Test | Context loaded | Speed |
|---|---|---|---|
| Service, business logic | unit test, JUnit + Mockito, no Spring | none | ms |
| Controller (mapping, JSON, validation, status, security) | `@WebMvcTest` + `MockMvcTester` | web layer only | fast |
| Repository (queries, mapping) | `@DataJpaTest` | JPA only | fast, slower with a real DB |
| REST client | `@RestClientTest` + `MockRestServiceServer` | client + JSON | fast |
| JSON serialization of a DTO | `@JsonTest` | Jackson only | fast |
| Whole application, flows across layers | `@SpringBootTest` (+ Testcontainers) | everything | slow |

The usual balance: many unit tests, slice tests for each technical layer, a few `@SpringBootTest` integration tests for the main flows. Dependency: `spring-boot-starter-test` (JUnit, Mockito, AssertJ, Spring Test); since Spring Boot 4, each technology has its own test starter for its slice (`spring-boot-starter-webmvc-test`, `spring-boot-starter-data-jpa-test`, `spring-boot-starter-security-test`).

## Unit testing a service

> **In short**: A service using constructor injection is tested without Spring, with Mockito mocks: fast, but it doesn't check the wiring, transactions, security or SQL.

A service with constructor injection is tested without Spring: its dependencies are mocks passed to the constructor (see [Mockito](MOCKITO.md)).

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

  @Mock OrderRepository orderRepository;
  @Mock PaymentClient paymentClient;
  @InjectMocks OrderService orderService;

  @Test
  void confirmsAPaidOrder() {
    when(orderRepository.findById(42L)).thenReturn(Optional.of(new Order(42L, 100.0)));
    when(paymentClient.pay(100.0)).thenReturn(PaymentResult.success("tx-1"));

    orderService.confirm(42L);

    verify(paymentClient).pay(100.0);
  }
}
```

- No context to start: milliseconds per test. The default for business logic.
- What it doesn't check: the Spring wiring, `@Transactional`, `@PreAuthorize`, the SQL. Those belong to slice or integration tests.

## @SpringBootTest: the whole application

> **In short**: `@SpringBootTest` starts the full context. With the default `MOCK` environment, no server starts and MockMvc calls the `DispatcherServlet` in memory, in the test thread (rollback, `@WithMockUser`); with `RANDOM_PORT`, a real server answers real HTTP calls (`RestTestClient`), closer to production.

`@SpringBootTest` starts the full application context, as in production (all beans, auto-configuration, `application.yml`).

```java
@SpringBootTest
@AutoConfigureMockMvc // needed since Spring Boot 4: no MockMvc by default
class OrderFlowIT {

  @Autowired MockMvcTester mvc;
  @Autowired OrderRepository orderRepository;

  @Test
  void createsAnOrder() {
    assertThat(mvc.post().uri("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"customer": "alice", "lines": [{"productId": 1, "quantity": 2}]}
                """))
        .hasStatus(HttpStatus.CREATED);

    assertThat(orderRepository.findAll()).hasSize(1);
  }
}
```

`webEnvironment` decides how the web layer runs:

| `webEnvironment` | Server | Client |
|---|---|---|
| `MOCK` (default) | no real server, mock servlet environment | `MockMvcTester` (`@AutoConfigureMockMvc`) |
| `RANDOM_PORT` / `DEFINED_PORT` | real embedded server | `RestTestClient` (`@AutoConfigureRestTestClient`, Spring Boot 4), `TestRestTemplate` |
| `NONE` | no web layer | direct bean calls |

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@AutoConfigureRestTestClient
class OrderApiIT {

  @Test
  void findsAnOrder(@Autowired RestTestClient client) {
    client.get().uri("/api/orders/42")
        .exchange()
        .expectStatus().isOk()
        .expectBody(OrderDto.class).value(order -> assertThat(order.id()).isEqualTo(42));
  }
}
```

`MOCK` starts no server and opens no port: MockMvc calls the `DispatcherServlet` directly, with in-memory `MockHttpServletRequest` / `MockHttpServletResponse`, in the test thread. The filters still run (security included), MockMvc chains them itself. Tomcat only starts with `RANDOM_PORT` / `DEFINED_PORT`.

```text
MOCK:         test → MockMvc → filters → DispatcherServlet → controller   (in memory, same thread)
RANDOM_PORT:  test → HTTP client → network → Tomcat → filters → DispatcherServlet → controller
```

Loading the context is the expensive part in both cases: starting Tomcat adds little. The real difference is the shared thread:

| | `MOCK` + `MockMvcTester` | `RANDOM_PORT` + `RestTestClient` |
|---|---|---|
| Database cleanup | rollback with `@Transactional` on the test | manual (`deleteAll()`, SQL scripts): the server uses its own threads and transactions |
| Security | `@WithMockUser`, `.with(jwt())` (same-thread `SecurityContext`) | real credentials or token |
| Internals | resolved exception, handler, model | HTTP response only |
| `/error` page | not simulated: after a `sendError(400)`, the status is right but the body is empty | real `BasicErrorController` body |
| Tomcat behaviour (multipart limits, compression, container filters) | not simulated | real |
| Close to production | almost | yes |

`MOCK` for most integration tests (rollback, mock user); `RANDOM_PORT` for a few end-to-end tests, or when the real HTTP behaviour is what's tested. Many teams run everything on `RANDOM_PORT` with Testcontainers, and handle the cleanup.

## Controller tests: @WebMvcTest

> **In short**: `@WebMvcTest` loads only the web layer (controllers, advices, filters, security) and replaces the services with `@MockitoBean`; `MockMvcTester` checks the mapping, JSON, validation, status codes and security rules.

`@WebMvcTest` loads only the web layer: controllers, `@RestControllerAdvice`, filters, `WebMvcConfigurer`, Jackson, Spring Security. Services and repositories aren't loaded: they're replaced with `@MockitoBean`.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

  @Autowired MockMvcTester mvc;
  @MockitoBean OrderService orderService;

  @Test
  void returnsAnOrder() {
    when(orderService.findById(42L)).thenReturn(new OrderDto(42L, "PAID"));

    assertThat(mvc.get().uri("/api/orders/42"))
        .hasStatusOk()
        .bodyJson().extractingPath("$.status").isEqualTo("PAID");
  }

  @Test
  void rejectsAnInvalidBody() {
    assertThat(mvc.post().uri("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"customer\": \"\"}"))
        .hasStatus(HttpStatus.BAD_REQUEST);
  }

  @Test
  void returns404WhenNotFound() {
    when(orderService.findById(1L)).thenThrow(new OrderNotFoundException(1L));

    assertThat(mvc.get().uri("/api/orders/1")).hasStatus(HttpStatus.NOT_FOUND);
  }
}
```

- What it tests: URL mapping, parameter binding, JSON (de)serialization, validation, exception handlers, status codes, security rules. Not the business logic (mocked).
- **`MockMvcTester`** (Spring Framework 6.2+): AssertJ-style API on top of `MockMvc`. The classic style still works: `mockMvc.perform(get("/api/orders/42")).andExpect(status().isOk()).andExpect(jsonPath("$.status").value("PAID"))`.
- `@WebMvcTest(OrderController.class)` loads only this controller; without a class, all controllers.
- `@MockitoBean` replaces a bean in the context with a mock (it replaces `@MockBean`, removed in Spring Boot 4).

**With Spring Security**, the filter chain is active in `@WebMvcTest`: an unauthenticated request gets `401`. Simulate a user (`spring-boot-starter-security-test`):

```java
@Test
@WithMockUser(roles = "ADMIN")
void adminCanDelete() {
  assertThat(mvc.delete().uri("/api/orders/42").with(csrf())).hasStatus(HttpStatus.NO_CONTENT);
}
```

- `@WithMockUser(username = "alice", roles = "USER")`: an authenticated user in the `SecurityContext`, without going through the login.
- Per request: `.with(user("alice").roles("USER"))`, `.with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders")))` for a JWT resource server, `.with(csrf())` for a `POST` when CSRF is enabled.
- The application's `SecurityFilterChain` must be loaded: `@Import(SecurityConfig.class)` if it isn't picked up by the slice.

## Repository tests: @DataJpaTest

> **In short**: `@DataJpaTest` loads only JPA and rolls back each test; it tests queries and mappings, ideally against a real PostgreSQL (Testcontainers) rather than the default embedded H2.

`@DataJpaTest` loads only JPA: entities, Spring Data repositories, the `DataSource`, Flyway / Liquibase. Each test is **transactional and rolled back** at the end.

```java
@DataJpaTest
class OrderRepositoryTest {

  @Autowired OrderRepository orderRepository;
  @Autowired TestEntityManager em;

  @Test
  void findsPendingOrdersOfACustomer() {
    em.persist(new Order("alice", OrderStatus.PENDING));
    em.persist(new Order("alice", OrderStatus.PAID));
    em.persist(new Order("bob", OrderStatus.PENDING));
    em.flush();
    em.clear(); // forces the query to hit the database, not the persistence context

    List<Order> orders = orderRepository.findByCustomerAndStatus("alice", OrderStatus.PENDING);

    assertThat(orders).hasSize(1);
  }
}
```

- What it tests: derived queries, `@Query` (JPQL / native), mappings, constraints, entity graphs. Simple `save` / `findById` of Spring Data don't need a test.
- `TestEntityManager`: an `EntityManager` for tests, to insert the data set (`persist`, `persistAndFlush`, `find`).
- `flush()` + `clear()`: without them, the data stays in the persistence context, the SQL constraints aren't checked, and some reads come from the first-level cache (see [Hibernate cache](../hibernate/HIBERNATE_CACHE.md)).
- **Embedded database by default**: if H2 is on the classpath, it replaces the configured database. H2 doesn't behave like PostgreSQL (SQL dialect, native queries, types): prefer a real PostgreSQL with Testcontainers and `@AutoConfigureTestDatabase(replace = Replace.NONE)` (see below).
- Rollback disabled: `@Transactional(propagation = Propagation.NOT_SUPPORTED)` on the class or method.

## Other test slices

> **In short**: Other slices load one technology each: `@RestClientTest` with `MockRestServiceServer`, `@JsonTest` with `JacksonTester`, `@JdbcTest`, `@WebFluxTest`...

| Annotation | Loads | Typical tool |
|---|---|---|
| `@WebMvcTest` | controllers, advices, filters, security | `MockMvcTester` |
| `@DataJpaTest` | JPA, repositories | `TestEntityManager` |
| `@JdbcTest` / `@DataJdbcTest` | `JdbcTemplate`, `JdbcClient` / Spring Data JDBC | |
| `@RestClientTest` | `RestClient.Builder`, Jackson, the given client | `MockRestServiceServer` |
| `@JsonTest` | Jackson, `@JacksonComponent` | `JacksonTester` |
| `@WebFluxTest` | WebFlux controllers | `WebTestClient` |
| `@DataMongoTest`, `@DataRedisTest` | Spring Data MongoDB / Redis | |

**REST client** with a mock server:

```java
@RestClientTest(PaymentClient.class)
class PaymentClientTest {

  @Autowired PaymentClient paymentClient;
  @Autowired MockRestServiceServer server;

  @Test
  void readsAPayment() {
    server.expect(requestTo("https://payment.example.com/payments/1"))
        .andRespond(withSuccess("""
            {"id": 1, "status": "OK"}
            """, MediaType.APPLICATION_JSON));

    assertThat(paymentClient.findById(1).status()).isEqualTo("OK");
  }
}
```

With a `RestClient.Builder`, the expectations use the full URL. For a real HTTP server simulated outside Spring: WireMock.

**JSON**:

```java
@JsonTest
class OrderDtoJsonTest {
  @Autowired JacksonTester<OrderDto> json;

  @Test
  void serializes() throws Exception {
    assertThat(json.write(new OrderDto(42L, "PAID")))
        .extractingJsonPathStringValue("$.status").isEqualTo("PAID");
  }
}
```

A slice scans only its own types: a `@Service` or a `@Component` isn't loaded. Add one with `@Import`, or mock it with `@MockitoBean`.

## Real dependencies with Testcontainers

> **In short**: Testcontainers starts the real database or broker in Docker, and `@ServiceConnection` configures the connection automatically; `@DynamicPropertySource` covers the other properties.

Testcontainers starts the real database (or Kafka, Redis) in a Docker container for the tests. `@ServiceConnection` (Spring Boot 3.1+) configures the connection automatically from the container: no `spring.datasource.url` to set.

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfig {

  @Bean
  @ServiceConnection
  PostgreSQLContainer postgres() {
    return new PostgreSQLContainer("postgres:17");
  }
}

@SpringBootTest
@Import(TestcontainersConfig.class)
class OrderFlowIT { ... }

@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE) // keep PostgreSQL, don't switch to H2
@Import(TestcontainersConfig.class)
class OrderRepositoryTest { ... }
```

- The container is a bean: started once per context, shared by all tests using the same context (see below).
- For a property `@ServiceConnection` doesn't know: `@DynamicPropertySource`.

```java
@DynamicPropertySource
static void properties(DynamicPropertyRegistry registry) {
  registry.add("shop.payment-url", wireMock::baseUrl);
}
```

- Same container at development time: `SpringApplication.from(ShopApplication::main).with(TestcontainersConfig.class).run(args)` in a test `main`, to run the application locally with its database.

## Integration test setup (real server)

> **In short**: A shared base class (`RANDOM_PORT`, `RestTestClient`, Testcontainers) keeps a single context; the database is cleaned with `TRUNCATE` before each test, the tests send real JWTs signed with a test key, and external APIs are simulated with WireMock.

Tests as close to production as possible: real server (`RANDOM_PORT`), real HTTP calls with `RestTestClient`, real PostgreSQL (Testcontainers), database cleaned between tests. A common base class keeps one configuration, so one context and one container for the whole suite:

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@AutoConfigureRestTestClient
@Import({TestcontainersConfig.class, TestJwtConfig.class})
@Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
public abstract class IntegrationTest {
  @Autowired protected RestTestClient client;
  @Autowired protected TestTokens tokens;
}

class OrderApiIT extends IntegrationTest {

  @Test
  void createsAndReadsAnOrder() {
    client.post().uri("/api/orders")
        .header(HttpHeaders.AUTHORIZATION, "Bearer " + tokens.forUser("alice", "orders"))
        .contentType(MediaType.APPLICATION_JSON)
        .body(new CreateOrderRequest("alice", List.of(new OrderLineRequest(1L, 2))))
        .exchange()
        .expectStatus().isCreated();
  }
}
```

**Database cleanup**: the server runs in its own threads and transactions, `@Transactional` on the test can't roll back. With PostgreSQL, `TRUNCATE` is the simplest:

```sql
-- src/test/resources/cleanup.sql
TRUNCATE orders, order_lines, customers RESTART IDENTITY CASCADE;
```

- `CASCADE` handles foreign keys (`deleteAll()` needs the right order, and is slower); `RESTART IDENTITY` resets the sequences, for predictable ids.
- Cleaning **before** each test rather than after: a failed test leaves its data to inspect, and the next test still starts clean.
- Don't truncate reference data inserted by Flyway migrations, nor `flyway_schema_history`.

**Authentication**: `@WithMockUser` doesn't apply (the request runs on a server thread), the test sends a real token. A good compromise: the tests sign their own JWTs with a test key, and the application validates them with that key instead of the real issuer.

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestJwtConfig {

  private final RSAKey rsaKey;

  TestJwtConfig() throws JOSEException {
    rsaKey = new RSAKeyGenerator(2048).generate(); // test key pair (Nimbus)
  }

  @Bean
  JwtDecoder jwtDecoder() throws JOSEException { // replaces the decoder built from issuer-uri
    return NimbusJwtDecoder.withPublicKey(rsaKey.toRSAPublicKey()).build();
  }

  @Bean
  TestTokens testTokens() {
    return new TestTokens(new NimbusJwtEncoder(new ImmutableJWKSet<>(new JWKSet(rsaKey))));
  }
}

public record TestTokens(JwtEncoder encoder) {
  public String forUser(String username, String scope) {
    JwtClaimsSet claims = JwtClaimsSet.builder()
        .subject(username)
        .claim("scope", scope)
        .expiresAt(Instant.now().plusSeconds(300))
        .build();
    return encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
  }
}
```

The security configuration itself (rules, scopes, `401` / `403`) is fully tested. More faithful but heavier: a Keycloak container (Testcontainers) issuing the tokens.

**External APIs**: simulated with **WireMock**, so the outgoing HTTP call stays real (URL, headers, JSON, error handling of the client). Its URL replaces the real one with `@DynamicPropertySource`:

```java
class PaymentIT extends IntegrationTest {

  @RegisterExtension
  static WireMockExtension paymentApi = WireMockExtension.newInstance()
      .options(wireMockConfig().dynamicPort())
      .build();

  @DynamicPropertySource
  static void paymentUrl(DynamicPropertyRegistry registry) {
    registry.add("spring.http.serviceclient.payment.base-url", paymentApi::baseUrl);
  }

  @Test
  void confirmsAnOrderWhenPaymentSucceeds() {
    paymentApi.stubFor(post("/payments").willReturn(okJson("""
        {"id": "tx-1", "status": "OK"}
        """)));
    // ... call the API with client, then:
    paymentApi.verify(postRequestedFor(urlEqualTo("/payments")));
  }
}
```

- A `@MockitoBean` also works with `RANDOM_PORT` (same JVM, same context), but skips the HTTP layer of the client, and each different set of mocks creates a new context.
- `@DynamicPropertySource` in a subclass also changes the context configuration: keep the WireMock setup in the base class if many test classes need it.

## Test configuration and context caching

> **In short**: Spring Test caches and reuses the application context for every test with the same configuration; a different set of `@MockitoBean`, properties or profiles creates a new one, so keep test configurations uniform.

- **`@TestConfiguration`**: beans for tests only, added to the context with `@Import` (not picked up by the component scan of the application).
- **Properties**: `@SpringBootTest(properties = "shop.cache.enabled=false")`, `@TestPropertySource`, or a profile with `@ActiveProfiles("test")` and `application-test.yml`.
- **`@Transactional` on a test**: each test is rolled back (default in `@DataJpaTest`). Pitfall: the test then runs in a single transaction, which hides bugs (lazy loading works where production would throw a `LazyInitializationException`, no flush means no constraint error). Not useful with `RANDOM_PORT`, where the server uses its own transactions.

**Context caching**: Spring Test caches the application context and reuses it for every test class with the same configuration. Loading it once for the whole test suite is what keeps `@SpringBootTest` affordable. A new context is created (and the cache grows) when the configuration differs:

- a different set of `@MockitoBean` / `@MockitoSpyBean`;
- different `properties`, `@ActiveProfiles`, `@Import`, or a different slice;
- `@DirtiesContext`, which discards the context after the test.

Keep the configurations uniform: a common base class or a custom annotation for integration tests, rather than different mocks in each class.
