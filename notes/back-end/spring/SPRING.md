## IoC and dependency injection

> **In short**: IoC: the framework, not the application code, controls object creation, lifecycle and calls. DI is its application to dependencies: a class declares what it needs, preferably through its constructor, and Spring's container (the `ApplicationContext`) provides it.

**IoC (Inversion of Control)** is the general principle: the application code no longer controls object creation and the program flow, a framework does, and calls the code ("don't call us, we'll call you"). Spring's **IoC container** (the `ApplicationContext`) creates the objects (**beans**), wires them together, manages their lifecycle (`@PostConstruct`, scopes, destruction) and wraps them in proxies (`@Transactional`, `@Async`). Other forms of IoC: a controller called by the `DispatcherServlet`, a `@Scheduled` method, JUnit callbacks.

**DI (Dependency Injection)** is how IoC applies to dependencies: an object doesn't create what it needs, it receives it.

```java
// Without DI: the class picks and builds its dependencies (tight coupling)
public class OrderService {
  private final OrderRepository repository = new JpaOrderRepository();
  private final PaymentClient paymentClient = new StripePaymentClient();
}

// With DI: the class declares its dependencies, Spring provides them
@Service
public class OrderService {
  private final OrderRepository repository;
  private final PaymentClient paymentClient;

  public OrderService(OrderRepository repository, PaymentClient paymentClient) {
    this.repository = repository;
    this.paymentClient = paymentClient;
  }
}
```

- **Decoupling**: `OrderService` depends on interfaces, the implementation is chosen elsewhere (configuration, profile).
- **Testability**: mocks are passed to the constructor, without Spring (see [Mockito](../test/MOCKITO.md)).
- **Shared configuration**: singletons shared by the whole application, with cross-cutting behaviour added by Spring (transactions, security).

| Injection | Example | Use |
|---|---|---|
| **Constructor** | above | recommended: `final` dependencies, always complete object, testable without Spring, circular dependencies detected at startup. `@Autowired` is optional with a single constructor |
| **Setter** | `@Autowired void setMailer(Mailer mailer)` | rare: dependency changed after creation, framework classes with many optional settings |
| **Field** | `@Autowired private Mailer mailer;` | discouraged: no `final`, hidden dependencies, needs reflection in a test without Spring |

How Spring finds the beans:

- Declared by component scan (`@Component`, `@Service`, `@Repository`, `@Controller`) or by `@Bean` methods in a `@Configuration` class.
- Resolved **by type**; with several candidates, `@Qualifier`, `@Primary` or the parameter name decide (see below).
- No candidate, or several without a way to choose: the application fails at startup (`NoSuchBeanDefinitionException`, `NoUniqueBeanDefinitionException`).
- Optional dependency, still in the constructor: `Optional<AuditLog>` (`Optional.empty()` without a bean, Spring 4.1+), `@Nullable AuditLog` (`null`), or `ObjectProvider<AuditLog>` (resolved lazily, `getIfAvailable(() -> AuditLog.NOOP)`). Resolve it right away (`auditLog.orElse(AuditLog.NOOP)`) rather than keeping an `Optional` field. Several candidates without a way to choose still fail.

In short: IoC is the principle (the framework controls object creation and lifecycle), DI is its implementation (dependencies provided from outside, preferably through the constructor), and Spring's IoC container is the `ApplicationContext`.

## Declaring a bean

> **In short**: Stereotype annotations (`@Component`, `@Service`, `@Repository`...) with component scan for the application's own classes, `@Bean` methods in a `@Configuration` for third-party classes or creation logic, and `@Import` / `BeanRegistrar` for advanced cases.

**1. Stereotype annotation + component scan**: the usual way for the application's own classes. Spring scans the packages (from the `@SpringBootApplication` class, or `@ComponentScan("com.example")`) and registers each annotated class.

```java
@Service
public class OrderService { ... }
```

