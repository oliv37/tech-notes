## What Spring Security is

Spring Security secures a Spring application, mainly at the HTTP level:

- **Authentication**: who is calling? (login/password, JWT token, OAuth2 login, certificate).
- **Authorization**: is this caller allowed to do this? (URL rules, `@PreAuthorize` on methods).
- **Protection against common attacks**: CSRF, session fixation, clickjacking and other security headers.

|          | Authentication                                     | Authorization                          |
| -------- | -------------------------------------------------- | -------------------------------------- |
| Question | who are you?                                       | are you allowed?                       |
| Failure  | `401 Unauthorized`                                 | `403 Forbidden`                        |
| Done by  | an authentication filter + `AuthenticationManager` | `AuthorizationFilter`, method security |

With Spring Boot, adding `spring-boot-starter-security` secures everything by default: every endpoint requires authentication, with a form login and HTTP Basic, a single user `user` with a password generated and printed at startup (`spring.security.user.name` / `password` to set them). Declaring a `SecurityFilterChain` bean replaces this default configuration (see below).

## Architecture: the security filter chain

Spring Security is a set of servlet filters (see [Servlet](SERVLET.md)), run before the `DispatcherServlet`:

```text
Request → [container filters] → DelegatingFilterProxy → FilterChainProxy → SecurityFilterChain (filters 1..n) → DispatcherServlet → controller
```

- **`DelegatingFilterProxy`**: a standard servlet filter registered in the container, that delegates to a Spring bean (`springSecurityFilterChain`). It bridges the container and the Spring context.
- **`FilterChainProxy`**: that bean. It picks the first `SecurityFilterChain` matching the request and runs its filters.
- **`SecurityFilterChain`**: an ordered list of security filters, built with `HttpSecurity`. An application can have several chains, e.g. one for `/api/**` and one for the rest (see below).

Each feature enabled on `HttpSecurity` adds its filter: `httpBasic()` adds `BasicAuthenticationFilter`, `csrf()` adds `CsrfFilter`. The filter list is logged at startup at DEBUG level, and each filter invocation at TRACE level, useful to find where a `401` or `403` comes from:

```properties
logging.level.org.springframework.security=TRACE
```

## Main security filters

In order (the default chain of Spring Boot with form login and HTTP Basic):

| Filter                                                                  | Role                                                                                                                  |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `DisableEncodeUrlFilter`                                                | prevents the session id from being added to URLs                                                                      |
| `WebAsyncManagerIntegrationFilter`                                      | propagates the `SecurityContext` to async requests (`Callable`)                                                       |
| `SecurityContextHolderFilter`                                           | loads the `SecurityContext` (from the session) into the `SecurityContextHolder` for the request, clears it at the end |
| `HeaderWriterFilter`                                                    | adds security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Cache-Control`, HSTS on HTTPS                    |
| `CorsFilter`                                                            | handles CORS preflight requests (if `cors()` is enabled)                                                              |
| `CsrfFilter`                                                            | checks the CSRF token on `POST`, `PUT`, `DELETE`...                                                                   |
| `LogoutFilter`                                                          | handles `POST /logout`: clears the session and the context                                                            |
| `UsernamePasswordAuthenticationFilter`                                  | form login: authenticates `POST /login` (`username`, `password`)                                                      |
| `DefaultLoginPageGeneratingFilter`, `DefaultLogoutPageGeneratingFilter` | generate the default login / logout pages                                                                             |
| `BearerTokenAuthenticationFilter`                                       | reads `Authorization: Bearer <token>` (OAuth2 resource server, JWT)                                                   |
| `BasicAuthenticationFilter`                                             | reads `Authorization: Basic base64(user:password)`                                                                    |
| `RequestCacheAwareFilter`                                               | after login, replays the request saved before the redirect to the login page                                          |
| `SecurityContextHolderAwareRequestFilter`                               | implements `request.isUserInRole()`, `getUserPrincipal()`                                                             |
| `AnonymousAuthenticationFilter`                                         | if nobody is authenticated, sets an anonymous `Authentication`                                                        |
| `SessionManagementFilter`                                               | session fixation protection, concurrent session control (when configured)                                             |
| `ExceptionTranslationFilter`                                            | turns security exceptions thrown further down into `401` / `403` (see below)                                          |
| `AuthorizationFilter`                                                   | last filter: checks the `authorizeHttpRequests` rules, throws `AccessDeniedException` if refused                      |

The authentication filters only try to authenticate the request when their credentials are present: without an `Authorization` header, `BasicAuthenticationFilter` does nothing, and the request reaches `AuthorizationFilter` as anonymous. The full order is defined in `FilterOrderRegistration`.

## SecurityContext and Authentication

The authenticated user is stored in the **`SecurityContextHolder`**, a `ThreadLocal` by default, filled by the filters for the duration of the request:

```text
SecurityContextHolder → SecurityContext → Authentication
                                             ├─ principal   (the user: UserDetails, Jwt...)
                                             ├─ credentials (password, erased after authentication)
                                             └─ authorities (ROLE_ADMIN, SCOPE_orders.read...)
