## Mockito: what it is and how it works

Mockito creates **mocks**: fake implementations of a class's dependencies, to test that class in isolation. Testing an `OrderService` shouldn't need a real database or a real payment API: its `OrderRepository` and `PaymentClient` are replaced by mocks whose answers are defined by the test (**stubbing**), and whose calls can be checked afterwards (**verification**).

How it works: at runtime, Mockito generates a class (with ByteBuddy) that intercepts every method call on the mock. By default, a method of a mock does nothing and returns an "empty" value: `null`, `0`, `false`, an empty collection or `Optional.empty()`. Stubbing records "for this method with these arguments, return this"; every call is also recorded, so the test can verify it. The component creating mocks is the **mock maker**:

- **Subclass mock maker** (the old default): generates a subclass overriding every method to intercept it. Can't mock what can't be extended or overridden: `final` classes (records, Kotlin classes), `final`, `static` and `private` methods.
- **Inline mock maker** (default since Mockito 5): modifies the bytecode of the real class at runtime (Java instrumentation API), inserting at the start of each method a check "if `this` is a mock, ask Mockito". No subclass, so `final` is no longer a problem, and static methods and constructors can be mocked too (`mockStatic()`, `mockConstruction()`). It loads a Java agent dynamically, which triggers a warning since JDK 21 (and will be disallowed by default in future JDKs): declare Mockito as `-javaagent` in the test plugin (Surefire, Gradle) instead.

Creating a mock doesn't call any constructor: Mockito allocates the instance directly (Objenesis), so the constructor code and field initializers never run, and fields stay `null` / `0`. Static initializers do run, since the class is loaded normally. Only a spy runs the constructor: `spy(new ArrayList<>())` wraps an object built by the test.

## Basic usage: stubbing and verification

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

  @Mock OrderRepository repository;
  @Mock PaymentClient paymentClient;
  @InjectMocks OrderService service; // real instance, created with the mocks (constructor injection)

  @Test
  void paysAndConfirmsTheOrder() {
    // given: stubbing
    Order order = new Order(42L, 100.0);
    when(repository.findById(42L)).thenReturn(Optional.of(order));
    when(paymentClient.pay(100.0)).thenReturn(PaymentResult.success("tx-1"));

    // when
    service.confirm(42L);

    // then: verification
    verify(paymentClient).pay(100.0);
    verify(repository).save(order);
  }

  @Test
  void failsWhenOrderDoesNotExist() {
    when(repository.findById(42L)).thenReturn(Optional.empty());

    assertThrows(OrderNotFoundException.class, () -> service.confirm(42L));
    verify(paymentClient, never()).pay(anyDouble()); // nobody was charged
  }
}
```

- `thenReturn(a, b)` returns `a` on the first call, then `b`; `thenThrow(new TimeoutException())` simulates a failure.
- `verify(mock, times(2))`, `never()`, `atLeastOnce()` check how many times a method was called; `verifyNoMoreInteractions(mock)` checks nothing else was called.
- Without the extension: `OrderRepository repository = mock(OrderRepository.class);`.
- BDD style, same features: `given(repository.findById(42L)).willReturn(...)` and `then(paymentClient).should().pay(100.0)` (`BDDMockito`).

## Argument matchers and captors

When the exact argument doesn't matter, or is built inside the tested code, matchers replace the values:

```java
when(repository.findByEmail(anyString())).thenReturn(Optional.empty());
verify(mailer).send(eq("a@b.com"), any(Mail.class));
verify(paymentClient).pay(argThat(amount -> amount > 0));
```

Rule: if one argument uses a matcher, all of them must (`eq()` wraps a plain value), otherwise `InvalidUseOfMatchersException`.

Main matchers (`ArgumentMatchers`, `AdditionalMatchers` for comparisons):

```java
// any value
any()                         // anything, including null
any(User.class), isA(User.class) // an instance of User, not null
anyString(), anyInt(), anyLong(), anyDouble(), anyBoolean() // not null (or the primitive)
anyList(), anySet(), anyMap(), anyCollection()

// exact values and null
eq("a@b.com")                 // equals(), needed when mixing with other matchers
same(order)                   // same instance (==)
isNull(), isNotNull()
nullable(User.class)          // a User or null

// strings
startsWith("ORD-"), endsWith(".pdf"), contains("error"), matches("[A-Z]{3}-\\d+")

// custom condition
argThat(user -> user.isActive() && user.getEmail().endsWith("@shop.com"))
longThat(id -> id > 0)        // primitive versions (intThat, doubleThat...) avoid unboxing a null

// comparisons and combinations (AdditionalMatchers)
gt(100.0), lt(0), geq(18), leq(65)
and(gt(0), lt(100)), or(eq("FR"), eq("BE")), not(eq("admin"))
aryEq(new byte[] {1, 2})      // array content, not reference