| Annotation | Use |
|---|---|
| `@Component` | generic, the others derive from it |
| `@Service` | business logic; nothing more than readability |
| `@Repository` | data access; also enables exception translation (`SQLException` → `DataAccessException`) |
| `@Controller` / `@RestController` | web layer (see [Spring Web MVC](SPRING_WEBMVC.md)) |
| `@Configuration` | a class declaring `@Bean` methods |

The bean name is the class name in camelCase (`orderService`), unless set: `@Service("orders")`.

**2. `@Bean` method in a `@Configuration` class**: for a class that can't be annotated (third-party library), or whose creation needs code.

```java
@Configuration
public class AppConfig {

  @Bean
  public Clock clock() {
    return Clock.systemUTC();
  }

  @Bean
  public PaymentClient paymentClient(RestClient.Builder builder, Clock clock) { // dependencies as parameters
    return new PaymentClient(builder.baseUrl("https://pay.example.com").build(), clock);
  }

  @Bean(initMethod = "start", destroyMethod = "stop")
  @Profile("prod")
  public CacheWarmer cacheWarmer() { ... }
}
```

- The bean name is the method name (`paymentClient`). Combines with `@Scope`, `@Primary`, `@Qualifier`, `@Lazy`, `@Profile`, `@ConditionalOn...`.
- A `@Configuration` class is proxied (CGLIB): calling `clock()` from another `@Bean` method returns the singleton, not a new instance. With `@Configuration(proxyBeanMethods = false)`, or `@Bean` methods in a plain `@Component` ("lite" mode), each call creates a new object: pass dependencies as parameters instead.

**3. `@Import`**: registers given classes as beans, without relying on the component scan.

```java
@Configuration
@Import({SecurityConfig.class, AuditService.class})
public class AppConfig { }
```

- The imported class doesn't need `@Configuration` (Spring 4.2+): a plain class becomes a bean, and its `@Bean` methods are registered (in lite mode, without `@Configuration`).
- It can also be an `ImportSelector` (returns the classes to import, chosen by code, see below), an `ImportBeanDefinitionRegistrar` or a `BeanRegistrar` (see below).
- Uses: a configuration outside the scanned packages (a library, a shared module), splitting a configuration explicitly, adding a configuration to a test slice (`@WebMvcTest` + `@Import(SecurityConfig.class)`), and the `@Enable...` annotations (`@EnableScheduling`, `@EnableJpaAuditing`), which are an `@Import` of their infrastructure.

**4. Programmatic registration with `BeanRegistrar`** (Spring Framework 7): beans registered by code (conditions, loops), without reflection, AOT-friendly.

```java
class MyBeanRegistrar implements BeanRegistrar {

  @Override
  public void register(BeanRegistry registry, Environment env) {
    registry.registerBean("foo", Foo.class);
    registry.registerBean("bar", Bar.class, spec -> spec
        .prototype()
        .lazyInit()
        .supplier(context -> new Bar(context.bean(Foo.class))));
    if (env.matchesProfiles("baz")) {
      registry.registerBean(Baz.class, spec -> spec.supplier(context -> new Baz("Hello World!")));
    }
  }
}

@Configuration
@Import(MyBeanRegistrar.class)
class MyConfiguration { }
```

Before Spring 7: `GenericApplicationContext.registerBean(...)` in an `ApplicationContextInitializer`, or a lower-level `BeanDefinitionRegistryPostProcessor`.

**5. Special cases**:

- **`FactoryBean<T>`**: a bean that produces another bean, for complex creation, mostly in libraries (`LocalContainerEntityManagerFactoryBean`).
- **Generated beans**: Spring implements an interface, e.g. Spring Data repositories (`interface OrderRepository extends JpaRepository<Order, Long>`) and `@HttpExchange` clients with `@ImportHttpServices` (see [Spring REST client](SPRING_REST_CLIENT.md)).
- **Spring Boot auto-configuration**: `@AutoConfiguration` classes listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, with `@ConditionalOnClass` / `@ConditionalOnMissingBean`.
- **XML**: `<bean id="orderService" class="...">`, legacy only.

