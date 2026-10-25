## @Controller vs @RestController

> **In short**: `@RestController` is `@Controller` + `@ResponseBody`: the return value is serialized to the response body (JSON), whereas a `@Controller` returns a view name rendered as an HTML template.

`@RestController` = `@Controller` + `@ResponseBody` on every method. The difference is what happens to the return value:

- **`@Controller`**: the returned `String` is a **view name**, resolved by a `ViewResolver` into a template (Thymeleaf, JSP) rendered as HTML.
- **`@RestController`**: the return value is **the response body**, serialized by an `HttpMessageConverter` (JSON with Jackson).

```java
@Controller
public class OrderPageController {

  @GetMapping("/orders/{id}")
  public String show(@PathVariable long id, Model model) {
    model.addAttribute("order", orderService.findById(id));
    return "order-detail"; // view name → templates/order-detail.html
  }

  @GetMapping("/orders/{id}/status")
  @ResponseBody             // this method returns data, not a view
  public OrderStatus status(@PathVariable long id) {
    return orderService.findById(id).status();
  }
}

@RestController
public class OrderApiController {

  @GetMapping("/api/orders/{id}")
  public OrderDto findById(@PathVariable long id) {
    return orderService.findById(id); // → {"id": 42, "status": "PAID", ...}
  }
}
```

| | `@Controller` | `@RestController` |
|---|---|---|
| Return value | view name (`String`, `ModelAndView`) | response body |
| Rendering | `ViewResolver` + template engine | `HttpMessageConverter` (JSON, XML) |
| Data passed by | `Model` | the returned object |
| Typical use | server-side HTML pages | REST API (Angular front-end, other services) |

- Both are `@Component`s, detected by component scan, and handled by the same `DispatcherServlet`; all mapping annotations work the same way.
- Pitfall: in a `@Controller`, an object returned without `@ResponseBody` isn't serialized: Spring puts it in the model and looks for a view named after the URL, which fails (template not found).
- `ResponseEntity<...>` is written to the body in both cases, even without `@ResponseBody`.

## Spring MVC REST controller

> **In short**: A `@RestController` maps HTTP requests to methods with `@GetMapping`, `@PostMapping`...; it binds the request with `@PathVariable`, `@RequestParam` and `@RequestBody`, and controls the status with `ResponseEntity` or `@ResponseStatus`. It returns DTOs, not JPA entities.

A REST controller is a bean whose methods handle HTTP requests: the `DispatcherServlet` finds the method matching the URL and the HTTP method, converts the request into its parameters, and serializes the return value to JSON (Jackson, through an `HttpMessageConverter`).

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

  private final OrderService orderService;

  public OrderController(OrderService orderService) {
    this.orderService = orderService;
  }

  @GetMapping("/{id}")
  public OrderDto findById(@PathVariable long id) {        // GET /api/orders/42
    return orderService.findById(id);
  }

  @GetMapping
  public List<OrderDto> search(@RequestParam(defaultValue = "PENDING") OrderStatus status,
                               @RequestParam(required = false) String customer) {
    return orderService.search(status, customer);          // GET /api/orders?status=PAID
  }

  @PostMapping
  public ResponseEntity<OrderDto> create(@Valid @RequestBody CreateOrderRequest request) {
    OrderDto created = orderService.create(request);
    URI location = URI.create("/api/orders/" + created.id());
    return ResponseEntity.created(location).body(created); // 201 + Location header
  }

  @PutMapping("/{id}")
  public OrderDto update(@PathVariable long id, @Valid @RequestBody UpdateOrderRequest request) {
    return orderService.update(id, request);
  }

  @DeleteMapping("/{id}")
  @ResponseStatus(HttpStatus.NO_CONTENT)                   // 204 instead of 200
  public void delete(@PathVariable long id) {
    orderService.delete(id);
  }
}
```

Annotations:

| Annotation | Role |
|---|---|
| `@RestController` | `@Controller` + `@ResponseBody`: return values are written to the body (JSON), not resolved as a view name |
| `@RequestMapping("/api/orders")` | URL prefix on the class; on a method, any HTTP method (`method = ...`) |
| `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping` | shortcuts for `@RequestMapping(method = ...)` |
| `@PathVariable` | a part of the path: `/{id}` |
| `@RequestParam` | a query parameter (`?status=PAID`) or form field; required by default, `required = false` or `defaultValue` |
| `@RequestBody` | the request body, deserialized from JSON |
| `@RequestHeader("X-Request-Id")`, `@CookieValue` | a header, a cookie |
| `@ResponseStatus` | the response status when the method succeeds |
| `@Valid` / `@Validated` | triggers Bean Validation on the parameter (see below) |
| `@CrossOrigin` | allows CORS calls on this controller (better configured globally) |

- **Mapping attributes**: `consumes` / `produces` restrict the content type (see below), `params` / `headers` add conditions. Since Spring Framework 7, `version = "2"` maps an API version (see [Spring Boot](SPRING_BOOT.md)).
- **`ResponseEntity`**: when the status or the headers depend on the result (`ResponseEntity.ok(...)`, `notFound().build()`, `created(uri)`). Otherwise, return the object directly.
- **Parameter names**: `@PathVariable long id` needs the `-parameters` compiler flag (set by Spring Boot's Maven and Gradle plugins), otherwise write the name: `@PathVariable("id")`.
- Return DTOs (records), not JPA entities: avoids exposing internal fields, `LazyInitializationException` and infinite loops on bidirectional relations.
- Other injectable parameters: `Pageable` (Spring Data, `?page=0&size=20&sort=date,desc`), `Principal` / `@AuthenticationPrincipal`, `HttpServletRequest`.

## Content type: consumes and produces

> **In short**: `consumes` restricts the request `Content-Type` (`415` otherwise), `produces` the response type negotiated with the `Accept` header (`406` otherwise). File uploads use `multipart/form-data` with `MultipartFile` and `@RequestPart`.

`consumes` and `produces`, on a mapping annotation of the class or the method, restrict the content types of a request and its response.

```java
@RestController
@RequestMapping(path = "/api/orders", produces = MediaType.APPLICATION_JSON_VALUE)
public class OrderController {

