## What Spring Boot is

> **In short**: Spring Boot configures and runs a Spring application with almost no setup: auto-configuration based on the classpath and properties, starters, an embedded server in an executable jar, externalized configuration, Actuator. It adds no new programming model: it's still Spring underneath.

Spring Boot is a layer on top of the Spring Framework that makes a Spring application ready to run with almost no configuration. Spring provides the features (dependency injection, transactions, MVC, data access), but leaves the setup to the developer: declare the `DataSource`, the `EntityManagerFactory`, the transaction manager, Jackson, deploy a WAR to a Tomcat server. Spring Boot does that setup, with sensible defaults that can be overridden.

What it adds to Spring:

- **Auto-configuration**: beans are created depending on the classpath and the properties. With the Postgres driver and `spring.datasource.url`, a `DataSource` (HikariCP), an `EntityManagerFactory` and a `JpaTransactionManager` are ready. Each one is conditional (`@ConditionalOnClass`, `@ConditionalOnMissingBean`), so declaring your own bean replaces the default one.
- **Starters**: one dependency per feature (`spring-boot-starter-webmvc`, `spring-boot-starter-data-jpa`) that brings compatible library versions, managed by the Spring Boot BOM.
- **Embedded server**: Tomcat (or Jetty) runs inside the application, packaged as an executable jar: `java -jar app.jar`, no external server.
- **Externalized configuration**: `application.yml`, profiles (`application-prod.yml`), environment variables, `@ConfigurationProperties` to bind them to a typed class.
- **Production features**: Actuator (health checks, metrics, info), observability (Micrometer), Docker / Kubernetes support.
- **Testing**: `@SpringBootTest` and test slices (`@WebMvcTest`, `@DataJpaTest`) that load only part of the context.

```java
@SpringBootApplication // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class ShopApplication {
  public static void main(String[] args) {
    SpringApplication.run(ShopApplication.class, args);
  }
}
```

In short: Spring is the framework, Spring Boot is the opinionated way to configure, package and run it. Spring Boot adds no new programming model: controllers, `@Transactional` and repositories are still plain Spring.

## Spring Boot 4 new features

> **In short**: Spring Boot 4 is based on Spring Framework 7 and Jakarta EE 11: modularized auto-configuration and starters, HTTP service clients, API versioning, built-in retry, JSpecify null safety, Jackson 3, `@MockitoBean` replacing the removed `@MockBean`.

Spring Boot 4.0 (November 2025) is based on Spring Framework 7 and Jakarta EE 11 (Servlet 6.1, JPA 3.2, Tomcat 11). Java 17 remains the minimum, Java 25 is supported.

- **Modularization**: auto-configuration is split into small modules (`spring-boot-webmvc`, `spring-boot-data-jpa`...), one per technology, with matching starters and test starters. Smaller jars, and only the auto-configuration of the technologies actually used. Some starters are renamed: `spring-boot-starter-web` becomes `spring-boot-starter-webmvc`, `spring-boot-starter-aop` becomes `spring-boot-starter-aspectj`. Flyway and Liquibase now need their own starter. `spring-boot-starter-classic` brings everything back, to ease migration.
- **HTTP service clients**: `@HttpExchange` interfaces registered as beans with `@ImportHttpServices`, configured by properties (`spring.http.serviceclient.<group>.base-url`), see [Spring REST client](SPRING_REST_CLIENT.md).
- **API versioning** (from Spring Framework 7): `@GetMapping(path = "/orders", version = "2")`, configured with `spring.mvc.apiversion.*` (version in a header, a parameter, the path or the media type).
- **Built-in retry and concurrency limit** (from Spring Framework 7): `@Retryable` and `@ConcurrencyLimit` replace the Spring Retry project, see below.
- **Null safety**: JSpecify annotations (`@Nullable`, `@NullMarked`) across the Spring portfolio, checked by tools like NullAway and by Kotlin.
- **Jackson 3**: new package `tools.jackson` (except the annotations, still in `com.fasterxml.jackson.annotation`), `JsonMapper` instead of `ObjectMapper` for customization. Jackson 2 is deprecated.
- **Testing**: `@MockBean` / `@SpyBean` are removed, replaced by `@MockitoBean` / `@MockitoSpyBean`. New `RestTestClient` to test REST endpoints. `@SpringBootTest` no longer provides `MockMvc` without `@AutoConfigureMockMvc`.
- **Observability**: `spring-boot-starter-opentelemetry` exports metrics and traces over OTLP.
- **Removed**: Undertow (no Servlet 6.1 support yet), the executable launch scripts.
- **Dependency upgrades**: Spring Security 7, Spring Data 2025.1, Hibernate 7, Spring Kafka 4, JUnit 6, Testcontainers 2.

## Profiles in Spring Boot

> **In short**: `@Profile` and profile activation come from Spring; Spring Boot adds the configuration per profile: `application-prod.yml`, `spring.config.activate.on-profile` sections, profile groups. A profile value overrides the default one.