In practice: stereotypes for the application's classes, `@Bean` for third-party or parameterized creation, `@Import` / `BeanRegistrar` for advanced cases.

## Several beans of the same type

> **In short**: With several candidates, Spring uses the `@Qualifier` of the injection point, then the `@Primary` bean, then a bean whose name matches the parameter name, otherwise startup fails. `@Qualifier` is the explicit, recommended choice; `@Fallback` marks a default bean to use only when no other one exists.

```java
@Configuration
public class PaymentConfig {
  @Bean PaymentClient stripeClient() { ... }
  @Bean PaymentClient paypalClient() { ... }
}
```

**Default behaviour**: Spring resolves by type, and finds two candidates. Without any annotation, it falls back to the **name**: the bean whose name (the `@Bean` method name, or the class name in camelCase) matches the parameter or field name is injected. Otherwise, the application fails at startup with `NoUniqueBeanDefinitionException`.

```java
public RefundService(PaymentClient paypalClient) { ... } // → paypalClient, by name
public OrderService(PaymentClient client) { ... }        // → NoUniqueBeanDefinitionException
```

This works but is implicit: renaming a parameter silently changes the injected bean.

**Recommended: `@Qualifier`** makes the choice explicit at the injection point:

```java
public RefundService(@Qualifier("paypalClient") PaymentClient client) { ... }
```

Only needed at the injection point: a bean's name is its default qualifier. `@Qualifier("paypal")` on the declaration sets another value (a short name, independent of the method name), or shares one between several beans (`@Qualifier("external") List<PaymentClient>` injects that group). The bean name still works too.

**`@Primary`** marks the default bean, injected when nothing else decides:

```java
@Bean @Primary PaymentClient stripeClient() { ... }

public OrderService(PaymentClient client) { ... }                                  // → stripeClient
public RefundService(@Qualifier("paypalClient") PaymentClient client) { ... }      // → paypalClient
```

`@Primary` and `@Qualifier` combine: the qualifier filters the candidates first, so it always wins over `@Primary`. Typical setup: `@Primary` on the main implementation, `@Qualifier` where another one is needed.

**`@Fallback`** (Spring Framework 6.2+) marks the loser instead of the winner: "pick anyone but me". With two beans, `@Primary` on one or `@Fallback` on the other gives the same result; the difference is who carries the annotation:

```java
// in a library: a default implementation
@Bean @Fallback PaymentClient noopPaymentClient() { ... }
```

- The library's default steps aside by itself: an application declaring its own `PaymentClient` gets it injected, without adding anything. With `@Primary`, the application would have to mark its bean (and two `@Primary` beans of the same type fail).
- It doesn't elect a winner: with A, B and a `@Fallback` C, A and B are still ambiguous, while `@Primary` on A always picks A.
- Unlike Spring Boot's `@ConditionalOnMissingBean`, the fallback bean is still created, and can be injected explicitly with `@Qualifier`.

In short: `@Primary` on the main bean when all declarations are under control, `@Fallback` on a replaceable default (library, shared module).

| | Choice made at | Effect |
|---|---|---|
| Parameter name | injection point (implicit) | default when nothing else is set |
| `@Qualifier` | injection point | explicit choice, wins over everything else |
| `@Primary` | bean declaration | default bean when several match |
| `@Fallback` | bean declaration | bean used only when no other one matches |

Resolution order: `@Qualifier` → `@Primary` → non-`@Fallback` beans → parameter name → `NoUniqueBeanDefinitionException`. All candidates can also be injected: `List<PaymentClient>`, or `Map<String, PaymentClient>` (bean name → bean).

## Spring bean scopes

> **In short**: A bean is a singleton by default (one shared instance, so it must be stateless); `prototype` gives a new instance on each injection, `request` / `session` one per HTTP request / session. A shorter-lived bean injected into a singleton needs a proxy (`@RequestScope`) or an `ObjectProvider` (prototype).

