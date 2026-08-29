---
share_link: https://share.note.sx/278kiase#pYWPQJ15e6bDB22hsdO9GQ
share_updated: 2026-08-02T03:04:45+03:00
---
## Fundamentals

**1. What is Spring Security?** A framework that handles authentication and authorization for Spring-based applications. It provides built-in protection against common web vulnerabilities (CSRF, session fixation, clickjacking) and supports many authentication mechanisms — form login, HTTP Basic, JWT, OAuth2, LDAP, and more.

**2. What's the difference between authentication and authorization?**

- **Authentication** — verifying _who_ someone is (e.g., checking a username/password or a token).
- **Authorization** — determining _what_ an authenticated user is allowed to do/access (e.g., only admins can delete users). Authentication always happens first; authorization decisions are made based on the authenticated identity.

**3. How do you add Spring Security to a Spring Boot project?** Add the `spring-boot-starter-security` dependency. As soon as it's on the classpath, Spring Boot auto-secures **all endpoints by default** (requiring login), and generates a default user with a random password logged to the console at startup — until you provide your own configuration.

**4. What is a `SecurityFilterChain`, and why is it important?** In modern Spring Security (5.7+/6.x), it's the recommended way to configure security rules — a `@Bean` of type `SecurityFilterChain` built using an `HttpSecurity` object, replacing the older, now-deprecated `WebSecurityConfigurerAdapter` class-extension approach. It defines which URLs require authentication, which roles/authorities are needed, and how login/logout/CSRF/session behavior works.

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/user/**").hasRole("USER")
            .requestMatchers("/", "/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

**5. Why was `WebSecurityConfigurerAdapter` deprecated, and what replaced it?** It was deprecated in favor of a **component-based configuration style** — instead of extending a base class and overriding `configure()` methods, you now define `@Bean`s (like `SecurityFilterChain`, `UserDetailsService`, `PasswordEncoder`) directly. This aligns better with Spring's general move toward composition over inheritance and makes it easier to define multiple, independent filter chains.

**6. What is `requestMatchers()` used for?** Used inside `authorizeHttpRequests()` to define URL patterns and the access rules that apply to them — e.g., `hasRole()`, `hasAuthority()`, `permitAll()`, `denyAll()`, or `authenticated()`. It replaced the older `antMatchers()`/`mvcMatchers()` methods.

---

## Password Handling

**7. What is `PasswordEncoder`, and why is it necessary?** An interface responsible for hashing passwords before storing them and verifying a raw password against a stored hash during login. Passwords should **never** be stored in plain text — `PasswordEncoder` implementations like `BCryptPasswordEncoder` use one-way hashing algorithms (with salt) so even if the database is compromised, actual passwords aren't exposed.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

**8. Why is BCrypt commonly preferred over something like MD5 or SHA-256 for passwords?** BCrypt is intentionally slow and includes a built-in salt, making brute-force and rainbow-table attacks much harder. Generic hash functions like MD5/SHA-256 are designed to be _fast_, which is actually a weakness for password storage since it makes brute-forcing easier.

---

## Core Authentication Components

**9. What is `UserDetailsService`?** An interface with a single method, `loadUserByUsername(String username)`, that Spring Security calls to fetch user data (username, password hash, authorities/roles) during authentication — typically implemented to pull user data from a database.

**10. What is `UserDetails`?** An interface representing the core user information Spring Security needs: username, password, authorities (roles/permissions), and account status flags (enabled, locked, expired, credentials expired).

**11. What is `SecurityContextHolder`?** A static holder that stores the `SecurityContext` for the current thread, which in turn holds the `Authentication` object representing the currently logged-in user. You can access the current user anywhere in the app via:

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

**12. What is the `Authentication` object, and what does it contain?** Represents the authenticated (or attempting-to-authenticate) user. Key fields:

- **principal** — the identity of the user (often a `UserDetails` object)
- **credentials** — usually the password (often cleared after authentication for security)
- **authorities** — the granted roles/permissions

**13. What is an `AuthenticationManager`?** The core interface responsible for processing an authentication request — it takes an `Authentication` object (with unverified credentials) and returns a fully authenticated `Authentication` object (or throws an exception if authentication fails). It typically delegates to one or more `AuthenticationProvider`s.

**14. What is an `AuthenticationProvider`?** Performs the actual authentication logic for a specific mechanism (e.g., `DaoAuthenticationProvider` checks username/password against a `UserDetailsService` + `PasswordEncoder`). An `AuthenticationManager` can be configured with multiple providers to support different authentication types.

---

## The Filter Chain

**15. How does Spring Security process incoming requests?** Every request passes through a chain of **servlet filters** before reaching your controller. Each filter has a specific responsibility — e.g., checking for a session, validating a JWT, handling login form submission, enforcing access rules — and can either pass the request along or reject it (e.g., redirect to login, return 401/403).

**16. What is `DelegatingFilterProxy`?** A standard servlet `Filter` that bridges the servlet container's filter lifecycle with Spring's `ApplicationContext` — it delegates actual filtering logic to a Spring-managed bean, letting Spring Security filters participate in dependency injection.

**17. What is `FilterChainProxy`?** The Spring-managed bean that `DelegatingFilterProxy` delegates to — it holds and manages the actual chain of Spring Security filters (defined by `SecurityFilterChain`), applying them in order to each request.

**18. Name a few important built-in security filters and their roles.**

- `UsernamePasswordAuthenticationFilter` — handles form-based login submissions
- `BasicAuthenticationFilter` — handles HTTP Basic auth headers
- `BearerTokenAuthenticationFilter` — handles JWT/OAuth2 bearer tokens
- `SessionManagementFilter` — manages session-related concerns (e.g., session fixation protection)
- `ExceptionTranslationFilter` — catches security exceptions and converts them into appropriate HTTP responses (401/403 or redirects)
- `FilterSecurityInterceptor` / `AuthorizationFilter` — makes the final access-control decision for the request

**19. Why does the _order_ of filters in the chain matter?** Because each filter depends on state set up by earlier ones — e.g., you can't make an authorization decision (`AuthorizationFilter`) before the user has been authenticated (`UsernamePasswordAuthenticationFilter`/`BearerTokenAuthenticationFilter`). Spring Security manages a well-defined default order, but understanding it matters when adding **custom filters** at the right position (`addFilterBefore`/`addFilterAfter`).

---

## Authorization

**20. What's the difference between a role and an authority in Spring Security?**

- **Authority** — a fine-grained permission string (e.g., `"READ_PRIVILEGES"`, `"users:write"`).
- **Role** — a broader grouping, conventionally stored as an authority prefixed with `ROLE_` (e.g., `ROLE_ADMIN`). Methods like `hasRole("ADMIN")` automatically add the `ROLE_` prefix under the hood, while `hasAuthority("ROLE_ADMIN")` requires the exact string.

**21. What is method-level security, and how do you enable it?** Enabling security checks directly on service/controller methods (rather than only at the URL level) using annotations, enabled via `@EnableMethodSecurity` (modern) or the older `@EnableGlobalMethodSecurity`.

**22. What do `@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, and `@PostFilter` do?**

- `@PreAuthorize` — checks an expression **before** the method executes; if false, the method never runs
- `@PostAuthorize` — checks an expression **after** the method executes, can inspect the return value; if false, access is denied even though the method ran
- `@PreFilter` — filters a collection **argument** before the method executes, removing elements that don't match a condition
- `@PostFilter` — filters a collection **return value**, removing elements the current user shouldn't see

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) { ... }
```

These use **SpEL (Spring Expression Language)** to write flexible access rules.

**23. What is `@Secured`, and how is it different from `@PreAuthorize`?** An older, simpler annotation that only checks role names (e.g., `@Secured("ROLE_ADMIN")`) — no SpEL support, so it can't express complex conditions like `@PreAuthorize` can.

---

## Web Security Concerns

**24. What is CSRF, and how does Spring Security protect against it?** **Cross-Site Request Forgery** — an attack where a malicious site tricks a logged-in user's browser into submitting an unwanted request to your app (since browsers auto-attach session cookies). Spring Security protects against this by requiring a unique, unpredictable **CSRF token** to be included in state-changing requests (POST/PUT/DELETE); without a valid token, the request is rejected. CSRF protection is typically **disabled for stateless APIs** using token-based auth (like JWT), since there's no session cookie to exploit.

**25. What is CORS, and how do you configure it in Spring Security?** **Cross-Origin Resource Sharing** — a browser security mechanism that blocks JavaScript from making requests to a different origin (domain/port/protocol) unless the server explicitly allows it via response headers. In Spring Security, you configure a `CorsConfigurationSource` bean and enable it in the filter chain with `.cors(Customizer.withDefaults())`.

**26. What's the difference between CSRF and CORS? (Common source of confusion)**

- **CORS** controls **which origins are allowed** to make requests to your API from a browser — it's about relaxing browser restrictions safely.
- **CSRF** is an **attack** that Spring Security defends against by requiring a token — it's about preventing unwanted requests from being executed using the victim's existing session. They solve different problems and are configured independently.

**27. What is session fixation, and how does Spring Security prevent it?** An attack where an attacker sets a known session ID on a victim before they log in, then hijacks that session after authentication. Spring Security prevents this by **creating a new session ID** upon successful login by default (session fixation protection).

**28. What's the difference between stateful and stateless authentication?**

- **Stateful**: the server maintains a session (e.g., via a session ID cookie) that tracks the authenticated user across requests — traditional form-login web apps.
- **Stateless**: no server-side session; each request carries its own proof of identity (e.g., a JWT in the `Authorization` header) — common for REST APIs, especially in microservices, since it removes the need for shared session storage.

---

## JWT & OAuth2

**29. What is JWT, and why is it commonly used with Spring Security for REST APIs?** **JSON Web Token** — a compact, self-contained, digitally signed token containing claims (e.g., user ID, roles, expiration) that can be verified without a database lookup. It's popular for stateless REST APIs because the server doesn't need to store session state — it just validates the token's signature and expiration on each request.

**30. What are the three parts of a JWT?** `header.payload.signature` — the header describes the signing algorithm, the payload contains the claims (data), and the signature verifies the token hasn't been tampered with.

**31. How would you implement JWT authentication in a Spring Boot app at a high level?**

1. User logs in with credentials via a login endpoint
2. Server validates credentials, generates a signed JWT, and returns it
3. Client stores the token and sends it in the `Authorization: Bearer <token>` header on subsequent requests
4. A custom filter (added to the `SecurityFilterChain`, typically before `UsernamePasswordAuthenticationFilter`) extracts and validates the token, then sets the `Authentication` object in `SecurityContextHolder` if valid
5. Since it's stateless, CSRF protection and HTTP sessions are typically disabled

**32. What is OAuth2, and how is it different from JWT?** OAuth2 is an **authorization framework/protocol** that defines how a user can grant a third-party application limited access to their resources without sharing credentials (e.g., "Sign in with Google"). JWT is just a **token format** — OAuth2 flows often _use_ JWTs as the access token format, but they're solving different problems (OAuth2 = delegation protocol, JWT = a way to encode token data).

**33. What is the difference between an access token and a refresh token?**

- **Access token** — short-lived, used to authenticate API requests.
- **Refresh token** — longer-lived, used to obtain a new access token once the old one expires, without requiring the user to log in again.

---

## Practical / Scenario Questions

**34. How would you secure a REST API so only users with role `ADMIN` can access `/api/admin/**`?**

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
);
```

**35. How would you allow public access to `/api/auth/login` and `/api/auth/register` while securing everything else?**

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/login", "/api/auth/register").permitAll()
    .anyRequest().authenticated()
);
```

**36. If a user gets a 403 Forbidden instead of 401 Unauthorized, what does that typically mean?**

- **401 Unauthorized** — the user isn't authenticated at all (missing/invalid credentials or token).
- **403 Forbidden** — the user _is_ authenticated, but doesn't have permission for the requested resource (e.g., a regular user trying to access an admin endpoint).

**37. How would you test a secured Controller endpoint with Spring's testing tools?** Use `MockMvc` with `@WithMockUser` to simulate an authenticated user with specific roles, without needing a real login flow:

```java
@Test
@WithMockUser(roles = "ADMIN")
void shouldAllowAdminAccess() throws Exception {
    mockMvc.perform(get("/api/admin/data"))
        .andExpect(status().isOk());
}

@Test
@WithMockUser(roles = "USER")
void shouldDenyNonAdminAccess() throws Exception {
    mockMvc.perform(get("/api/admin/data"))
        .andExpect(status().isForbidden());
}
```

**38. What would you do if you needed multiple, independent security configurations (e.g., different rules for `/api/**` vs `/admin/**`)?** Define multiple `SecurityFilterChain` beans, each with a `securityMatcher()` (or `@Order`) specifying which requests it applies to — Spring Security picks the first chain whose matcher matches the incoming request.

---