`@Profile` and profile activation come from Spring Framework (see [Spring](SPRING.md)). Spring Boot adds the configuration loaded per profile:

| | Spring Framework | Spring Boot |
|---|---|---|
| `@Profile("prod")` on a bean or a configuration | ✓ | |
| Activation: `spring.profiles.active`, `@ActiveProfiles` in tests | ✓ | |
| Profile-specific files: `application-prod.yml`, loaded on top of `application.yml` | | ✓ |
| `spring.config.activate.on-profile` in a multi-document YAML | | ✓ |
| Profile groups: `spring.profiles.group.prod=db,metrics` | | ✓ |

```yaml
# application.yml
shop:
  payment-url: http://localhost:8081
---
spring:
  config:
    activate:
      on-profile: prod
shop:
  payment-url: https://payment.example.com
```

A profile-specific value overrides the one of `application.yml`; with several active profiles, the last one wins.

## Registering a servlet filter (FilterRegistrationBean)

> **In short**: Spring Boot registers every `Filter` bean in the container, for all URLs. A `FilterRegistrationBean` controls the URL patterns, the order and the dispatcher types, or disables the automatic registration (`setEnabled(false)`) for a filter used only inside the Spring Security chain.

With the embedded server, Spring Boot registers every `Filter` bean in the container (see [Servlet](SERVLET.md)): a filter declared as a `@Component` applies to every URL (`/*`), with no further configuration.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
public class RequestIdFilter extends OncePerRequestFilter {
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                  FilterChain chain) throws ServletException, IOException {
    MDC.put("requestId", UUID.randomUUID().toString());
    try {
      chain.doFilter(request, response);
    } finally {
      MDC.remove("requestId");
    }
  }
}
```

**`FilterRegistrationBean`** gives full control over the registration: URL patterns, order, dispatcher types, name, init parameters.

```java
@Configuration
public class FilterConfig {

  @Bean
  FilterRegistrationBean<AuditFilter> auditFilter(AuditLog auditLog) {
    FilterRegistrationBean<AuditFilter> registration = new FilterRegistrationBean<>(new AuditFilter(auditLog));
    registration.addUrlPatterns("/api/*");               // servlet patterns, not Ant patterns: /api/* not /api/**
    registration.setOrder(1);                            // lower = earlier
    registration.setDispatcherTypes(DispatcherType.REQUEST); // default: REQUEST only
    registration.setName("auditFilter");
    return registration;
  }
}
```

Same thing with an annotation on the `@Bean` method (Spring Boot 3.5+): `@FilterRegistration(name = "auditFilter", urlPatterns = "/api/*", order = 1)`.

When to use it:

- **Restrict a filter to some URLs**: a `@Component` filter applies everywhere.
- **Order a filter whose class can't be changed** (a third-party filter): the order comes from `@Order` / `Ordered` on the filter class, or from `setOrder()`. `@Order` on a `@Bean` method has no effect on a filter.
- **Disable the automatic registration**: a filter bean added to the Spring Security chain (`http.addFilterBefore(jwtFilter, ...)`) is also registered by Spring Boot in the container, so it runs twice (once only with `OncePerRequestFilter`, but still outside the security chain's URL matching). Fix:

```java
@Bean
FilterRegistrationBean<JwtFilter> jwtFilterRegistration(JwtFilter filter) {
  FilterRegistrationBean<JwtFilter> registration = new FilterRegistrationBean<>(filter);
  registration.setEnabled(false); // not registered in the container, only in the security chain
  return registration;
}
```

- Debugging: `logging.level.web=debug` logs every registered filter with its order and URL patterns at startup.
- Spring Security's own chain is registered the same way, as one filter (`DelegatingFilterProxyRegistrationBean`, order `-100` by default, `spring.security.filter.order`): a filter with a lower order runs before security.
- Same principle for servlets and listeners: `ServletRegistrationBean`, `ServletListenerRegistrationBean`. `@WebFilter` / `@WebServlet` classes are registered with `@ServletComponentScan`.

## Conditional auto-configuration (@ConditionalOn...)

> **In short**: Auto-configurations are `@AutoConfiguration` classes whose beans are created under conditions: `@ConditionalOnClass`, `@ConditionalOnProperty`, `@ConditionalOnMissingBean`... `@ConditionalOnMissingBean` lets the application replace a default bean simply by declaring its own.

Spring Boot's auto-configuration is a set of `@AutoConfiguration` classes (Boot 2.7+, meta-annotated with `@Configuration(proxyBeanMethods = false)`, with `before` / `after` attributes to order them), listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Their beans are created only under conditions, built on Spring's `@Conditional` (see [Spring](SPRING.md)):

| Annotation | The bean is created if... |
|---|---|
| `@ConditionalOnProperty(name = "shop.cache.enabled", havingValue = "true")` | the property has this value (`matchIfMissing` when absent) |
| `@ConditionalOnBooleanProperty("shop.cache.enabled")` (Boot 3.5+) | the property is `true`, shorter |
| `@ConditionalOnClass(DataSource.class)` / `@ConditionalOnMissingClass` | a class is / isn't on the classpath |
| `@ConditionalOnBean` / `@ConditionalOnMissingBean` | a bean of this type exists / doesn't exist yet |
| `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | the type of application |
| `@ConditionalOnResource`, `@ConditionalOnExpression`, `@ConditionalOnJava`, `@ConditionalOnCloudPlatform` | a resource, a SpEL expression, a Java version, a cloud platform |

