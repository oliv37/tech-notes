## Servlet and servlet container

> **In short**: A servlet is a Java class handling HTTP requests; the servlet container (Tomcat, Jetty) listens on the port, turns the HTTP request into Java objects, calls the servlet mapped to the URL and manages its lifecycle. In Spring MVC, a single servlet, the `DispatcherServlet`, receives every request.

A servlet is a Java class that handles HTTP requests and produces responses. It doesn't run on its own: a **servlet container** (Tomcat, Jetty, Undertow) listens on the port, parses the HTTP request into Java objects, picks the servlet mapped to the URL, and writes the response back. The API is defined by the Jakarta Servlet spec (`jakarta.servlet.*` since Jakarta EE 9 / Spring Boot 3, `javax.servlet.*` before).

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
  @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
    resp.setContentType("text/plain");
    resp.getWriter().write("Hello " + req.getParameter("name"));
  }
}
```

Lifecycle, managed by the container:

- **`init()`**: called once, when the servlet is created (at first request, or at startup with `loadOnStartup`).
- **`service()`**: called for every request; `HttpServlet` dispatches it to `doGet()`, `doPost()`, `doPut()`, `doDelete()`...
- **`destroy()`**: called once, when the application stops.

Threading: a single instance per servlet, shared by all requests, each request running in its own thread (from Tomcat's pool, ~200 threads by default, or a virtual thread). Hence a servlet must be stateless: an instance field would be shared by all concurrent requests.

In Spring MVC, there's only one servlet: the **`DispatcherServlet`**, mapped to `/`, which receives every request and routes it to the right `@Controller` method. Spring Boot starts an embedded Tomcat and registers it automatically.

## HttpServletRequest and HttpServletResponse

> **In short**: Created by the container for each request: `HttpServletRequest` gives everything the client sent (method, URI, parameters, headers, body readable once), `HttpServletResponse` builds the answer (status, headers, body).

Created by the container for each request, they only live for the duration of that request.

**`HttpServletRequest`**: everything the client sent.

- `getMethod()`, `getRequestURI()`, `getQueryString()`
- `getParameter("name")`: query string or form parameter
- `getHeader("Authorization")`, `getCookies()`
- `getInputStream()` / `getReader()`: the body, readable **only once** (hence `ContentCachingRequestWrapper` to log it)
- `getAttribute()` / `setAttribute()`: data shared between components handling the same request (filter → servlet)
- `getSession()`: the user's `HttpSession` (created if missing, `getSession(false)` to avoid it)

**`HttpServletResponse`**: what is sent back.

- `setStatus(404)`, `setHeader()`, `addCookie()`, `setContentType()`
- `getWriter()` (text) / `getOutputStream()` (binary): the body
- `sendRedirect("/login")`: `302` with a `Location` header

Once the response is **committed** (buffer flushed, headers sent), the status and headers can't be changed anymore: `isCommitted()` tells it, and changing them afterwards is silently ignored or throws `IllegalStateException`.

## Scopes: request, session, ServletContext

> **In short**: Attributes can be stored for one request (`HttpServletRequest`), one user session (`HttpSession`), or the whole application (`ServletContext`).

Three places to store attributes, from the shortest to the longest lifetime:

| Scope       | Object               | Lifetime                                      | Shared between      |
| ----------- | -------------------- | --------------------------------------------- | ------------------- |
| Request     | `HttpServletRequest` | one request                                   | one request         |
| Session     | `HttpSession`        | one user, until timeout or `invalidate()`     | requests of a user  |
| Application | `ServletContext`     | the whole application, from start to shutdown | all users           |

**`ServletContext`**: one per web application, created at startup. It represents the application itself:

- Global attributes (`setAttribute()`), shared by all users and threads, so must be thread-safe.
- Init parameters (`getInitParameter()`), the context path (`getContextPath()`), access to resources (`getResourceAsStream()`).
- Programmatic registration of servlets, filters and listeners at startup (`addServlet()`, `addFilter()`).
- Spring stores its root `WebApplicationContext` in it as an attribute.

**`HttpSession`**: identified by the `JSESSIONID` cookie, stored in the server's memory by default. Hence the sticky sessions or shared store (Spring Session + Redis) needed with several instances. A REST API with JWT is usually stateless and doesn't use it.

`ServletConfig` (not to be confused with `ServletContext`) is per servlet: its name and its own init parameters.

## Listeners

> **In short**: A listener is notified of container lifecycle events: application start and stop (`ServletContextListener`), sessions created or destroyed, requests started or finished.

A listener is notified of lifecycle events of the container, without handling any request itself. Registered with `@WebListener`, in `web.xml`, or as a bean in Spring Boot.

| Listener                     | Events                                                           |
| ---------------------------- | ---------------------------------------------------------------- |
| `ServletContextListener`     | application started / stopped (`contextInitialized`, `contextDestroyed`) |
| `HttpSessionListener`        | session created / destroyed (e.g. counting connected users)     |
| `ServletRequestListener`     | request started / finished                                       |
| `*AttributeListener`         | attribute added / removed / replaced in a scope                  |

```java
@WebListener
public class StartupListener implements ServletContextListener {
  @Override
  public void contextInitialized(ServletContextEvent event) {
    // init resources: cache, connection pool...
  }