  @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)  // JSON body only
  public OrderDto create(@RequestBody CreateOrderRequest request) { ... }

  @GetMapping(path = "/{id}/invoice", produces = MediaType.APPLICATION_PDF_VALUE) // replaces the class value
  public byte[] invoice(@PathVariable long id) { ... }

  @GetMapping(path = "/{id}", produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
  public OrderDto findById(@PathVariable long id) { ... } // format chosen from the Accept header
}
```

| | Compared with | Mismatch |
|---|---|---|
| `consumes` | the request `Content-Type` | `415 Unsupported Media Type` |
| `produces` | the request `Accept` | `406 Not Acceptable` |

- A method value **replaces** the class value, it doesn't add to it.
- Without `produces`, Spring negotiates between the `Accept` header and the available converters (JSON in practice): it's only needed to restrict the formats, to map two methods on the same URL (JSON and CSV), or for a non-JSON response.
- Several values, or a negation: `consumes = {"application/json", "application/*+json"}`, `consumes = "!text/plain"`.
- When the type depends on the result, set it on the response:

```java
return ResponseEntity.ok()
    .contentType(MediaType.APPLICATION_PDF)
    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=invoice-42.pdf")
    .body(pdfBytes);
```

File upload: a `multipart/form-data` request holds several parts, each with its own content type: here a file and JSON metadata.

```java
@PostMapping(path = "/{id}/documents", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<DocumentDto> upload(@PathVariable long id,
                                          @RequestPart("file") MultipartFile file,
                                          @RequestPart("metadata") DocumentMetadata metadata)
    throws IOException {
  if (file.isEmpty()) {
    return ResponseEntity.badRequest().build();
  }
  DocumentDto saved = documentService.store(id, file.getOriginalFilename(),
      file.getContentType(), file.getInputStream(), metadata);
  return ResponseEntity.status(HttpStatus.CREATED).body(saved);
}
```

```bash
curl -F "file=@invoice.pdf;type=application/pdf" \
     -F 'metadata={"type":"INVOICE"};type=application/json' \
     http://localhost:8080/api/orders/42/documents
```

- `@RequestPart` converts a part with the converter matching its `Content-Type` (JSON → `DocumentMetadata`). `@RequestParam MultipartFile file` is enough for a file alone.
- `MultipartFile`: `getOriginalFilename()`, `getContentType()`, `getSize()`, `getInputStream()`, `transferTo(path)`. Prefer streaming (`getInputStream()`) over `getBytes()`, which loads the whole file in memory. Several files: `List<MultipartFile> files`.
- Size limits (Spring Boot): `spring.servlet.multipart.max-file-size` (1MB by default) and `max-request-size` (10MB). Exceeding them throws `MaxUploadSizeExceededException` (`413`).
- Don't trust the client: `getOriginalFilename()` may contain a path (`../../etc`), and `getContentType()` is declared by the client. Generate the stored file name, check the actual type.

## Request validation

> **In short**: `@Valid` on a `@RequestBody` triggers Bean Validation (`@NotBlank`, `@Positive`...), and an invalid body gives a `400`; constraints on simple parameters are checked too since Spring 6.1.

Bean Validation (`spring-boot-starter-validation`) checks the request body declared with `@Valid`:

```java
public record CreateOrderRequest(
    @NotBlank String customer,
    @NotEmpty List<@Valid OrderLineRequest> lines,
    @Email String contactEmail) {}

public record OrderLineRequest(@NotNull Long productId, @Positive int quantity) {}
```

- An invalid body throws `MethodArgumentNotValidException` → `400 Bad Request` (response format: see below).
- Constraints on simple parameters (`@PathVariable @Positive long id`, `@RequestParam @Size(max = 50) String q`) are checked too since Spring Framework 6.1 (built-in method validation), throwing `HandlerMethodValidationException`.
- `@Validated` (Spring) also accepts validation groups.

## Error handling

> **In short**: Without a handler, Spring MVC sets the status of its own exceptions (`DefaultHandlerExceptionResolver`), and Spring Boot's `/error` page writes the body, also for unhandled exceptions (`500`). A `@RestControllerAdvice` with `@ExceptionHandler` methods handles the application's errors, typically as `ProblemDetail` (RFC 9457); extending `ResponseEntityExceptionHandler` formats Spring's exceptions the same way.

When a controller throws an exception, the `DispatcherServlet` asks its `HandlerExceptionResolver`s, in order, to turn it into a response:

1. **`ExceptionHandlerExceptionResolver`**: calls a matching `@ExceptionHandler` method (see below).
2. **`ResponseStatusExceptionResolver`**: uses the status of a `ResponseStatusException`, or of `@ResponseStatus` on the exception class.
3. **`DefaultHandlerExceptionResolver`**: maps Spring MVC's own exceptions to a status code.
4. None of them: the exception reaches the servlet container → `500` (see below).

**Default behaviour**, without any handler: `DefaultHandlerExceptionResolver` only knows a fixed list of Spring MVC exceptions (and those implementing `ErrorResponse`). It sets the status with `response.sendError(...)`, without a body:

| Exception | Status |
|---|---|
| `MethodArgumentNotValidException`, `HandlerMethodValidationException` (validation) | `400` |
| `HttpMessageNotReadableException` (malformed JSON), `MissingServletRequestParameterException`, `MethodArgumentTypeMismatchException` (`/orders/abc` for a `long`) | `400` |
| `NoResourceFoundException` (no handler for the URL, Spring 6.1+) | `404` |
| `HttpRequestMethodNotSupportedException` | `405` |
| `HttpMediaTypeNotAcceptableException` / `HttpMediaTypeNotSupportedException` | `406` / `415` |
| `HttpMessageNotWritableException` (response serialization failed), `MissingPathVariableException` | `500` |

Any other exception, e.g. an `IllegalStateException` from a service, isn't resolved: the `DispatcherServlet` rethrows it, it goes up through the filters to Tomcat, which logs it (`Servlet.service() ... threw exception`) and treats it as a `500`.

In both cases, the body comes from the **`/error` page of Spring Boot**:

1. Spring Boot registers `/error` as the global error page of the container (`ErrorPageFilter` for a WAR on an external Tomcat).
2. Tomcat forwards the request internally to `/error` (dispatch type `ERROR`), with the status and the exception as request attributes (`jakarta.servlet.error.status_code`, `jakarta.servlet.error.exception`).
3. The `DispatcherServlet` routes it to **`BasicErrorController`**, which builds the body with `DefaultErrorAttributes`: JSON for an API client, the "Whitelabel" HTML page for a browser.

```json
{ "timestamp": "2026-10-24T10:15:30.000+00:00", "status": 400, "error": "Bad Request", "path": "/api/orders" }
```

- No detail by default (no message, no validation errors, no stack trace), so as not to leak internals: `server.error.include-message`, `include-binding-errors`, `include-stacktrace` change it.
- Customization: an `ErrorAttributes` bean (extending `DefaultErrorAttributes`) changes the content, an `ErrorController` bean replaces `BasicErrorController`.
- With Spring Security 6+, the `ERROR` dispatch is authorized like any request: with `anyRequest().authenticated()`, an anonymous client gets a `401` instead of the actual `400` / `500`. Fix: `.dispatcherTypeMatchers(DispatcherType.ERROR).permitAll()`, or `.requestMatchers("/error").permitAll()`.
- Without Spring Boot, Tomcat would return its own HTML error page. `/error` is a safety net: an API usually handles its errors before, in an advice (see below).

**`@ResponseStatus`** on an exception class gives it a status, without writing a handler:

```java
@ResponseStatus(value = HttpStatus.NOT_FOUND, reason = "Order not found")
public class OrderNotFoundException extends RuntimeException {
  public OrderNotFoundException(long id) {
    super("Order " + id + " not found");
  }
}
```

```java
@GetMapping("/{id}")
public OrderDto findById(@PathVariable long id) {
  return orderService.findById(id).orElseThrow(() -> new OrderNotFoundException(id)); // → 404
}
```

- `ResponseStatusExceptionResolver` calls `sendError(404, reason)`: the body is still the one of `/error` (`"error": "Not Found"`, and `"message": "Order not found"` only with `server.error.include-message=always`). Without `reason`, the exception message is used.
- Also found on a cause: an exception wrapping an `OrderNotFoundException` gets the `404` too.
- Without a dedicated class: `throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Order 42 not found")`.
- Simple, but ties a business exception to HTTP. An `@ExceptionHandler` in an advice keeps the mapping in the web layer and controls the body.

**`@ExceptionHandler`** handles an exception type with a custom response. In a controller, it only applies to that controller; in a **`@RestControllerAdvice`** (`@ControllerAdvice` + `@ResponseBody`), to all controllers (`basePackages` or `assignableTypes` to restrict). This keeps `try/catch` out of the controllers.

```java
@RestControllerAdvice
public class ApiExceptionHandler {

  @ExceptionHandler(OrderNotFoundException.class)
  public ProblemDetail handleNotFound(OrderNotFoundException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    problem.setTitle("Order not found");
    problem.setProperty("orderId", ex.getOrderId()); // custom field
    return problem;
  }

  @ExceptionHandler(Exception.class) // last resort: log it, return a generic 500
  public ProblemDetail handleUnexpected(Exception ex) {
    log.error("Unexpected error", ex);
    return ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
  }
}
```

- The most specific exception type wins: `OrderNotFoundException` goes to the first method, not to `Exception`.
- A handler can return a `ProblemDetail`, a `ResponseEntity<...>` (status and headers) or any object with `@ResponseStatus`.
- Handlers of the controller itself come before the advices; between advices, `@Order` decides.

**`ProblemDetail`** (Spring 6.0+) is the standard JSON error format of RFC 9457, with the `application/problem+json` content type:

```json
{ "type": "about:blank", "title": "Order not found", "status": 404,
  "detail": "Order 42 not found", "instance": "/api/orders/42", "orderId": 42 }
```

- `type`: a URI identifying the kind of error (`about:blank` by default), `title`: short summary (the status reason phrase by default), `status`, `detail`: explanation for this occurrence, `instance`: the request path (filled by Spring).
- `setProperty()` adds custom fields at the top level (`orderId`, a list of validation errors).
- Exceptions can carry their own: `ErrorResponseException` (and the Spring MVC exceptions, which implement `ErrorResponse`) holds a `ProblemDetail`.

**`ResponseEntityExceptionHandler`** is a base class for an advice, with an `@ExceptionHandler` for every Spring MVC exception handled by `DefaultHandlerExceptionResolver`. The difference: it returns a `ProblemDetail` body instead of only a status. Each exception has a `protected` method to override, to customize one case:

```java
@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {

  @Override
  protected ResponseEntity<Object> handleMethodArgumentNotValid(
      MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatusCode status, WebRequest request) {
    ProblemDetail problem = ex.getBody(); // 400, "Invalid request content."
    problem.setProperty("errors", ex.getFieldErrors().stream()
        .map(error -> Map.of("field", error.getField(), "message", error.getDefaultMessage()))
        .toList());
    return handleExceptionInternal(ex, problem, headers, status, request);
  }

  @ExceptionHandler(OrderNotFoundException.class) // own exceptions in the same class
  public ProblemDetail handleNotFound(OrderNotFoundException ex) { ... }
}
```

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "Invalid request content.",
  "instance": "/api/orders", "errors": [ { "field": "customer", "message": "must not be blank" } ] }
