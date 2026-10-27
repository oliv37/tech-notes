## What Spring Security is

> **In short**: Spring Security handles authentication (who are you? → `401`) and authorization (are you allowed? → `403`) through a chain of servlet filters, plus protections against common attacks (CSRF, security headers). With Spring Boot, everything is secured by default; a `SecurityFilterChain` bean replaces the default rules.

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

## The security filter chain

> **In short**: Spring Security is a set of servlet filters run before the `DispatcherServlet`. `DelegatingFilterProxy` bridges the container and Spring, `FilterChainProxy` picks the matching `SecurityFilterChain`, whose filters authenticate the request, then `AuthorizationFilter` checks the rules; `ExceptionTranslationFilter` turns failures into `401` / `403`.

```text
Request → DelegatingFilterProxy → FilterChainProxy → SecurityFilterChain (filters 1..n) → DispatcherServlet → controller
```

- **`DelegatingFilterProxy`**: a servlet filter registered in the container (see [Servlet](../java/SERVLET.md)), delegating to the Spring bean `springSecurityFilterChain`.
- **`FilterChainProxy`**: that bean; picks the first `SecurityFilterChain` matching the request.
- **`SecurityFilterChain`**: an ordered list of filters, built with `HttpSecurity`; each feature adds its filter (`httpBasic()` → `BasicAuthenticationFilter`).

Main filters, in order:

| Filter | Role |
|---|---|
| `SecurityContextHolderFilter` | loads the `SecurityContext` (from the session) for the request, clears it at the end |
| `HeaderWriterFilter` | security headers: `X-Content-Type-Options`, `X-Frame-Options`, HSTS... |
| `CorsFilter` | answers CORS preflight requests |
| `CsrfFilter` | checks the CSRF token on `POST`, `PUT`, `DELETE`... |
| `LogoutFilter` | `POST /logout` |
| `UsernamePasswordAuthenticationFilter` | form login: `POST /login` |
| `BearerTokenAuthenticationFilter` / `BasicAuthenticationFilter` | `Authorization: Bearer <jwt>` / `Basic base64(user:password)` |
| `AnonymousAuthenticationFilter` | nobody authenticated: sets an anonymous `Authentication` |
| `ExceptionTranslationFilter` | security exceptions → `401` / `403` (see below) |
| `AuthorizationFilter` | last: checks the `authorizeHttpRequests` rules |

- An authentication filter only acts when its credentials are present: without `Authorization` header, the request reaches `AuthorizationFilter` as anonymous.
- Debugging: `logging.level.org.springframework.security=TRACE` logs each filter invocation, to find where a `401` / `403` comes from.

## SecurityContext and Authentication

> **In short**: The authenticated user is an `Authentication` (principal, authorities) stored in the `SecurityContext`, held by the `SecurityContextHolder` in a `ThreadLocal` for the duration of the request. It's kept in the HTTP session for a login, or rebuilt from the token on each request for a stateless API.

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

**`Principal`, `Authentication` and `UserDetails`**: the word "principal" is used at two levels.

```text
java.security.Principal          (JDK)              getName()
  └─ Authentication              (Spring Security)  + getAuthorities(), getCredentials(), getDetails(),
                                                      getPrincipal(), isAuthenticated()
       └─ getPrincipal() → Object: the user itself
            ├─ UserDetails       (login/password)   getUsername(), getPassword(), getAuthorities(), isEnabled()...
            ├─ Jwt               (resource server)
            └─ OAuth2User / OidcUser (oauth2Login)
```