How Boot provides a default `DataSource` that the application can replace:

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean // not created if the application declares its own DataSource
  DataSource dataSource(DataSourceProperties props) { ... }
}
```

- `@ConditionalOnBean` / `@ConditionalOnMissingBean` only see the beans registered **so far**: reliable in an auto-configuration, processed after the application's configurations (deferred import, see [Spring](SPRING.md)), but order-dependent in the application's own configuration.
- `@ConditionalOnClass` on a class reads the annotation without loading the class (bytecode metadata). On a `@Bean` method, prefer the name (`@ConditionalOnClass(name = "com.zaxxer.hikari.HikariDataSource")`), or a nested configuration class.
- Debugging: `debug=true` (or `--debug`) prints the **conditions evaluation report**: which auto-configurations applied, and why. Also available at `/actuator/conditions`.

In short: `@Conditional` (Spring) registers a bean depending on a condition evaluated at startup, `@Profile` being one; Spring Boot's `@ConditionalOn...` are the mechanism of auto-configuration, and `@ConditionalOnMissingBean` lets the application replace a default bean by declaring its own.

## Spring retry (@Retryable)

> **In short**: Since Spring Framework 7, `@Retryable` (enabled by `@EnableResilientMethods`) or `RetryTemplate` call a failing method again, with backoff, for transient errors. It's proxy-based, for idempotent operations only, and must wrap the transaction so that each attempt runs in a new one.

Since Spring Framework 7 (Spring Boot 4), retry is built into the core (`org.springframework.resilience`): a failing method is called again automatically, for transient errors (timeout, `503`, lost connection).

```java
@Configuration
@EnableResilientMethods // enables @Retryable and @ConcurrencyLimit
class AppConfig {}

@Service
public class NotificationService {

  @Retryable(
      includes = MessageDeliveryException.class, // only these exceptions (and their causes)
      maxRetries = 4,  // 1 initial call + 4 retries
      delay = 100,     // 100 ms before the 1st retry
      multiplier = 2,  // exponential backoff: 100, 200, 400, 800 ms
      maxDelay = 1000,
      jitter = 10)     // random ± 10 ms, so clients don't all retry at the same time
  public void send(Notification notification) { ... }
}
```

- Defaults: every exception is retried, 3 retries, 1 second between attempts. `excludes` and `predicate` refine which exceptions are retried.
- When all attempts fail, the last exception is propagated to the caller.
- Also works on reactive methods (`Mono` / `Flux`).

Programmatic version, without annotation:

```java
var retryTemplate = new RetryTemplate(RetryPolicy.builder()
    .includes(MessageDeliveryException.class)
    .maxRetries(4)
    .delay(Duration.ofMillis(100))
    .multiplier(2)
    .build());

retryTemplate.invoke(() -> client.send(notification));
```

Pitfalls:

- **Proxy-based**: a call from inside the same class (`this.send()`) isn't retried, like `@Transactional`.
- **Idempotence**: only retry operations that are safe to run twice (a payment whose first call may have succeeded isn't).
- **Only transient errors**: a `4xx` or a business exception fails the same way every time.
- **With `@Transactional`**: the retry must wrap the transaction, so each attempt runs in a new one (see below).

Retry and transaction: Hibernate sends the `UPDATE`s at flush time, usually at commit, which the `@Transactional` proxy runs **after** the method returns. A retry inside the transaction never sees the error: an optimistic locking failure (`@Version`, the typical case) is thrown at commit, outside the retry. And even when the error is thrown inside (explicit `flush()`), the transaction is already lost: marked rollback-only, aborted by the database (PostgreSQL), and the persistence context still holds the stale entity. Each attempt needs a new transaction that reloads the data. Simplest setup: the retry in one bean, the transaction in another, so each attempt goes through the transactional proxy.

```java
@Service
public class OrderFacade { // must NOT be @Transactional, or the attempts would join its transaction
  private final OrderService orderService;

  @Retryable(includes = ObjectOptimisticLockingFailureException.class, maxRetries = 3, delay = 50)
  public void confirm(long orderId) {
    orderService.confirm(orderId); // new transaction for each attempt
  }
}

@Service
public class OrderService {
  @Transactional
  public void confirm(long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow(); // reloaded on each attempt
    order.confirm();
  }
}
```

Programmatic alternative in a single method: `retryTemplate.invoke(() -> transactionTemplate.executeWithoutResult(status -> ...))`.

Related: `@ConcurrencyLimit(10)` limits concurrent calls to a method, e.g. to protect a fragile API (useful with virtual threads).