// reflection equality, ignoring some fields
refEq(expectedUser, "id", "createdAt")
```

When to use `any()`: a matcher accepts wrong arguments too, so overusing it hides bugs.

- **Stubbing**: prefer exact values when the test knows them. `when(repository.findById(42L))` also acts as an implicit check: if the code calls `findById(0L)` by mistake, the stub doesn't match and the test fails (strict stubs even report the mismatch), while `findById(any())` would let it pass. `any()` is fine when the argument is irrelevant or built inside the tested code, or to simulate a failure for any input (`when(client.call(any())).thenThrow(...)`).
- **`verify`**: check the values that matter (`verify(paymentClient).pay(100.0)`), with a captor or `argThat()` for complex objects.
- **`never()`**: the opposite, `any()` is the right choice. `verify(paymentClient, never()).pay(100.0)` only proves there was no call **with 100.0**, and passes if `pay(50.0)` was called. `verify(paymentClient, never()).pay(anyDouble())`, or `verifyNoInteractions(paymentClient)`, proves there was no call at all.
- Prefer the typed matchers: `any()` matches `null`, while `any(Mail.class)`, `anyString()`, `anyLong()` reject it and check the type.

An `ArgumentCaptor` captures the argument passed to a mock, to check its content in detail, typically an object created by the tested code:

```java
@Captor ArgumentCaptor<Mail> mailCaptor;

@Test
void sendsAConfirmationMail() {
  service.confirm(42L);

  verify(mailer).send(mailCaptor.capture());
  Mail mail = mailCaptor.getValue();
  assertEquals("a@b.com", mail.to());
  assertTrue(mail.body().contains("Order 42"));
}
```

To check some fields of an argument passed to a stubbed method (e.g. `save()`), separate the two roles: a permissive stub only makes the code work, the assertions are done afterwards on the captured argument.

```java
@Captor ArgumentCaptor<User> userCaptor;

// in the test
when(userRepository.save(any(User.class))).thenAnswer(invocation -> invocation.getArgument(0));

service.register("a@b.com");

verify(userRepository).save(userCaptor.capture());
assertThat(userCaptor.getValue())
    .extracting(User::getEmail, User::isActive)
    .containsExactly("a@b.com", true);
// or: .usingRecursiveComparison().ignoringFields("id", "createdAt").isEqualTo(expectedUser)
```

- A failure points to the wrong field (`expected: "a@b.com" but was: "A@B.COM"`). With `when(userRepository.save(argThat(user -> ...)))` instead, a wrong field means the stub doesn't match, `save()` returns `null`, and the test fails later with an unrelated `NullPointerException`.
- Avoid assertions inside a `thenAnswer`: the `AssertionError` is thrown inside `save()`, where a `catch (Throwable)` in the tested code can swallow it.

## Void methods, answers and spies

`when(...)` can't wrap a `void` method (there's no return value to pass to it): the `do...` family is used instead, with the call written after.

```java
doThrow(new MailException()).when(mailer).send(any());
doNothing().when(auditLog).record(any()); // the default, rarely needed
```

`thenAnswer` computes the result from the actual arguments, e.g. a `save()` returning the entity it received with an id:

```java
when(repository.save(any(Order.class))).thenAnswer(invocation -> {
  Order order = invocation.getArgument(0);
  return order.withId(1L);
});
```

A **spy** wraps a real object: methods call the real code, unless stubbed. Useful to stub one method of a legacy class, but often a sign the class does too much. Stub a spy with `doReturn(...).when(spy).method()`, since `when(spy.method())` would call the real method during stubbing.

```java
List<String> list = spy(new ArrayList<>());
list.add("a");                      // real call
doReturn(100).when(list).size();    // stubbed
```

## Strictness, static mocks and Spring

- **Strict stubs**: with `MockitoExtension`, a stubbing never used by the test fails it (`UnnecessaryStubbingException`), and a stubbed method called with different arguments is reported. This keeps tests free of leftover stubs. `lenient().when(...)` relaxes it for a single stubbing.
- **Static methods** (inline mock maker): scoped to a try-with-resources block, then restored.

```java
try (MockedStatic<UUID> uuid = mockStatic(UUID.class)) {
  uuid.when(UUID::randomUUID).thenReturn(fixedUuid);
  // code calling UUID.randomUUID() gets fixedUuid here
}
```

Often better avoided by injecting the dependency instead (a `Clock` rather than mocking `LocalDate.now()`, a `Supplier<UUID>`).

- **In Spring tests**: `@MockitoBean` (Spring Framework 6.2+) replaces a bean of the application context with a mock, e.g. in a `@WebMvcTest`. It replaces Spring Boot's `@MockBean`, deprecated since Spring Boot 3.4. `@MockitoSpyBean` does the same with a spy.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
  @Autowired MockMvc mockMvc;
  @MockitoBean OrderService orderService; // the controller gets this mock
}
```

What not to mock: value objects and simple data (build real ones), and ideally types you don't own (a third-party API): wrap them behind your own interface and mock it, or test them with an integration test (WireMock, Testcontainers). Verify only interactions that matter (a payment made, a mail sent), not every call, otherwise tests break on each refactoring.