The scope defines how many instances of a bean the container creates, and how long they live. Set with `@Scope` (or shortcuts like `@RequestScope`) on the class or the `@Bean` method.

| Scope         | Instances                                      | Typical use                                 |
| ------------- | ---------------------------------------------- | ------------------------------------------- |
| `singleton`   | one per container (**default**)                | services, repositories, controllers         |
| `prototype`   | a new one each time it's injected or requested | stateful, short-lived objects               |
| `request`     | one per HTTP request                           | request-specific data (current user, trace) |
| `session`     | one per HTTP session                           | user-specific data (shopping cart)          |
| `application` | one per `ServletContext`                       | rarely used, close to `singleton`           |

- A singleton is shared by all threads (one per request): it must be stateless, or its fields thread-safe.
- Spring doesn't manage the end of a prototype: `@PreDestroy` is never called on it.

Pitfall: a shorter-lived bean injected into a singleton is injected **only once**, when the singleton is created, so the singleton keeps the same instance forever.

- **`request` / `session`**: nothing to do with `@RequestScope` / `@SessionScope`, which inject a proxy by default, forwarding each call to the bean of the current request or session. Only a plain `@Scope("request")` needs `proxyMode = ScopedProxyMode.TARGET_CLASS`.
- **`prototype`**: inject an `ObjectProvider<T>` and call `getObject()` when a new instance is needed. A proxy would create a new instance on **every method call**, so two consecutive calls wouldn't hit the same object.

```java
@Component
@Scope("prototype")
public class ReportBuilder { // stateful: accumulates lines
  private final List<String> lines = new ArrayList<>();
  private final MoneyFormatter formatter; // injected by Spring

  public ReportBuilder(MoneyFormatter formatter) {
    this.formatter = formatter;
  }

  public void addLine(Order order) { lines.add(order.id() + ": " + formatter.format(order.total())); }
  public String build() { return String.join("\n", lines); }
}

@Service
public class ReportService { // singleton
  private final ObjectProvider<ReportBuilder> builderProvider;

  public ReportService(ObjectProvider<ReportBuilder> builderProvider) {
    this.builderProvider = builderProvider;
  }

  public String monthlyReport(List<Order> orders) {
    ReportBuilder builder = builderProvider.getObject(); // new instance for this report
    orders.forEach(builder::addLine);
    return builder.build();
  }
}
```

Injecting `ReportBuilder` directly would share a single builder between all calls (and threads), mixing the lines of every report.

A prototype bean is only worth it when the object needs Spring: injected dependencies (`MoneyFormatter` here), `@Value`, or AOP (`@Transactional`...). Without any of that, a plain `new ReportBuilder()` is simpler.

## @Conditional and @Profile

> **In short**: `@Conditional` registers a bean only if a condition evaluated at startup is true; `@Profile` (Spring Framework) is one, based on the active profiles. Spring Boot builds its `@ConditionalOn...` annotations on it.

**`@Conditional`** registers a `@Configuration` class, a `@Component` or a `@Bean` method only if a condition is true. The condition is evaluated **at startup**, when Spring reads the bean definitions, not at each injection.

```java
public class OnLinuxCondition implements Condition {
  @Override
  public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    return context.getEnvironment().getProperty("os.name", "").toLowerCase().contains("linux");
  }
}

@Bean
@Conditional(OnLinuxCondition.class)
FileWatcher inotifyWatcher() { ... }
```

- `ConditionContext` gives access to the `Environment` (properties, profiles), the `BeanFactory`, the `ClassLoader` and the resources.
- Several conditions on the same element combine with AND.
- Spring Boot builds ready-made conditions on it (`@ConditionalOnProperty`, `@ConditionalOnMissingBean`...), see [Spring Boot](SPRING_BOOT.md).

**`@Profile`** (Spring Framework 3.1+, not Spring Boot) is a `@Conditional` testing the active profiles:

```java
@Configuration
@Profile("prod")
public class ProdMailConfig {
  @Bean MailSender mailSender() { return new SmtpMailSender(...); }
}

@Bean
@Profile("!prod") // any profile except prod
MailSender fakeMailSender() { return new LoggingMailSender(); }
```

- Activated with `spring.profiles.active=prod` (property, `-Dspring.profiles.active=prod`, `SPRING_PROFILES_ACTIVE` environment variable), or `environment.setActiveProfiles(...)`; in tests, `@ActiveProfiles("test")`.
- Expressions: `@Profile({"dev", "test"})` (OR), `"prod & eu"`, `"!prod"`.
- Without an active profile, the `default` profile is active: `@Profile("default")` beans are used.
- Profile-specific files (`application-prod.yml`) and profile groups come from Spring Boot (see [Spring Boot](SPRING_BOOT.md)).

## ImportSelector

> **In short**: An `ImportSelector`, passed to `@Import`, chooses by code which configuration classes to import: it's how `@Enable...` annotations pick their configuration. Its deferred variant, run after the application's configurations, is the base of Spring Boot's auto-configuration.

An `ImportSelector` is passed to `@Import` like a class, but isn't registered itself: Spring calls it to get, **by code**, the names of the classes to import. It's how an `@Enable...` annotation chooses its configuration from its attributes.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(NotificationImportSelector.class)
public @interface EnableNotifications {
  Channel channel() default Channel.EMAIL;
}

public class NotificationImportSelector implements ImportSelector {

  @Override
  public String[] selectImports(AnnotationMetadata metadata) { // metadata of the class carrying @EnableNotifications
    Map<String, Object> attributes = metadata.getAnnotationAttributes(EnableNotifications.class.getName());
    Channel channel = (Channel) attributes.get("channel");
    return switch (channel) {
      case EMAIL -> new String[] {EmailNotificationConfig.class.getName()};
      case SMS -> new String[] {SmsNotificationConfig.class.getName(), SmsProviderConfig.class.getName()};
    };
  }
}
```

```java
@SpringBootApplication
@EnableNotifications(channel = Channel.SMS) // imports SmsNotificationConfig and SmsProviderConfig
public class ShopApplication { }
```

- Returns class **names** (`String[]`): the classes can be optional, absent from the classpath, without breaking the selector itself.
- The imported classes are processed like any `@Import`: `@Configuration` classes, plain classes, or other selectors.
- To access the context, the selector can implement `EnvironmentAware`, `BeanFactoryAware`, `ResourceLoaderAware`, or receive them in its constructor.
- In Spring: `@EnableAsync`, `@EnableCaching`, `@EnableTransactionManagement` use one to choose between a proxy-based and an AspectJ configuration (`mode` attribute).

**`DeferredImportSelector`**: a variant called **after** all the `@Configuration` classes of the application have been processed. It's the base of Spring Boot's auto-configuration: `@SpringBootApplication` → `@EnableAutoConfiguration` → `@Import(AutoConfigurationImportSelector.class)`, a deferred selector that reads the `AutoConfiguration.imports` files. Being deferred, the auto-configurations are evaluated once the application's beans are known, so `@ConditionalOnMissingBean` can back off when the application declares its own bean.

| | Returns | Use |
|---|---|---|
| `ImportSelector` | class names to import | choose configurations from annotation attributes, classpath, environment |
| `DeferredImportSelector` | the same, after the application's configurations | defaults that the application can override (auto-configuration) |
| `ImportBeanDefinitionRegistrar` | nothing: registers bean definitions directly | beans built from scanned metadata (Spring Data repositories, `@EnableJpaRepositories`) |
| `BeanRegistrar` (Spring 7) | nothing: registers beans with a typed API | programmatic registration (see above) |

For a simple condition (a property, a profile, a class on the classpath), a `@Conditional...` annotation on a configuration is simpler than a selector.