  @Override
  public void contextDestroyed(ServletContextEvent event) {
    // release them
  }
}
```

This is how Spring started before Spring Boot: `ContextLoaderListener`, a `ServletContextListener`, created the Spring context at application startup.

## Filters and FilterChain

> **In short**: A filter intercepts requests before the servlet and responses on their way back, for cross-cutting concerns (security, logging, CORS); each filter calls `chain.doFilter()` to pass the request on. `OncePerRequestFilter` runs only once per request, and `DelegatingFilterProxy` lets a Spring bean act as a filter.

A filter intercepts requests **before** they reach the servlet, and responses on their way back. For cross-cutting concerns: authentication, logging, CORS, compression, encoding, adding a correlation id.

Filters form a chain: each one does its work, then calls `chain.doFilter()` to pass the request to the next filter, the last one passing it to the servlet. The code after `doFilter()` runs on the way back, once the servlet has produced the response.

```java
public class LoggingFilter implements Filter {
  @Override
  public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
      throws IOException, ServletException {
    long start = System.currentTimeMillis();
    chain.doFilter(req, resp); // next filter, then the servlet
    log.info("{} ms", System.currentTimeMillis() - start); // on the way back
  }
}
```

```
Request  → Filter A → Filter B → DispatcherServlet → Controller
Response ← Filter A ← Filter B ← DispatcherServlet ←
```

- **Not calling `chain.doFilter()`** stops the request there: the filter answers itself (e.g. `401` when the token is invalid).
- **Wrapping**: a filter can pass a wrapper of the request or response (`HttpServletRequestWrapper`) to modify it, e.g. to read the body twice.
- **Order** matters: `@Order` or `FilterRegistrationBean.setOrder()` in Spring Boot. A filter declared as a `@Component` is registered automatically for every URL; `FilterRegistrationBean` restricts it to some URL patterns (see [Spring Boot](SPRING_BOOT.md)).
- **`OncePerRequestFilter`** (Spring): guarantees a single execution per request, even when the request is forwarded internally (`forward`, error dispatch), where a plain filter could run twice.

Filter vs Spring `HandlerInterceptor`: a filter belongs to the container and runs before the `DispatcherServlet`, for every request (static files included), without knowing the controller. An interceptor runs inside Spring MVC, around the controller method, with access to the `HandlerMethod` (e.g. its annotations). See [Spring Web MVC](SPRING_WEBMVC.md).

**Spring Security** is a single filter registered in the container (`DelegatingFilterProxy`), which delegates to `FilterChainProxy`, holding its own chain of security filters (`SecurityFilterChain`: CORS, CSRF, authentication, authorization...). It handles security before the request reaches the `DispatcherServlet`.