- **`Principal`** (JDK, also returned by the servlet API's `request.getUserPrincipal()`): just a name.
- **`Authentication`** extends `Principal`: it's the authentication result, the object checked by authorization. `getName()` delegates to the user object (`UserDetails.getUsername()`, the `sub` claim of a `Jwt`).
- **`getPrincipal()`** returns the user object, whose type depends on the `Authentication` implementation (`UsernamePasswordAuthenticationToken` → `UserDetails`, `JwtAuthenticationToken` → `Jwt`, `OAuth2AuthenticationToken` → `OAuth2User`).
- The same token type is used before and after authentication, with different content: the login filter creates `UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret")` (principal = the username `String`, not authenticated), and the `AuthenticationManager` returns a new, authenticated token (principal = the `UserDetails`, credentials erased, authorities filled). Only the latter is stored in the `SecurityContext`.
- **`UserDetails`** doesn't extend `Principal`: it's the user as loaded by the `UserDetailsService` (see below). After authentication, the provider copies its authorities into the `Authentication`, and those are the ones checked.
- In a controller: a `Principal` or `Authentication` parameter gives the `Authentication`; `@AuthenticationPrincipal` gives `getPrincipal()`, already cast (`UserDetails`, `Jwt`).

## Configuring a SecurityFilterChain

> **In short**: The security rules are a `SecurityFilterChain` bean configured with `HttpSecurity` and the lambda DSL: `authorizeHttpRequests` rules checked in order (first match wins), then the authentication methods (form login, HTTP Basic, JWT). Several chains can coexist, each limited to some URLs with `securityMatcher`.

`WebSecurityConfigurerAdapter` is removed since Spring Security 6.

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

> **In short**: An authentication filter passes the credentials to the `AuthenticationManager` (`ProviderManager`), which delegates to an `AuthenticationProvider`. For login/password, the `DaoAuthenticationProvider` loads the user with the application's `UserDetailsService` and checks the password hash with the `PasswordEncoder` (BCrypt).

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

> **In short**: `@EnableMethodSecurity` + `@PreAuthorize("hasRole('ADMIN')")` checks rules on methods, with SpEL access to the current user and the arguments. Proxy-based like `@Transactional`; it protects the business logic whatever the entry point, where URL rules only protect HTTP endpoints.

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

> **In short**: `401`: not authenticated, handled by the `AuthenticationEntryPoint` (401 or redirect to the login page). `403`: authenticated but not allowed, handled by the `AccessDeniedHandler`. `ExceptionTranslationFilter` makes the translation; these errors happen in the filters, so a `@RestControllerAdvice` doesn't see them.

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

> **In short**: CSRF: another site makes the browser send a request with the session cookie; Spring requires a token on state-changing requests. Keep it with a session cookie, disable it for a stateless API using an `Authorization` header. CORS: allows a front-end on another origin to call the API; configure it in Spring Security, since the preflight must be answered before authentication.

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

> **In short**: As an OAuth2 resource server, the API validates the JWT sent in `Authorization: Bearer` on each request (signature with the issuer's public key, expiry, issuer), without session. Scopes become `SCOPE_...` authorities; `issuer-uri` is enough to configure it.

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

## Same app as OAuth client and API: session or JWT cookie

> **In short**: When one Spring app serves both the Angular front-end's login and the API, it can keep the tokens out of the browser. Either `oauth2Login()` with a session (the app is a BFF, the browser only gets `JSESSIONID`; Spring Session + Redis for several instances), or a JWT in an `HttpOnly` cookie read by a custom `BearerTokenResolver` (stateless). In both cases authentication goes through a cookie, so CSRF stays enabled.

**Session (BFF merged with the API)**: the app is the OAuth client, it performs the login with the authorization server and keeps the tokens in the HTTP session.

```java
@Bean
SecurityFilterChain browserChain(HttpSecurity http) throws Exception {
  http
      .oauth2Login(Customizer.withDefaults()) // login flow, tokens kept in the session
      .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
      .csrf(csrf -> csrf.spa()); // Spring Security 7: XSRF-TOKEN cookie for Angular
  return http.build();
}
```

- The API endpoints are protected by the session: no `oauth2ResourceServer()` needed for calls from Angular.
- If other services also call the API with a `Bearer` token, add a second `SecurityFilterChain` (with `securityMatcher` and `@Order`) configured as a resource server.
- Several instances: Spring Session with Redis shares the sessions, no sticky sessions needed.

**JWT in an `HttpOnly` cookie (stateless)**: the app sets the access token in a cookie and validates it as a resource server. `DefaultBearerTokenResolver` only reads the `Authorization` header (and optionally a request parameter), so the cookie needs a custom resolver:

```java
http.oauth2ResourceServer(rs -> rs
    .bearerTokenResolver(request -> {
      Cookie cookie = WebUtils.getCookie(request, "access_token");
      return cookie != null ? cookie.getValue() : null;
    })
    .jwt(Customizer.withDefaults()));
```

- No shared store: any instance validates the JWT with the issuer's public key.
- The trade-offs (refresh token in a browser cookie, revocation, cookie size) are compared with the BFF in [BFF vs access token in an HttpOnly cookie](../../front-end/angular/ANGULAR.md#bff-vs-access-token-in-an-httponly-cookie).
- Unlike the stateless API above, `csrf.disable()` is a vulnerability here: the browser sends the cookie automatically.

## Spring Security 7 (Spring Boot 4)

> **In short**: Lambda DSL only (`and()` and `authorizeRequests()` removed), `PathPatternRequestMatcher` replaces the Ant matchers, initial MFA support, `csrf.spa()`, and Spring Authorization Server is now part of Spring Security.

- Removed: `and()` in `HttpSecurity` (lambda DSL only), `authorizeRequests()` (use `authorizeHttpRequests()`), `AntPathRequestMatcher` / `MvcRequestMatcher` (replaced by `PathPatternRequestMatcher`, the same path syntax as Spring MVC), the OAuth2 password grant.
- New: initial multi-factor authentication support, `csrf.spa()` for SPAs, Password4j-based password encoders, OAuth2 support for HTTP service clients (`@ClientRegistrationId`).
- Spring Authorization Server (to issue OAuth2 / OIDC tokens) is now part of Spring Security, with PKCE enabled by default.