```

```java
// anywhere in the code (same thread)
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();

// in a controller: injected directly
@GetMapping("/me")
public UserDto me(@AuthenticationPrincipal UserDetails user) {
  return new UserDto(user.getUsername());
}
```

- `ThreadLocal`: the context isn't propagated to another thread (`@Async`, `CompletableFuture`) unless wrapped (`DelegatingSecurityContextExecutor`).
- Between requests, the context is kept in the HTTP session (stateful login), or rebuilt on each request from a token (stateless API).

## Configuring a SecurityFilterChain

Since Spring Security 6, the configuration is a `SecurityFilterChain` bean, with the lambda DSL (`WebSecurityConfigurerAdapter` is removed; `and()` and `authorizeRequests()` are removed in Spring Security 7).

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

  @Bean
  SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**", "/actuator/health").permitAll()
            .requestMatchers(HttpMethod.DELETE, "/api/orders/**").hasRole("ADMIN")
            .requestMatchers("/api/**").hasAnyRole("USER", "ADMIN")
            .anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .httpBasic(Customizer.withDefaults());
    return http.build();
  }
}
```

- Rules are checked **in order, first match wins**: specific rules first, `anyRequest()` last.
- `permitAll()`, `authenticated()`, `hasRole("ADMIN")` (checks the `ROLE_ADMIN` authority), `hasAuthority("orders:write")`, `denyAll()`, `access(...)` for a custom `AuthorizationManager`.
- Roles vs authorities: a role is an authority prefixed with `ROLE_`. `hasRole("ADMIN")` = `hasAuthority("ROLE_ADMIN")`.
- With Spring Boot, `@EnableWebSecurity` is optional: the auto-configuration enables it.

Several chains, e.g. a stateless API and a web application with a login form: `securityMatcher` limits each chain to some URLs, and `@Order` decides which one is tried first.

```java
@Bean
@Order(1)
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
  http.securityMatcher("/api/**")
      .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
      .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
  return http.build();
}

@Bean
@Order(2)
SecurityFilterChain webChain(HttpSecurity http) throws Exception {
  http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
      .formLogin(Customizer.withDefaults());
  return http.build();
}
```

## Authentication: AuthenticationManager, UserDetailsService, PasswordEncoder

An authentication filter extracts the credentials from the request and passes them to the **`AuthenticationManager`**:

```text
UsernamePasswordAuthenticationFilter
  → AuthenticationManager (ProviderManager)
      → AuthenticationProvider (DaoAuthenticationProvider)
          → UserDetailsService.loadUserByUsername("alice")   loads the user
          → PasswordEncoder.matches(raw, hash)                checks the password
  → authenticated Authentication stored in the SecurityContext
```

- **`ProviderManager`**: the usual `AuthenticationManager`, delegates to a list of `AuthenticationProvider`s, the first one supporting the type of `Authentication` handles it.
- **`AuthenticationProvider`**: one authentication method (`DaoAuthenticationProvider` for login/password from a `UserDetailsService`, `JwtAuthenticationProvider` for JWT).
- **`UserDetailsService`**: loads a user by username; the application implements it to read its users from the database:

```java
@Service
public class DbUserDetailsService implements UserDetailsService {
  private final UserRepository userRepository;

  @Override
  public UserDetails loadUserByUsername(String username) {
    AppUser user = userRepository.findByEmail(username)
        .orElseThrow(() -> new UsernameNotFoundException(username));
    return User.withUsername(user.getEmail())
        .password(user.getPasswordHash())
        .roles(user.getRole().name()) // → ROLE_USER
        .build();
  }
}

@Bean
PasswordEncoder passwordEncoder() {
  return PasswordEncoderFactories.createDelegatingPasswordEncoder(); // bcrypt by default
}
```

- Declaring a `UserDetailsService` bean and a `PasswordEncoder` bean is enough: Spring Security builds the `DaoAuthenticationProvider` with them.
- **Passwords are hashed, never encrypted**: BCrypt, Argon2, scrypt, PBKDF2 (slow hashes, with a salt). The delegating encoder stores the algorithm in the hash (`{bcrypt}$2a$10$...`), so the algorithm can change later without breaking existing passwords.
- `UsernameNotFoundException` and a wrong password both end as `BadCredentialsException`: the response doesn't reveal whether the user exists.

## Method security

Rules on service or controller methods, with the current user and the arguments:

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig { }

@Service
public class OrderService {

  @PreAuthorize("hasRole('ADMIN')")
  public void delete(long orderId) { ... }

  @PreAuthorize("#customerId == authentication.name")       // method argument
  public List<Order> findByCustomer(String customerId) { ... }

