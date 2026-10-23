## Spring MVC HandlerInterceptor

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