```

- Override the method, don't add `@ExceptionHandler(MethodArgumentNotValidException.class)`: it's already mapped in the base class, the duplicate fails at startup (ambiguous handler).
- Without writing a class: `spring.mvc.problemdetails.enabled=true` (Spring Boot, `false` by default) registers a `ProblemDetailsExceptionHandler`, an empty subclass of `ResponseEntityExceptionHandler`. It backs off when the application declares its own (`@ConditionalOnMissingBean`).

Validation outside a controller: `@Validated` on a `@Service` class and `@Valid` / `@NotNull` on its parameters go through an AOP proxy (`MethodValidationPostProcessor`), which throws a `ConstraintViolationException`. Spring MVC doesn't know it: no resolver handles it, the client gets a `500`. Map it to a `400` with an `@ExceptionHandler(ConstraintViolationException.class)`.

Summary:

| Class / annotation | Role |
|---|---|
| `HandlerExceptionResolver` | interface called by the `DispatcherServlet` to turn an exception into a response |
| `ExceptionHandlerExceptionResolver` | calls the `@ExceptionHandler` methods |
| `ResponseStatusExceptionResolver` | `@ResponseStatus` on the exception class, `ResponseStatusException` |
| `DefaultHandlerExceptionResolver` | default status (`400`, `404`, `405`...) for Spring MVC exceptions, no body |
| `BasicErrorController` (Spring Boot) | `/error`: writes the body when no handler did, also for an unhandled `500` |
| `@ExceptionHandler` + `@RestControllerAdvice` | custom handling, for all controllers |
| `ProblemDetail` | standard JSON error body (RFC 9457) |
| `ResponseEntityExceptionHandler` | base class for an advice: Spring MVC exceptions with a `ProblemDetail` body |

In short: without any code, Spring returns the right status with the `/error` JSON. A `@RestControllerAdvice` handles the application's exceptions. Extending `ResponseEntityExceptionHandler` (or `spring.mvc.problemdetails.enabled=true`) gives Spring's exceptions a `ProblemDetail` body too.

## Spring MVC HandlerInterceptor

> **In short**: A `HandlerInterceptor` runs around the controller method inside Spring MVC (`preHandle`, `postHandle`, `afterCompletion`); unlike a servlet filter, it knows the target `HandlerMethod` and its annotations. It's registered with a `WebMvcConfigurer`, with URL patterns.

An interceptor runs inside Spring MVC, around the controller method called by the `DispatcherServlet`. Unlike a servlet filter (see [Servlet](SERVLET.md)), it knows which controller method handles the request.

```java
@Component
public class AuditInterceptor implements HandlerInterceptor {