  @PostAuthorize("returnObject.owner == authentication.name") // checks the result
  public Order findById(long id) { ... }
}
```

- `@PreAuthorize`: checked before the call; `@PostAuthorize`: after, on `returnObject`; `@PreFilter` / `@PostFilter`: filter collections. `@Secured("ROLE_ADMIN")` and `@RolesAllowed` are simpler, without SpEL.
- Proxy-based (AOP), like `@Transactional`: a call from the same class (`this.delete()`) isn't checked.
- A refused call throws `AccessDeniedException` → `403`.
- URL rules protect entry points; method security protects the business logic whatever the caller (controller, scheduler, message listener).

## 401 vs 403: ExceptionTranslationFilter

`AuthorizationFilter` (or method security) throws an exception, and `ExceptionTranslationFilter`, placed just before it, turns it into a response:

| Exception                 | Caller                                 | Handled by                 | Default response                                                           |
| ------------------------- | -------------------------------------- | -------------------------- | -------------------------------------------------------------------------- |
| `AuthenticationException` | not authenticated (or bad credentials) | `AuthenticationEntryPoint` | `401` (API, with `WWW-Authenticate`), or redirect to `/login` (form login) |
| `AccessDeniedException`   | anonymous                              | `AuthenticationEntryPoint` | `401`: authenticate first                                                  |
| `AccessDeniedException`   | authenticated, missing authority       | `AccessDeniedHandler`      | `403`                                                                      |

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)) // 401 instead of a redirect
    .accessDeniedHandler((request, response, e) -> response.sendError(HttpServletResponse.SC_FORBIDDEN)));
```

- These exceptions are thrown in the filters, before the `DispatcherServlet`: a `@RestControllerAdvice` doesn't see them (except an `AccessDeniedException` from method security inside a controller, which the advice can intercept: rethrow it, or let it through).
- The `ERROR` dispatch to Spring Boot's `/error` page goes through the security chain too: permit it (`.dispatcherTypeMatchers(DispatcherType.ERROR).permitAll()`), otherwise an anonymous client gets a `401` instead of the real error (see [Spring Web MVC](SPRING_WEBMVC.md)).

## CSRF and CORS

**CSRF** (Cross-Site Request Forgery): a malicious site makes the user's browser send a request to the application, with the session cookie sent automatically. Spring Security requires a token on state-changing requests (`POST`, `PUT`, `PATCH`, `DELETE`), which the other site can't read.

- Enabled by default. Thymeleaf forms include the token automatically.
- **Session cookie + SPA (Angular)**: keep CSRF; the token goes in an `XSRF-TOKEN` cookie that Angular's `HttpClient` sends back in the `X-XSRF-TOKEN` header. Spring Security 7: `http.csrf(csrf -> csrf.spa())`.
- **Stateless API with a token** in the `Authorization` header (JWT): the browser doesn't send it automatically, so CSRF isn't possible: `csrf(csrf -> csrf.disable())` is fine. Not with a token stored in a cookie.

**CORS** (Cross-Origin Resource Sharing): lets a front-end served from another origin (`http://localhost:4200`) call the API. The browser sends a preflight `OPTIONS` request, which must be answered before authentication:

```java
http.cors(Customizer.withDefaults()); // uses the CorsConfigurationSource bean

@Bean
CorsConfigurationSource corsConfigurationSource() {
  CorsConfiguration config = new CorsConfiguration();
  config.setAllowedOrigins(List.of("http://localhost:4200"));
  config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
  config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
  config.setAllowCredentials(true);
  UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/api/**", config);
  return source;
}
```

Configured only in Spring MVC (`@CrossOrigin`, `WebMvcConfigurer`), CORS comes too late: the preflight is rejected by the security chain first.

## Stateless REST API with JWT

An API called by an Angular front-end or other services usually receives a JWT issued by an authorization server (Keycloak, Auth0, Spring Authorization Server): the OAuth2 **resource server** support validates it on each request, without session.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/shop # public keys fetched from the issuer
```

```java
@Bean
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
  http
      .authorizeHttpRequests(auth -> auth
          .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
          .requestMatchers("/api/orders/**").hasAuthority("SCOPE_orders")
          .anyRequest().authenticated())
      .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
      .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .csrf(csrf -> csrf.disable());
  return http.build();
}
```

- `BearerTokenAuthenticationFilter` reads `Authorization: Bearer <jwt>`; the `JwtDecoder` checks the signature (public key of the issuer), the expiry (`exp`) and the issuer (`iss`).
- The `scope` claim becomes `SCOPE_...` authorities by default; roles in another claim (Keycloak's `realm_access.roles`) need a `JwtAuthenticationConverter`.
- In a controller: `@AuthenticationPrincipal Jwt jwt` gives access to the claims (`jwt.getSubject()`).
- `STATELESS`: no HTTP session, the context is rebuilt from the token on each request. Logout = token expiry (short-lived access tokens, refresh tokens handled by the client).

## Spring Security 7 (Spring Boot 4)

- Removed: `and()` in `HttpSecurity` (lambda DSL only), `authorizeRequests()` (use `authorizeHttpRequests()`), `AntPathRequestMatcher` / `MvcRequestMatcher` (replaced by `PathPatternRequestMatcher`, the same path syntax as Spring MVC), the OAuth2 password grant.
- New: initial multi-factor authentication support, `csrf.spa()` for SPAs, Password4j-based password encoders, OAuth2 support for HTTP service clients (`@ClientRegistrationId`).
- Spring Authorization Server (to issue OAuth2 / OIDC tokens) is now part of Spring Security, with PKCE enabled by default.