  @Override
  public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
    if (handler instanceof HandlerMethod method
        && method.hasMethodAnnotation(AdminOnly.class)
        && !isAdmin(req)) {
      resp.setStatus(HttpServletResponse.SC_FORBIDDEN);
      return false; // stops here: the controller isn't called
    }
    req.setAttribute("start", System.currentTimeMillis());
    return true;
  }

  @Override
  public void afterCompletion(HttpServletRequest req, HttpServletResponse resp, Object handler,
                              Exception ex) {
    long duration = System.currentTimeMillis() - (long) req.getAttribute("start");
    log.info("{} {} → {} in {} ms", req.getMethod(), req.getRequestURI(), resp.getStatus(), duration);
  }
}
```

- **`preHandle`**: before the controller. Returning `false` stops the request, like a filter not calling `chain.doFilter()`.
- **`postHandle`**: after the controller, before the view is rendered; not called on exception. With `@RestController`, the JSON body is usually already written: too late to change the body or headers (use a `ResponseBodyAdvice` instead).
- **`afterCompletion`**: always called at the end, even on exception. For logs and cleanup (`ThreadLocal`, MDC).
- `handler` is the target `HandlerMethod`: its annotations can be read, which a filter can't do.

Unlike a filter, a `@Component` isn't enough: it must be registered, with its URL patterns.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
  private final AuditInterceptor auditInterceptor;

  public WebConfig(AuditInterceptor auditInterceptor) {
    this.auditInterceptor = auditInterceptor;
  }

  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(auditInterceptor)
        .addPathPatterns("/api/**")
        .excludePathPatterns("/api/health");
  }
}
```
