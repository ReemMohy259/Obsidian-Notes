---
share_link: https://share.note.sx/1epeths1#T+NgXlLJNtaMZzrBPSc/nA
share_updated: 2026-08-02T03:05:04+03:00
---
## Table of Contents

- [[#1. Authentication vs Authorization]]
- [[#2. Terminology — Principal, Authority, Authorization Models]]
- [[#3. Stateful vs Stateless Applications]]
- [[#4. The Spring Security Filter Chain Architecture]]
- [[#5. Default Security Filters & Headers]]
- [[#6. Password Storage & Encoding]]
- [[#7. Authentication Architecture — Core Contracts]]
- [[#8. UserDetailsService & Other Authentication Types]]
- [[#9. Basic Authentication — End-to-End Flow]]
- [[#10. Authentication Persistence & Session Management]]
- [[#11. Authentication Events]]
- [[#12. Logout]]
- [[#13. Authorization Architecture]]
- [[#14. Authorizing HttpServletRequest]]
- [[#15. Method Security]]
- [[#16. Authorization Events]]
- [[#17. Cryptography Fundamentals]]
- [[#18. JOSE Framework & JWT Structure]]
- [[#19. OAuth 2.0 & OIDC]]
- [[#20. OAuth 2.0 Grants / OIDC Flows]]
- [[#21. PKCE]]
- [[#22. Tokens — ID Token, Access Token, Refresh Token]]
- [[#23. Backend For Frontend (BFF) Pattern]]
- [[#24. Keycloak Terminology & Sessions]]
- [[#25. Spring Security's OAuth 2.0 Support]]
- [[#26. Common Web Attacks & Protections]]
- [[#27. Quick Revision Cheat Sheet]]

---

## 1. Authentication vs Authorization

||Authentication|Authorization|
|---|---|---|
|**Question answered**|"Who are you?"|"What are you allowed to do?"|
|**Happens**|First|After authentication succeeds|
|**Typical mechanism**|Username/password, JWT, OAuth2 token, certificates|Roles, permissions, `@PreAuthorize`, ACLs|
|**Failure status**|`401 Unauthorized`|`403 Forbidden`|
|**Spring interface**|`AuthenticationManager`, `AuthenticationProvider`|`AccessDecisionManager` / `AuthorizationManager` (Spring Security 6+)|

```
Request
   │
   ▼
Authentication  → "Is this really Alice?"      → 401 if it fails
   │
   ▼
Authorization   → "Can Alice DELETE /orders/5?" → 403 if it fails
   │
   ▼
Controller executes
```

> [!tip] `401` = not authenticated at all. `403` = authenticated, but not permitted. Getting this pair right in your API responses is a very common interview check.

---

## 2. Terminology — Principal, Authority, Authorization Models

|Term|Definition|
|---|---|
|**Principal**|An entity (typically a user) that can be authenticated by a system or network|
|**Authority**|A permission granted to a user to perform some action — can be a fine-grained **privilege** (`repo:create`, `pr:write`) or a coarse **role** (`ROLE_ADMIN`, `ROLE_USER`)|
|**Authentication**|_"Are you who you say you are?"_ — verified via something like a password or fingerprint|
|**Authorization**|_"Do you have the authority to perform this action?"_|

### Authorization Models

|Model|Basis|Example|
|---|---|---|
|**RBAC** (Role-Based Access Control)|Authorities/privileges (`CAN_CREATE_PR`, `pr:write`) or roles (`ADMIN`)|Classic "does this role allow this action"|
|**ABAC** (Attribute-Based Access Control)|Contextual attributes|e.g. only allow access during working hours|
|**ReBAC** (Relationship-Based Access Control)|The relationship between the user and the resource|e.g. "did this user create this PR? If so, they can edit it"|

---

## 3. Stateful vs Stateless Applications

### Server-Side Rendered (Stateful — `HttpSession`)

```
Browser                          Server
  │──① POST /login (user/pass)────►│
  │                                │─② saves user in HttpSession
  │◄──③ JSESSIONID cookie──────────│
  │──④ GET /protected-resource────►│
  │                                │─⑤ renders HTML
  │◄──⑥ fully rendered page────────│
```

- Needs **sticky sessions** (the load balancer always routes a client back to the same server instance) or a **distributed session store** (e.g. Redis via Spring Session) to work correctly across multiple server instances.

```
Request → Load Balancer (sticky sessions) → Server A or Server B → shared Session Storage
```

### SPA + REST API (Stateless — JWT)

```
Browser                          Server
  │──① POST /login (user/pass)────►│
  │◄──② JWT────────────────────────│
  │─③ stores JWT
  │──④ GET /protected-resource────►│
  │     Authorization: Bearer <JWT>
  │◄──⑤ JSON payload───────────────│
  │─⑥ renders JSON payload
```

- **Stateless**: no server-side session at all — everything needed to authenticate travels with the token.
- **Scales without a distributed session store**: any server instance behind the load balancer can validate the token independently, since nothing is "remembered" server-side per client. Setting `SessionCreationPolicy.STATELESS` tells Spring Security to never create or read an `HttpSession` — every request must be fully self-authenticating. This is what makes horizontal scaling trivial: contrast with the classic session-replication problem stateful apps face.

```
Request (carries JWT) → Load Balancer → ANY server instance (no stickiness needed)
```

---

## 4. The Spring Security Filter Chain Architecture

Spring Security is implemented almost entirely as a chain of **Servlet Filters**, registered **before** `DispatcherServlet` ever sees the request.

```
HTTP Request
      │
      ▼
┌─────────────────────────────────────────┐
│         Spring Security Filter Chain      │
│                                            │
│  SecurityContextHolderFilter               │  ← restores SecurityContext (session-based)
│  CsrfFilter                                │  ← CSRF token validation
│  UsernamePasswordAuthenticationFilter      │  ← form login
│  BasicAuthenticationFilter                 │  ← HTTP Basic
│  BearerTokenAuthenticationFilter           │  ← JWT/OAuth2 bearer tokens
│  ExceptionTranslationFilter                │  ← converts security exceptions to 401/403
│  AuthorizationFilter                       │  ← final authorization decision
└─────────────────────────────────────────┘
      │
      ▼
DispatcherServlet → Controller
```

Each filter has one job; a request passes through all of them in order before reaching your controller.

### The Basic Servlet Filter Chain

```
Client ⇄ FilterChain: Filter₀ ⇄ Filter₁ ⇄ Filter₂ ⇄ Servlet
```

Plain servlet filters process requests before they reach a servlet — Spring Security's registered filter is itself just one entry in this chain, but that one entry is where all the magic happens.

### `DelegatingFilterProxy` — Bridging Spring Beans Into the Servlet Chain

- A plain `Filter` (implements the `Filter` interface) that **binds the lifecycle of a Spring-managed `Filter` bean** in the `ApplicationContext` to the lifecycle of the servlet container.
- The target bean is fetched **lazily**, only when the filter is actually invoked.
- Implements the **Virtual Proxy** design pattern: instead of a heavy/complex object being wired directly, a lightweight skeleton stands in for it and loads the real object **on demand**.
- This is how Spring Security integrates with the plain servlet spec while still being fully Spring-bean-configurable — registered as a **single `DelegatingFilterProxy`** in the actual servlet filter chain.

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    Filter delegate = getFilterBean(someBeanName);
    delegate.doFilter(request, response);
}
```

Breaking down the name:

|Part|Meaning|
|---|---|
|**Delegating**|Delegates to a Spring bean implementing `jakarta.servlet.Filter`|
|**Filter**|It itself implements the `Filter` interface|
|**Proxy**|Virtual Proxy pattern implementation|

### `FilterChainProxy` — Spring Security's Actual Filter

Spring Security's registered `Filter` **is** a `FilterChainProxy` — internally, it's a **whole `FilterChain`** packed inside a single `Filter`.

```java
public interface SecurityFilterChain {
    boolean matches(HttpServletRequest request);
    List<Filter> getFilters();
}

public class FilterChainProxy extends GenericFilterBean {
    private List<SecurityFilterChain> filterChains;
}
```

```
Client ⇄ Filter₀ ⇄ [DelegatingFilterProxy → FilterChainProxy] ⇄ Filter₂ ⇄ Servlet
                              │
                     picks ONE matching SecurityFilterChain
                              │
                  ┌───────────┴──────────┐
          SecurityFilterChain₀       SecurityFilterChain n
             matches "/api/**"         matches "/**"
```

You can register **multiple** `SecurityFilterChain`s, each guarded by a `matches()` predicate — `FilterChainProxy` picks the first one whose predicate matches the incoming request and runs only _that_ chain's filters.

### Why `FilterChainProxy` Exists (It's Not Optional)

- It's the actual **start** of Spring Security's servlet support.
- It decides which `SecurityFilterChain` to invoke based on the **full `HttpServletRequest`** (path, headers, method, etc.) — not just the URI, unlike a plain servlet container's simpler path-based dispatch.

### Debugging: Seeing Which Filters Run

At application startup (INFO level), Spring logs the full filter list for the default chain. 

![[Pasted image 20260802020130.png]]

Setting `logging.level.org.springframework.security=TRACE` shows **every filter invoked per request**, in order, plus the actual decision made (e.g. `Invoking CsrfFilter (5/15)` → `Invalid CSRF token found` → `Responding with 403 status code`) — invaluable for diagnosing "why did my request get rejected."

### `ExceptionTranslationFilter` — Converting Security Exceptions Into HTTP Responses

Handles `AuthenticationException` and `AccessDeniedException` thrown by **anything after it** in the chain.

```java
try {
    filterChain.doFilter(request, response);
} catch (AccessDeniedException | AuthenticationException ex) {
    if (!authenticated || ex instanceof AuthenticationException) {
        startAuthentication();
    } else {
        accessDenied();
    }
}
```

```
                     ① filterChain.doFilter() — normal processing
SecurityFilterChain ────────────────► Continue Processing Normally
        │
        │ Security Exception thrown
        ▼
   ② Not authenticated / AuthenticationException?
        │                                    │
       Yes                                   No — AccessDeniedException
        ▼                                    ▼
  Start Authentication                 ③ Access Denied
  - SecurityContextHolder cleared       - AccessDeniedHandler invoked
  - Request saved (RequestCache)
  - AuthenticationEntryPoint requests credentials
```

1. First, `ExceptionTranslationFilter` invokes `FilterChain.doFilter(...)` to run the rest of the application.
2. If the user isn't authenticated (or it's an `AuthenticationException`) → **Start Authentication**: clear the `SecurityContextHolder`, save the `HttpServletRequest` so it can be **replayed** after successful login, and use the configured `AuthenticationEntryPoint` to request credentials (e.g. redirect to a login page, or send a `WWW-Authenticate` header).
3. Otherwise, if it's an `AccessDeniedException` → **Access Denied**: the `AccessDeniedHandler` handles the rejection.

### `RequestCache` — Replaying the Original Request After Login

- The `HttpServletRequest` is saved in a `RequestCache` before authentication starts; once login succeeds, the cache is used to replay the original request so the user lands where they meant to go.
- `RequestCacheAwareFilter` is what actually uses the `RequestCache` to save/replay the request.
- **Default**: `HttpSessionRequestCache` — checks the `HttpSession` for a saved request, gated by a `continue` parameter.
- **Stateless alternative**: `CookieRequestCache`, suitable for applications with no server-side session at all.

### Minimal Modern Configuration (Spring Security 6+, Lambda DSL)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.disable())              // e.g. for stateless JWT APIs
            .sessionManagement(sm -> sm
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

> [!warning] `WebSecurityConfigurerAdapter` is removed (Spring Security 5.7+/6.x). Older tutorials extend `WebSecurityConfigurerAdapter` — that class is deprecated/removed. The current approach is exposing a `SecurityFilterChain` **`@Bean`** as shown above, using the lambda-based DSL.

---

## 5. Default Security Filters & Headers

### The Default `SecurityFilterChain` (Spring Boot Auto-Configuration)

|Filter|Responsibility|
|---|---|
|`DisableEncodeUrlFilter`|Disables saving the HTTP session via URL rewriting|
|`WebAsyncManagerIntegrationFilter`|Manages saving the authentication object across async servlet APIs|
|`SecurityContextHolderFilter`|Initializes `SecurityContextHolder` with a persisted `SecurityContext` (e.g. from the HTTP session), if present|
|`HeaderWriterFilter`|Writes security-related HTTP headers (`X-Frame-Options`, etc. — see below)|
|`CorsFilter`|Handles CORS|
|`CsrfFilter`|Validates CSRF tokens|
|`LogoutFilter`|Calls `LogoutHandler`s when a user logs out|
|`UsernamePasswordAuthenticationFilter`|Form-based login (username + password)|
|`DefaultLoginPageGeneratingFilter`|Generates a default login page|
|`DefaultLogoutPageGeneratingFilter`|Generates a default logout page|
|`BasicAuthenticationFilter`|HTTP Basic auth (`Authorization: Basic username:password`)|
|`RequestCacheAwareFilter`|Retrieves the originally intended request after successful login (e.g. `/admin` → login → back to `/admin`)|
|`SecurityContextHolderAwareRequestFilter`|Wraps `HttpServletRequest` with authentication-related decorator methods|
|`AnonymousAuthenticationFilter`|Creates an anonymous `SecurityContext` if the user isn't authenticated|
|`ExceptionTranslationFilter`|Handles `AuthenticationException`/`AccessDeniedException`|
|`AuthorizationFilter`|Restricts access to URLs by role/authentication status, delegating to `AuthorizationManager`|

### Default Response Headers

```
# Prevent viewing sensitive data via the back button after logout
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0

# Don't sniff content type — prevents XSS
X-Content-Type-Options: nosniff

# Force HTTPS treatment, protects against MITM (only sent over HTTPS)
Strict-Transport-Security: max-age=31536000 ; includeSubDomains

# Prevent Clickjacking (see section 26)
X-Frame-Options: DENY

# Disabled — legacy browser XSS filter, deprecated because it introduced its own vulnerabilities
X-XSS-Protection: 0
```

### Customizing Headers

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.headers(headers -> headers
        .frameOptions(frameOptions -> frameOptions.sameOrigin())
    );
    return http.build();
}
```

Or add a custom header writer directly:

```java
http.headers(headers -> headers
    .addHeaderWriter(new XFrameOptionsHeaderWriter(XFrameOptionsMode.SAMEORIGIN))
);
```

---

## 6. Password Storage & Encoding

### The Evolution of Password Storage

|Approach|Problem|
|---|---|
|**Plaintext**|Catastrophic if the database leaks|
|**Hash** (e.g. SHA-256)|Vulnerable to **rainbow tables** — precomputed hash→password lookup tables|
|**Salted hash**|Random salt stored alongside the hash defeats rainbow tables — but modern hardware computes billions of hashes/second, making even salted **fast** cryptographic hashes crackable via brute force|
|**Salt + adaptive hash function**|Intentionally CPU **and** memory intensive (~1 second to verify one password) — makes brute-forcing impractical even at scale, via a tunable **"work factor"**|

> [!important] Why "intentionally slow" is the whole point. A password hash needs to be **expensive to compute** — the opposite of what you want from, say, a `HashMap` key hash. Adaptive functions (BCrypt, Argon2, PBKDF2, SCrypt) let you dial the cost up over time as hardware improves.

> [!danger] Never store plaintext passwords. Always hash passwords with a **slow, salted, adaptive** hashing algorithm — never plain SHA-256/MD5 (too fast, enables brute-forcing at scale) and never reversible encryption.

### `PasswordEncoder` Interface

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}
```

### BCrypt — The Standard Default

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(); // default strength = 10
}
```

- **Salted automatically** — the salt is generated per-password and stored **inside** the resulting hash string itself, so you never manage salts separately.
- **Adaptive/slow by design** — has a configurable "strength" (work factor); increasing it deliberately makes brute-forcing more expensive as hardware gets faster.
- Encoding the same password twice produces **different** hash strings (different salts) — that's expected; `matches()` is the correct (and only) way to verify a password, never string-equality on hashes.

### Other Options

|Encoder|Notes|
|---|---|
|**BCrypt**|Industry standard default in Spring Security|
|**Argon2**|Winner of the Password Hashing Competition, memory-hard (resists GPU cracking even better)|
|**PBKDF2**|NIST-approved, common in regulated/government contexts|
|**SCrypt**|Also memory-hard, less common in the Spring ecosystem than Argon2|

### `PasswordEncoder` and `DelegatingPasswordEncoder`

`DelegatingPasswordEncoder` is the **default** password encoder for good reason:

- Lets you **migrate** to newer, more secure hash algorithms as they become available, without breaking existing stored passwords.
- Lets you **support legacy** hashed passwords already in your database.
- Achieved by prefixing the stored value with an algorithm identifier: `{algorithmId}password` (e.g. `{bcrypt}$2a$10$...`) — the encoder reads the prefix to know which algorithm to use for verification.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    // prefixes hashes like {bcrypt}$2a$10$... so multiple algorithms can coexist/migrate over time
}
```

---

## 7. Authentication Architecture — Core Contracts

### `SecurityContextHolder` and `SecurityContext`

At the heart of Spring Security's authentication model:

```
SecurityContextHolder
  └── SecurityContext
        └── Authentication
              ├── Principal
              ├── Credentials
              └── Authorities
```

- `SecurityContextHolder` is **where** Spring Security stores who's currently authenticated.
- Spring Security **doesn't care how** it gets populated — if it holds a value, that value is treated as the current authenticated user. This is a deliberately loose contract, letting you populate it manually (e.g. for a custom auth filter).

```java
// Manually populating it
SecurityContext context = SecurityContextHolder.createEmptyContext();
Authentication authentication =
    new TestingAuthenticationToken("username", "password", "ROLE_USER");
context.setAuthentication(authentication);
SecurityContextHolder.setContext(context);

// Reading from it
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
String username = authentication.getName();
Object principal = authentication.getPrincipal();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```

`SecurityContextHolder` supports different storage **strategies** (configurable via `spring.security.strategy`):

|Strategy|Behavior|
|---|---|
|`MODE_THREADLOCAL` (default)|One context per thread|
|`MODE_INHERITABLETHREADLOCAL`|Child threads inherit the parent thread's context|
|`MODE_GLOBAL`|One shared context across the whole JVM|

### `Authentication` — Two Purposes

1. **Input** to `AuthenticationManager`, carrying the credentials a user has supplied (before verification — at this point `isAuthenticated()` is `false`).
2. **Output/representation** of the currently authenticated user, obtainable from the `SecurityContext` after successful authentication.

|Field|Meaning|
|---|---|
|`principal`|Identifies the user — often a `UserDetails` instance when authenticating via username/password|
|`credentials`|Often the password — typically **cleared** after authentication to avoid leaking it (e.g. via caching, or lingering in the session)|
|`authorities`|The `GrantedAuthority` instances — high-level permissions (roles, scopes)|

### `AuthenticationManager` and `ProviderManager`

`AuthenticationManager` is the API defining **how** Spring Security's filters actually perform authentication. The `Authentication` it returns is set on the `SecurityContextHolder` by the filter that invoked it.

**`ProviderManager`** is the most common implementation:

```
                 Authentication Providers
              ┌──────────────────────────────┐
ProviderManager──►│ AuthenticationProvider₀     │
              │ AuthenticationProvider₁ ...  │
              │ AuthenticationProviderₙ     │
              └──────────────────────────────┘
```

- Delegates to a **list** of `AuthenticationProvider`s.
- Each provider gets a chance to: succeed, fail, or **abstain** (let a downstream provider decide).
- If **none** can authenticate, throws `ProviderNotFoundException` (a special `AuthenticationException` meaning "no provider was configured to support this `Authentication` type").

**Parent `AuthenticationManager`** (optional):

```
              AuthenticationManager (Parent)
                          ▲
              ┌───────────┴───────────┐
        ProviderManager₁          ProviderManager₂
        (Providers A, B)          (Providers C, D)
```

Consulted only if **no** configured provider can authenticate. Common when multiple `SecurityFilterChain`s share some authentication mechanisms (the shared parent) but differ in others (the distinct child `ProviderManager`s).

> [!warning] Credential erasure and caching. By default, `ProviderManager` clears sensitive credentials (e.g. the password) from the returned `Authentication` object after a successful login, to avoid retaining them longer than necessary in the `HttpSession`. This can cause subtle issues if you **cache** user objects (e.g. for performance in a stateless app) and then expect credentials to still be present later.

### `AuthenticationProvider`

Each provider handles **one specific type** of authentication:

|Provider|Handles|
|---|---|
|`DaoAuthenticationProvider`|Username/password authentication via a `UserDetailsService`|
|`JwtAuthenticationProvider`|JWT-based authentication|

### `AuthenticationEntryPoint`

Used to send an HTTP response that **requests credentials** from an unauthenticated client trying to access a protected resource — a redirect to a login page, or a `WWW-Authenticate` header.

```java
public class BasicAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                          AuthenticationException authException) throws IOException {
        response.setHeader("WWW-Authenticate", "Basic realm=\"" + this.realmName + "\"");
        response.sendError(HttpStatus.UNAUTHORIZED.value(), HttpStatus.UNAUTHORIZED.getReasonPhrase());
    }
}
```

---

## 8. UserDetailsService & Other Authentication Types

The core abstraction for **loading user-specific data** during authentication.

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

### Custom Implementation

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getUsername())
                .password(user.getHashedPassword())
                .roles(user.getRoles().toArray(new String[0]))
                .build();
    }
}
```

### How It Fits the Authentication Flow

```
Login request (username/password)
        │
        ▼
AuthenticationManager
        │
        ▼
DaoAuthenticationProvider
        │
        ├── calls UserDetailsService.loadUserByUsername(username)
        │        → returns UserDetails (username, hashed password, authorities)
        │
        ├── calls PasswordEncoder.matches(rawPassword, storedHash)
        │
        ▼
Authentication succeeds/fails → SecurityContext populated
```

### `UserDetails` Interface

Represents the authenticated principal — key methods: `getUsername()`, `getPassword()`, `getAuthorities()`, plus account-status flags: `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()`, `isEnabled()`.

### Built-in `UserDetailsService` Implementations

|Implementation|Backing Store|
|---|---|
|`InMemoryUserDetailsManager`|In-memory, hardcoded users — dev/testing only|
|`JdbcUserDetailsManager`|Relational database via JDBC|
|Custom (e.g. JPA-backed)|Your own `UserRepository`-backed implementation|

### Other Authentication Mechanisms

|Type|Notes|
|---|---|
|**LDAP**|Central directory for user info **and** authentication, often used by organizations; can also store role info|
|**Form authentication**|For server-side rendered (SSR) applications|
|**Digest authentication**|⚠️ Not recommended — largely superseded by more secure mechanisms|

---

## 9. Basic Authentication — End-to-End Flow

```
Client                                            Server
  │──① GET / HTTP/1.1───────────────────────────►│
  │◄──401 Unauthorized─────────────────────────────│
  │   WWW-Authenticate: Basic realm="..."           │
  │                                                  │
[ask user for credentials]
  │──② GET / HTTP/1.1───────────────────────────►│
  │   Authorization: Basic <base64 user:pass>       │──③ check credentials
  │◄──200 OK  or  401 Unauthorized──────────────────│
```

> [!danger] Basic Auth sends credentials encoded, NOT encrypted. Base64 encoding is trivially reversible — Basic Authentication is **completely insecure** unless the entire exchange happens over **HTTPS/TLS**.

### Step by Step, Tying the Pieces Together

1. A user makes an unauthenticated request to a protected resource (`/private`).
2. `AuthorizationFilter` denies it, throwing `AccessDeniedException`.
3. Since the user isn't authenticated, `ExceptionTranslationFilter` initiates **Start Authentication** → the configured `BasicAuthenticationEntryPoint` sends the `WWW-Authenticate` header. (`RequestCache` here is typically a `NullRequestCache`, since the client already knows how to replay its own request.)
4. Client retries with credentials → `BasicAuthenticationFilter` → `AuthenticationManager` → `DaoAuthenticationProvider` → calls `UserDetailsService.loadUserByUsername()` and `PasswordEncoder.matches()`.
5. On success: `SecurityContextRepository#saveContext`, `SecurityContextHolder#setContext`, `AuthenticationEventPublisher#publishAuthenticationSuccess`, `RememberMeServices#loginSuccess`.
6. On failure: `SecurityContextRepository#clearContext`, `AuthenticationEntryPoint#commence` (asks again), `AuthenticationEventPublisher#publishAuthenticationFailure`, `RememberMeServices#loginFailure`.

```java
// BasicAuthenticationFilter.java
catch (AuthenticationException ex) {
    this.securityContextHolderStrategy.clearContext();
    this.authenticationEntryPoint.commence(request, response, ex);
    return;
}
```

---

## 10. Authentication Persistence & Session Management

### `SecurityContextHolderFilter` — the Persistence Mechanism

```java
public class SecurityContextHolderFilter extends GenericFilterBean {
    private final SecurityContextRepository securityContextRepository;

    private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
        Supplier<SecurityContext> deferredContext = securityContextRepository.loadDeferredContext(request);
        try {
            securityContextHolderStrategy.setDeferredContext(deferredContext);
            chain.doFilter(request, response);
        } finally {
            securityContextHolderStrategy.clearContext();
        }
    }
}
```

- Loads a **deferred** `SecurityContext` (lazily computed — only actually resolved if something in the chain asks for it) at the start of each request, and **always clears** the context at the end — ensuring no leakage between requests sharing a thread (thread pools reuse threads across unrelated requests).

### `SecurityContextRepository` Implementations

|Implementation|Behavior|
|---|---|
|`HttpSessionSecurityContextRepository`|Persists to the `HttpSession` — the classic stateful approach|
|`RequestAttributeSecurityContextRepository`|Persists to request attributes only — useful with `FORWARD`/`INCLUDE` dispatches|
|`DelegatingSecurityContextRepository`|Delegates to multiple repositories|
|`NullSecurityContextRepository`|Persists nothing — used for **stateless** applications|

### Configuring a Stateless Application

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) {
    return httpSecurity
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .build();
}
```

`SessionCreationPolicy.STATELESS` tells Spring Security to **never** create or read an `HttpSession` — every request must be fully self-authenticating.

---

## 11. Authentication Events

Spring Security can publish an application event for every authentication success/failure — useful for auditing, rate limiting, alerting.

```java
@Component
public class AuthenticationEvents {

    @EventListener
    public void onSuccess(AuthenticationSuccessEvent success) { }

    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent failure) { }
}
```

### Exception → Event Mapping

|Exception|Event|
|---|---|
|`BadCredentialsException`|`AuthenticationFailureBadCredentialsEvent`|
|`UsernameNotFoundException`|`AuthenticationFailureBadCredentialsEvent`|
|`AccountExpiredException`|`AuthenticationFailureExpiredEvent`|
|`ProviderNotFoundException`|`AuthenticationFailureProviderNotFoundEvent`|
|`DisabledException`|`AuthenticationFailureDisabledEvent`|
|`LockedException`|`AuthenticationFailureLockedEvent`|
|`AuthenticationServiceException`|`AuthenticationFailureServiceExceptionEvent`|
|`CredentialsExpiredException`|`AuthenticationFailureCredentialsExpiredEvent`|
|`InvalidBearerTokenException`|`AuthenticationFailureBadCredentialsEvent`|

Publishing setup:

```java
@Bean
public AuthenticationEventPublisher authenticationEventPublisher(ApplicationEventPublisher publisher) {
    AuthenticationEventPublisher eventPublisher = new DefaultAuthenticationEventPublisher(publisher);
    eventPublisher.setDefaultAuthenticationFailureEvent(GenericAuthenticationFailureEvent.class);
    return eventPublisher;
}
```

> [!info] This is Observer in production. This is `ApplicationEventPublisher`/`@EventListener` — Spring's built-in Observer pattern implementation — applied specifically to the authentication lifecycle.

---

## 12. Logout

- Full mechanics: `LogoutFilter` invokes registered `LogoutHandler`s (clearing the `SecurityContext`, invalidating the session, clearing cookies, etc.) then a `LogoutSuccessHandler`.
- **In stateless applications, there's no real concept of "logout"** — since nothing is stored server-side, there's no session to invalidate. "Logging out" of a JWT-based system really means: the client discards its token, and/or (for OAuth2/OIDC) explicit token revocation is triggered.

---

## 13. Authorization Architecture

### Roles vs Authorities

- An **authority** is a granular permission string (e.g. `ORDER_DELETE`).
- A **role** is conventionally an authority **prefixed with `ROLE_`** (e.g. `ROLE_ADMIN`) — Spring Security's `hasRole("ADMIN")` helper automatically adds that prefix for you.

### `GrantedAuthority`

```java
public interface GrantedAuthority {
    String getAuthority();
}
```

- Used by an `AuthorizationManager` to obtain a precise `String` representation of the authority.
- If an authority **can't** be precisely represented as a `String`, it's considered "complex," and `getAuthority()` must return `null` — signaling to the `AuthorizationManager` that it needs specific support for that authority type.
- **`SimpleGrantedAuthority`** is Spring Security's one concrete implementation, converting any user-specified `String` into a `GrantedAuthority`. Every built-in `AuthenticationProvider` uses it to populate the `Authentication` object.

### The `ROLE_` Prefix

By default, role-based authorization rules include `ROLE_` as a prefix — a rule requiring role `"USER"` actually looks for a `GrantedAuthority` returning `"ROLE_USER"`.

```java
@Bean
static GrantedAuthorityDefaults grantedAuthorityDefaults() {
    return new GrantedAuthorityDefaults("MYPREFIX_");   // customize the default prefix
}
```

> [!info] Must be `static`. Exposed via a `static` method so Spring publishes it **before** initializing Spring Security's method-security `@Configuration` classes.

### `AuthorizationManager` — Supersedes `AccessDecisionManager`/`AccessDecisionVoter`

Called by Spring Security's **request-based**, **method-based**, and **message-based** authorization components, making the final access-control decision.

```java
@FunctionalInterface
public interface AuthorizationManager<T> {   // T: HttpServletRequest, MethodInvocation, etc.

    default void verify(Supplier<Authentication> authentication, T object) {
        AuthorizationDecision decision = check(authentication, object);
        if (decision != null && !decision.isGranted()) {
            throw new AccessDeniedException("Access Denied");
        }
    }

    @Nullable
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);
}
```

> [!tip] A clever use of Template Method. `verify()` is a fixed `default` method enforcing an exception-based contract on top of the pluggable `check()` step — and `check()` returning `null` specifically means "I refrain from deciding," letting a delegating manager fall through to another one.

### The `AuthorizationManager` Hierarchy

```
                    AuthorizationManager
                            │
   ┌──────────────────┬────┴────┬───────────────────┐
RequestMatcherDelegating  AuthorityAuthorizationManager  PreAuthorizeAuthorizationManager
AuthorizationManager       AuthenticatedAuthorizationManager  PostAuthorizeAuthorizationManager
                            SecuredAuthorizationManager        Jsr250AuthorizationManager
```

- `RequestMatcherDelegatingAuthorizationManager` matches the request to the most appropriate delegate based on a `RequestMatcher` (path, headers, etc.) — this is what powers `authorizeHttpRequests(...)`.
- For **method security**, `AuthorizationManagerBeforeMethodInterceptor` and `AuthorizationManagerAfterMethodInterceptor` are AOP aspects automatically registered by `@EnableMethodSecurity`.

---

## 14. Authorizing HttpServletRequest

```
                              ① SecurityContextHolder
                                        │
                                        ▼
SecurityFilterChain                Authentication
     │                                  │
AuthorizationFilter ────② ────► RequestMatcherDelegatingAuthorizationManager
     │                                  │
     │                            Authorized?
     │                          ┌───────┴────────┐
     │                        Denied ③        Success ④
     │                    AccessDeniedException   Continue Processing Normally
```

### `AuthorizationFilter` Runs on Every Dispatch, Not Just the Original Request

`AuthorizationFilter` is **last by default**, and authorizes **every dispatch type** — not just the initial `REQUEST` — including `FORWARD`, `ERROR`, and `INCLUDE`:

```java
http.authorizeHttpRequests((authorize) -> authorize
    .dispatcherTypeMatchers(DispatcherType.FORWARD, DispatcherType.ERROR).permitAll()
    .requestMatchers("/endpoint").permitAll()
    .anyRequest().denyAll()
);
```

> [!tip] Authentication lookup is deferred. When a request is **always** permitted or **always** denied (matched before needing to check `Authentication` at all), the `Authentication` is never even queried — making those requests faster.

### Matching Requests — All the Ways

```java
// By path (MVC-aware matchers)
.requestMatchers("/endpoint").hasAuthority("USER")
.anyRequest().authenticated()

// By regex
.requestMatchers(RegexRequestMatcher.regexMatcher("/resource/[A-Za-z0-9]+")).hasAuthority("USER")

// By HTTP method
.requestMatchers(HttpMethod.GET).hasAuthority("read")
.requestMatchers(HttpMethod.POST).hasAuthority("write")

// Complex, combining several rules with AND
.requestMatchers("/db/**").access(allOf(hasAuthority("db"), hasRole("ADMIN")))
```

### Basic URL-Based Authorization

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.DELETE, "/api/orders/**").hasAuthority("ORDER_DELETE")
    .requestMatchers("/api/orders/**").hasAnyRole("USER", "ADMIN")
    .anyRequest().authenticated()
);
```

### A More Complete Example

```java
@Bean
SecurityFilterChain web(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authorize -> authorize
            .dispatcherTypeMatchers(FORWARD, ERROR).permitAll()
            .requestMatchers("/static/**", "/signup", "/about").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/db/**").access(allOf(hasAuthority("db"), hasRole("ADMIN")))
            .anyRequest().denyAll()
        );
    return http.build();
}
```

> [!important] Favor `permitAll()` over ignoring static resources entirely.
> 
> ```java
> http.authorizeHttpRequests(authorize -> authorize
>     .requestMatchers("/css/**").permitAll()
>     .anyRequest().authenticated()
> );
> ```
> 
> Historically, explicitly permitting resources this way (instead of telling Spring Security to "ignore" them entirely) came with a performance cost, since the session used to be consulted on every request. **As of Spring Security 6**, the session is **not pinged unless the matched authorization rule actually requires it** — this removed the performance excuse, so `permitAll()` for all requests (rather than bypassing the filter chain) is now the recommended, more secure default.

### `securityMatcher` — Scoping a Whole `SecurityFilterChain`

```java
http.securityMatcher("/api/**")
```

Decides whether **this entire `SecurityFilterChain`** applies to a given request at all — distinct from `requestMatchers(...)` inside `authorizeHttpRequests`, which decides authorization rules **within** an already-selected chain.

---

## 15. Method Security

### Why Use Method Security

- Extracting **fine-grained** authorization logic — especially when method **parameters and return values** contribute to the decision.
- Enforcing security at the **service layer**.
- Stylistically favoring **annotations** over `HttpSecurity`-based DSL configuration.

> [!tip] Enforce security as close to the data as possible. Ideally on `Repository` classes, since that's the lowest layer — the closer to the data, the fewer code paths can accidentally bypass the check.

### Enabling It

```java
@Configuration
@EnableMethodSecurity   // enables @PreAuthorize/@PostAuthorize/@Secured
public class MethodSecurityConfig { }
```

### The Method Security Annotations

```java
@Component
public class BankService {

    @PreAuthorize("hasRole('ADMIN')")
    public Account readAccount(Long id) { /* only invoked if Authentication has ROLE_ADMIN */ }

    @PreAuthorize("hasRole('USER') and #order.ownerId == authentication.principal.id")
    public void updateOrder(Order order) { /* access the argument + the principal in the SpEL expression */ }

    @PostAuthorize("returnObject.owner == authentication.name")
    public Account readAccount(Long id) { /* only RETURNED if the Account belongs to the caller */ }

    @PreFilter("filterObject.owner == authentication.name")
    public Collection<Account> updateAccounts(Account... accounts) {
        /* `accounts` will only contain the ones owned by the caller */
        return updated;
    }

    @PostFilter("filterObject.owner == authentication.name")
    public Collection<Account> readAccounts(String... ids) {
        /* the RETURN VALUE is filtered to only contain accounts owned by the caller */
        return accounts;
    }
}
```

|Annotation|Checked|Notes|
|---|---|---|
|`@PreAuthorize`|Before method execution|Most flexible — full SpEL, can reference method args, operates on arguments|
|`@PostAuthorize`|After method execution|Operates on the return value (`returnObject`)|
|`@PreFilter`|Before method execution|Filters a **collection argument** in place|
|`@PostFilter`|After method execution|Filters the **returned collection**|
|`@Secured`|Before method execution|Simpler, older, only supports role names — no SpEL|
|`@RolesAllowed`|Before method execution|JSR-250 standard annotation, portable outside Spring|

> [!danger] `@PreFilter`/`@PostFilter` should not be your primary access control. You should not even pass/return records the user doesn't own in the first place — this in-memory filtering happens **after** a potentially expensive fetch. Filter at the **data layer** (the query itself) whenever possible; use `@Pre/PostFilter` sparingly, being mindful it can be an expensive way to enforce something the repository query could have done directly.

> [!danger] Unannotated methods are NOT secured. With annotation-based method security, a method with **no** `@PreAuthorize`/etc. is simply **not protected at all**. Always declare a catch-all authorization rule in your `HttpSecurity` configuration as a safety net, so a forgotten annotation doesn't silently expose an endpoint.

> [!important] `@PreAuthorize` is implemented via AOP, and self-invocation bypasses it. Method security is a proxy-based `@Around`-style advice, wrapping the bean and checking the SpEL expression before/after invocation. Calling an `@PreAuthorize`-annotated method **from within the same class** bypasses the proxy and **skips the security check entirely** — the exact same self-invocation pitfall as `@Transactional`.

### Stacking `@PreAuthorize` — Not Supported, Use `and` in SpEL Instead

```java
@PreAuthorize("hasAuthority('permission:read')")
@PreAuthorize("hasAuthority('permission:bla')")   // NOT supported — only one @PreAuthorize per method
```

Combine with SpEL's `and`/`or` inside a **single** annotation instead.

### Request-Level vs Method-Level Authorization

||Request-level|Method-level|
|---|---|---|
|Granularity|Coarse-grained|Fine-grained|
|Configuration location|A dedicated config class|Local to the method declaration|
|Configuration style|DSL (`authorizeHttpRequests`)|Annotations|
|Authorization definitions|Programmatic|SpEL|

### SpEL Expressions for Authorization

|Reference|Meaning|
|---|---|
|`#paramName`|Refers to a method parameter or the return object, inside the SpEL evaluation context (`@PreAuthorize`/`@PostAuthorize`/`@PreFilter`/`@PostFilter`)|
|`@beanName`|Refers to any bean in the application context — e.g. `@myBean.authorize(#param)`|

Spring Security exposes a rich **root object** (`SecurityExpressionRoot`, specialized as `MethodSecurityExpressionRoot`) to every SpEL expression:

|Method|Meaning|
|---|---|
|`permitAll`|No authorization needed — `Authentication` is never even retrieved from the session|
|`denyAll`|Never allowed, under any circumstances — same "never retrieves `Authentication`" optimization|
|`hasAuthority('x')`|Requires an exact `GrantedAuthority` match|
|`hasRole('X')`|Shortcut for `hasAuthority`, auto-prefixing `ROLE_` (or the configured default prefix)|
|`hasAnyAuthority(...)` / `hasAnyRole(...)`|Matches any of several given values|
|`hasPermission(...)`|Hook into your own `PermissionEvaluator` for object-level authorization|

|Field|Meaning|
|---|---|
|`authentication`|The `Authentication` instance for this invocation|
|`principal`|`Authentication#getPrincipal` for this invocation|

### Using a Custom Bean in SpEL — Centralizing Authorization Logic

```java
@Component("authz")
public class AuthorizationLogic {
    public boolean decide(MethodSecurityExpressionOperations operations) {
        // authorization logic
    }
}
```

```java
@PreAuthorize("@authz.decide(#root)")
```

> [!tip] Why this is often the best approach. All authorization logic lives in **one independently unit-testable class**, with full access to the entire Java language — rather than being smeared across many `hasAuthority(...)` SpEL strings scattered through your codebase.

### Favor Granted Authorities Over Complicated SpEL

```java
@PreAuthorize("hasAuthority('permission:read') || hasRole('ADMIN')")   // works, but harder to maintain
```

Prefer a **`RoleHierarchy`** so `ROLE_ADMIN` implicitly grants `permission:read`:

```java
@Bean
static RoleHierarchy roleHierarchy() {
    return new RoleHierarchyImpl("ROLE_ADMIN > permission:read");
}
```

```java
@PreAuthorize("hasAuthority('permission:read')")   // now ADMINs pass too, without an OR clause
```

> [!warning] Composite/hierarchical roles are convenient but can hurt performance and maintainability if overused — keep hierarchies shallow.

### Specifying Order — Method Security Interceptors Are AOP Advisors

Each method-security annotation has its own AOP method interceptor, with a **fixed default position** in the Spring AOP advisor chain:

|Interceptor|Default Order|
|---|---|
|`@PreFilter`|100|
|`@PreAuthorize`|200|
|(others follow similarly)||
|`@EnableTransactionManagement`|`Integer.MAX_VALUE` (last, by default)|

> [!tip] Making a transaction stay open through `@PostAuthorize`. If you want `@Transactional` + `@PostAuthorize` on the same method, and you want a rollback to actually happen when `@PostAuthorize` denies access, the transaction needs to still be **open** when `@PostAuthorize` runs:
> 
> ```java
> @EnableTransactionManagement(order = 0)   // moves transaction advice BEFORE method-security advice
> ```
> 
> Method security is implemented as ordinary Spring AOP advice, subject to the exact same ordering rules as any other advisor.

---

## 16. Authorization Events

- For every **denied** authorization, an `AuthorizationDeniedEvent` is fired; for **granted** ones, an `AuthorizationGrantedEvent` **can** be fired.
- `AuthorizationGrantedEvent`s are **not published by default** — they have the potential to be extremely noisy (fired on essentially every successful check), so publishing them requires deliberate business logic to avoid flooding your event/logging infrastructure.

```java
@Component
public class MyAuthorizationEventPublisher implements AuthorizationEventPublisher {
    private final AuthorizationEventPublisher delegate;

    public MyAuthorizationEventPublisher(ApplicationEventPublisher publisher) {
        this.delegate = new SpringAuthorizationEventPublisher(publisher);
    }

    @Override
    public <T> void publishAuthorizationEvent(Supplier<Authentication> authentication,
                                               T object, AuthorizationDecision decision) {
        if (decision == null) return;
        if (!decision.isGranted()) {
            // publish only DENIED events, or add your own throttling logic here
        }
    }
}
```

---

## 17. Cryptography Fundamentals

A mathematical toolkit guaranteeing:

|Property|Meaning|
|---|---|
|**Confidentiality**|Only the intended receiver can read the message|
|**Authenticity**|We're sure of who sent the message|
|**Integrity**|The message wasn't modified in transit|
|**Non-repudiation**|The sender cannot deny having sent the message|

### Encoding — Not Security

A **two-way**, reversible transformation between formats (e.g. binary → text). Base64 is the classic example.

```
input ──encode()──► output ──decode()──► input   (fully reversible, no secret needed)
```

> [!danger] Encoding provides zero confidentiality. Anyone can decode it — this is exactly why a JWT's payload is readable by anyone, even though it's tamper-evident.

### Hashing — One-Way

```
input ──hash()──► fixed-length output   (cannot be reversed back to input)
```

- Same input always produces the same output.
- Used for password storage (with salt + adaptive cost) and for integrity checks.

### Encryption — Guarantees Confidentiality

**Symmetric Encryption**

```
Plain Text ──Encryption (Secret Key)──► Cipher Text ──Decryption (SAME Secret Key)──► Plain Text
```

- Same key encrypts and decrypts.
- e.g. **AES** (Advanced Encryption Standard).

**Asymmetric Encryption**

```
Plain Text ──Encryption (Public Key)──► Cipher Text ──Decryption (Private/Secret Key)──► Plain Text
```

- Encrypt with **either** key, decrypt with the **other**.
- e.g. **RSA** (Rivest–Shamir–Adleman).

### Public Key Certificate (Digital / Identity Certificate)

An electronic document proving the **validity of a public key** — includes the public key, information about the owner (the **subject**), and a **digital signature** from a verifying entity (the **issuer**). The signature is the **hash of the certificate body, encrypted with the issuer's private key**.

### Certificate Chains & Root CA

```
DigiCert (signed by MasterCert)
   ↓ signed by
MasterCert (signed by MasterCert — SELF-SIGNED)
   ↓
Root CA (trust anchor)
```

Trust flows from a **self-signed Root CA** down through a chain of signatures — your browser trusts a certificate because it can trace an unbroken signature chain back to a Root CA it already trusts.

### Digital Signature — Integrity + Authenticity + Non-repudiation (Asymmetric)

```
Alice: "Hello Bob!" ──Sign (Alice's PRIVATE key)──► Signed Message
Bob:   Signed Message ──Verify (Alice's PUBLIC key)──► confirms it's really from Alice, unmodified
```

### Message Authentication Code (MAC) — Integrity + Authenticity (Symmetric)

```
Sender:   Message + Key(K) → MAC Algorithm → MAC ──sent alongside the message──►
Receiver: Message + Key(K) → MAC Algorithm → MAC'
          Compare MAC == MAC'?  Same → authentic & intact.  Different → tampered/wrong sender.
```

> [!tip] Digital Signature vs MAC. Both prove integrity + authenticity — the difference is **asymmetric** (Digital Signature, adds non-repudiation since only the sender holds the private key) vs **symmetric** (MAC, both parties share the same secret key, so non-repudiation isn't guaranteed — either party could have produced the MAC).

---

## 18. JOSE Framework & JWT Structure 

A **JWT (JSON Web Token)** is a compact, URL-safe, **self-contained** token format — three Base64URL-encoded segments separated by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbGljZSIsImV4cCI6MTcxOjM0fQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
└────────────────────┘ └─────────────────────────────────────┘ └───────────────────────────────────────┘
       HEADER                          PAYLOAD                                  SIGNATURE
```

### 1. Header

Metadata about the token itself.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- `alg` — signing algorithm (`HS256` = HMAC-SHA256 symmetric, `RS256` = RSA asymmetric, common for OAuth2/OIDC identity providers).

### 2. Payload (Claims)

The actual data. **Not encrypted** — just Base64URL-**encoded**, so anyone can decode and read it (never put secrets here).

```json
{
  "sub": "alice",
  "roles": ["USER"],
  "iat": 1719000000,
  "exp": 1719003600
}
```

|Claim|Meaning|
|---|---|
|`sub`|Subject — who the token represents|
|`iss`|Issuer — who created/signed the token|
|`aud`|Audience — intended recipient(s)|
|`exp`|Expiration time (Unix timestamp)|
|`iat`|Issued-at time|
|`nbf`|"Not before" — token not valid until this time|
|Custom claims|Anything else the issuer wants to embed (roles, tenant ID, etc.)|

### 3. Signature

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

- Proves the token **hasn't been tampered with** and **really came from the issuer** — a receiver with the correct verification key (the shared secret for HMAC, or the issuer's public key for RSA/ECDSA) can recompute the signature and compare.
- If a client edits the payload (e.g. changes `"roles":["USER"]` to `["ADMIN"]`), the signature no longer matches, and verification fails.

> [!danger] Encoded ≠ Encrypted. Anyone can decode a JWT's payload without any secret — it's plainly visible (try pasting one into jwt.io). The signature only guarantees **integrity/authenticity**, not **confidentiality**. Never put passwords, SSNs, or other sensitive data directly in JWT claims.

### The JOSE Framework — The Standards Family Behind JWTs

**JOSE (JSON Object Signing and Encryption)** is a collection of related standards:

|Standard|Purpose|
|---|---|
|**JWT** — JSON Web Token|Compact, standardized way to transmit information between parties as JSON|
|**JWS** — JSON Web Signature|Specification for digitally signing JSON data|
|**JWE** — JSON Web Encryption|Specification for encrypting JSON data|
|**JWK** — JSON Web Key|Specification for representing cryptographic keys in JSON format|
|**JWA** — JSON Web Algorithms|Specification defining the cryptographic algorithms usable in JWS/JWE|

```json
{
  "kty": "RSA",
  "kid": "rsa-key-1",
  "use": "sig",
  "n": "0vx7agoebGcQSuuPiLJXZpt...",
  "e": "AQAB"
}
```

A JWK like this represents a public key in JSON form, identified by a **key ID** (`kid`) — used to verify a JWS-signed JWT.

> [!important] What's actually used for authentication, in practice. A **JWT signed with JWS using a JWA**, verifiable with a public key distributed via **JWK**. This is the concrete pipeline underneath the JWT header/payload/signature structure above.

### JWT Signing: Digital Signature vs HMAC

- **99% of real-world JWTs are digitally signed** (asymmetric — e.g. `RS256`), which is what allows a **Resource Server** to verify a token's authenticity using only the Authorization Server's **public** key, without ever holding a shared secret.
- The remainder use an **HMAC** signature (symmetric — e.g. `HS256`), which requires both parties to share the same secret — practical mainly when the issuer and verifier are the same service or tightly coupled.

---

## 19. OAuth 2.0 & OIDC 

### OAuth 2.0 — Delegated Access Control

**The problem it solves**: "I have an application, and I want it to access your Google Drive on your behalf — how do we do that safely, without you giving my application your actual Google password?" OAuth2 is an **authorization** framework (not authentication) that lets a third-party application access a user's resources **without ever seeing their password**.

### OpenID Connect (OIDC) — User Authentication, Built on Top of OAuth 2.0

**The problem it solves**: "I want you to log in with your Google account, instead of creating a new account with me." Built directly on OAuth 2.0 with minimal additions.

> [!tip] Doing both at once. If you want to **both** authenticate the user **and** access APIs on their behalf, you use OAuth 2.0 and OIDC together — this is in fact the most common real-world setup ("Login with Google" that also requests Drive access, for instance). The common confusion: OAuth2 alone doesn't actually tell the Client **who** the user is — it just proves the Client got access to _something_. "Login with Google/Facebook/GitHub" buttons are OIDC (which uses OAuth2 under the hood), not plain OAuth2 — the distinguishing giveaway is the presence of an **ID Token** and the `openid` scope.

### Actors — Same People, Different Names

|Actor|OAuth 2.0 Term|OIDC Term|
|---|---|---|
|The user|**Resource Owner (RO)**|**End-User**|
|The API|**Resource Server (RS)**|—|
|The Authorization Server|**Authorization Server (AS)**|**OpenID Provider**|
|The requesting app|**Client**|**Relying Party**|

![[Pasted image 20260803182850.png|700]]

> [!tip] Client ≠ Resource Server, and often Authorization Server ≠ Resource Server too. In many setups, the Authorization Server (e.g. Keycloak) is a completely separate system from the actual APIs (Resource Servers) that use its tokens — the Resource Server just needs to be able to **validate** tokens issued by the Authorization Server (often via a shared public key or an introspection endpoint), not to run the login flow itself.

### OAuth2 vs OIDC — Side by Side

||OAuth2|OIDC (OpenID Connect)|
|---|---|---|
|**Purpose**|**Authorization** — "can this app access this resource?"|**Authentication** — "who is this user?" — built as a thin identity layer **on top of** OAuth2|
|**What you get back**|Access Token (opaque or JWT, meant for the Resource Server)|Access Token **+ ID Token** (always a JWT, meant for the **Client** to learn who logged in)|
|**Standard claims**|None mandated|`sub`, `name`, `email`, `email_verified`, etc. — standardized in the ID Token|
|**Typical use**|"Let this app post to my Twitter"|"Login with Google"|

### Why Use These Standards At All?

Authentication and authorization are **supporting domains**, not your product's **core domain** — don't waste engineering effort reinventing this notoriously hard-to-get-right wheel. Use a mature provider (**Keycloak**, Auth0, Okta, etc.) that already correctly implements both standards.

---

## 20. OAuth 2.0 Grants / OIDC Flows

> [!info] Grants and Flows are the same thing. "OAuth 2.0 Grant" and "OIDC Flow" describe the same concept: the specific sequence of messages actors exchange to reach the end goal (authentication and/or delegated access).

### Authorization Code Flow (Recommended, Confidential Clients)

The most common, most secure flow — used whenever a **confidential client** (a server-side app that can keep a secret) needs a user to log in.

![[Pasted image 20260803182606.png]]

- Note the `code_challenge`/`code_verifier` pair in the diagram — that's **PKCE** (see below), shown here as part of the _modern_ Authorization Code flow rather than a separate add-on.
- Also enables **Single Sign-On (SSO)**: since the browser already holds a session cookie with the Authorization Server, a second application can silently redirect through the same `/auth` endpoint and get tokens without the user re-entering credentials.

### Why the Extra "Code" Step?

The **Authorization Code** is a short-lived, one-time-use intermediate credential passed through the **browser's URL** (which is comparatively exposed — visible in browser history, logs, referrer headers). The actual **Access Token** is exchanged in a direct, back-channel server-to-server call, authenticated with the client's secret — meaning the sensitive token itself is **never exposed** in the browser/URL.

### Spring Security OAuth2 Client Configuration

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: your-client-id
            client-secret: your-client-secret
            scope: openid, profile, email
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
            token-uri: https://oauth2.googleapis.com/token
```

```java
http.oauth2Login(Customizer.withDefaults()); // enables the full Authorization Code flow via Spring
```

### Resource Owner Password Flow — ❌ Deprecated

```
Resource Owner        Application        Authorization Server    Resource Server
    │──① credentials────►│──② send credentials───►│                        │
    │                    │◄─③ access token────────│                        │
    │                    │──④ request + token─────────────────────────────►│
    │                    │◄─⑤ resource─────────────────────────────────────│
```

**Why deprecated**: the application itself handles the user's raw credentials — defeating the entire point of OAuth2 (never sharing your password with the requesting app).

### Implicit Flow — ❌ Deprecated

```
Resource Owner        Application        Authorization Server     Resource Server
    │──① access app──────►│                        │                        │
    │◄─② auth request─────│                        │                        │
    │──③ authenticate + grant─────────────────────►│                        │
    │                    │◄─④ access token in URL──│                        │
    │                    │──⑤ request + token in URL───────────────────────►│
    │                    │◄─⑥ resource──────────────────────────────────────│
```

**Why deprecated**: the token is passed in **plaintext in the redirect URL** — visible in browser history, referrer headers, and server logs, opening many attack vectors.

### Client Credentials Flow — Machine-to-Machine

Used for **machine-to-machine** communication — **no user** is involved at all (e.g. one microservice calling another's API).

```
Application          Authorization Server        Resource Server
    │──① client_id + secret───►│                        │
    │◄─② access token──────────│                        │
    │──③ request + token────────────────────────────────►│
    │◄─④ resource────────────────────────────────────────│
```

- **Only in OAuth 2.0** — there's no user involved at all, so OIDC's identity layer is irrelevant here.
- For **confidential clients only** — servers talking to other servers, where a secret can be safely kept.
- **All clients must be registered with the Authorization Server** beforehand.
- **Confidential vs Public clients**: confidential clients hold a password (`client_secret`); public clients (mobile apps, SPAs) can't safely hold one — there's no secure place to store it.
- This is the standard way one internal service proves its identity to another via OAuth2, in a microservices architecture.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          internal-service:
            provider: keycloak
            client-id: order-service
            client-secret: ${ORDER_SERVICE_SECRET}
            authorization-grant-type: client_credentials
```

---

## 21. PKCE

**PKCE (Proof Key for Code Exchange)**, pronounced "pixy," is an extension to the Authorization Code flow designed for **public clients** that **can't** safely keep a client secret — mobile apps, single-page apps (SPAs) running entirely in the browser.

### The Problem It Solves

Without PKCE, a malicious app on the same mobile device could potentially intercept the redirect containing the Authorization Code and exchange it for tokens itself, since a public client has no secret to prove it's the legitimate requester.

### How It Works

```
1. Client generates a random "code_verifier" (kept secret, in memory)
2. Client computes "code_challenge" = SHA256(code_verifier)
3. Client sends the code_challenge in the initial /authorize request
4. Authorization Server stores the code_challenge alongside the issued Authorization Code
5. Client exchanges the code for tokens, this time sending the ORIGINAL code_verifier
6. Authorization Server re-computes SHA256(code_verifier) and checks it matches
   the code_challenge from step 4 — proves this is the SAME client that started the flow
```

> [!important] PKCE is now recommended for **all** clients, not just public ones. Modern OAuth2 best practice (OAuth 2.1 draft) recommends PKCE universally, even for confidential/server-side clients — it's cheap to add and closes a class of authorization-code-interception attacks regardless of client type.

---

## 22. Tokens — ID Token, Access Token, Refresh Token

### ID Token — Authenticates the User to the Client

- Purpose: let the **Client** know who logged in (e.g. "Login with Google," or populating "logged in as ___" in your app's frontend/backend).
- Always a **JWT**. Key claims:

|Claim|Meaning|
|---|---|
|`exp` / `iat`|Expiration / issued-at time|
|`auth_time`|When the user actually authenticated|
|`jti`|Unique token ID|
|`iss`|Issuer URI|
|`aud`|Audience — who this token is meant for (must include the Relying Party it was issued to)|
|`sub`|The user's unique, stable identity|
|`typ`|Token type (`"ID"`)|
|`azp`|The Relying Party the token was issued to|
|`email_verified`, `name`, `preferred_username`, `given_name`, `family_name`, `email`|Standard profile/contact claims|

- Which claims appear depends on the **scope** requested in the OIDC auth request.
- The same claims can alternatively be fetched by `POST`ing an access token to the Authorization Server's `userinfo` endpoint.

|Scope|Claims Returned|
|---|---|
|`openid` (required)|`sub` (always), plus `iss`, `aud`, `exp`, `iat`, `at_hash` on the ID Token|
|`profile`|`name`, `family_name`, `given_name`, `middle_name`, `nickname`, `picture`, `updated_at`|
|`email`|`email`, `email_verified`|

### Access Token — Authorizes API Access

**Self-contained (JWT) Access Tokens**

- Same JWT structure as an ID token, but with different claims oriented around **authorization**, not identity:

|Claim|Meaning|
|---|---|
|`aud`|Typically the API/audience the token is valid for (e.g. `"account"`)|
|`typ`|`"Bearer"`|
|`allowed-origins`|Used for CORS preflight checks by Resource Servers|
|`realm_access.roles`|Global roles — the **intersection** of roles granted to the user and roles the client has access to|
|`resource_access.<client>.roles`|Client-specific roles|
|`scope`|Decides both what claims to include **and** what APIs the token can access|

**Opaque / Reference Tokens**

- Just an **identifier** — the Resource Server must call a **token introspection endpoint** to learn the token's actual details.
- **Better revocation properties** (can be invalidated instantly server-side) **at the cost of performance** (extra network round-trip per validation).
- Only genuinely worth the trade-off if you have **automatic fraud/anomaly detection** — otherwise, if your access tokens already expire in, say, 5 minutes, the improved revocation speed of an opaque token is largely moot (it'll expire on its own almost as fast anyway).

### Refresh Token — Getting New Access Tokens Without Re-Login

Access tokens are deliberately **short-lived** (minutes to hours) to limit the damage if one is stolen. Instead of forcing the user to log in again every time it expires, the client uses a longer-lived **Refresh Token** to silently get a new Access Token.

```
Client                                   Authorization Server
   │──1. access_token expired──────────────────►│
   │        (received 401 from Resource Server)  │
   │──2. POST /token  grant_type=refresh_token──►│
   │       + refresh_token                        │
   │◄──3. new access_token (+ possibly new       │
   │        refresh_token — "rotation")───────────│
```

**Why not just issue a long-lived access token instead?**

- Revoking a long-lived access token is painful — the Authorization Server would need to track every issued token and its status.
- Short-lived access tokens **reduce the blast radius** if one leaks.
- Refresh tokens let you **re-authenticate silently**, periodically, without resending credentials.
- **Refresh token rotation** gives better security properties (see below).
- Refresh tokens let user info **updates propagate** without forcing a full logout/login cycle.

> [!danger] If an attacker gets your refresh token, you're compromised regardless. A stolen refresh token gives **long-term** account access — everything below is about minimizing that risk, not eliminating it entirely.

### Where to Store Refresh Tokens

|Client Type|Storage|
|---|---|
|**Confidential client**|Preferably **encrypted** storage; `client_secret` is additionally required to actually use the token|
|**Public client (SPA)**|In-memory (e.g. via a library like KeycloakJS) — preferably via the **BFF pattern** (next section) instead of holding tokens in the browser at all|
|**Mobile client**|OS-secured storage, protected by the user's biometric/device credentials|

### How Long Should Refresh Tokens Live?

Wildly varies by context and risk tolerance — e.g. LinkedIn: months; a tightly-controlled enterprise app: as short as 12 hours. **Whatever you decide is a reasonable trade-off for your threat model.** Users/clients can also explicitly **revoke** refresh tokens ("Disconnect this app," "Connected Apps" settings).

### Refresh Token Rotation

Every time you receive new tokens, you must use the **latest** refresh token going forward — this applies both to refresh token rotation itself and to **key rotation** (the signing key used may also change over time).

```
Timeline:
Obtain AT1/RT1 (Auth Code flow)
     │
   AT1 expires → use RT1 → receive AT2/RT2
                                   │
                                 AT2 expires → use RT2 → receive AT3/RT3
```

Each refresh **issues a brand-new refresh token** and **invalidates the old one** — using an old, already-rotated-out refresh token is a strong signal of **theft**. Rotation makes stolen refresh tokens far less useful, since re-using an old one after rotation is a strong signal of theft.

### Refresh Token Reuse Detection

```
Obtain AT1/RT1 → AT1 expires → use RT1 → receive AT2/RT2
                                              │
                          Attacker steals RT2 from the user's browser
                                              │
                          Attacker uses RT2 → receives AT3/RT3
                                              │
              (later) Legitimate app also tries to use RT2 → ERROR
                                              │
        Authorization Server detects RT2 REUSE → revokes the ENTIRE token chain (including RT3)
                                              │
                    Attacker's RT3 is now revoked too — locked out
```

The Authorization Server detecting that an **already-used** refresh token was presented again is a red flag — it revokes the whole downstream chain of tokens derived from it, limiting the damage even after a theft.

### Known Attack Patterns Against Refresh Token Rotation

|Scenario|Attack|
|---|---|
|**Stealing access tokens**|Malicious code in the JS environment extracts tokens the SDK is actively using and exfiltrates them to an attacker server|
|**Sidestepping rotation**|Attacker monitors the app, silently keeps running the refresh flow in parallel, harvests new tokens continuously, and waits for the legitimate app to go idle before using stolen refresh tokens|
|**Impersonating the legitimate client**|—|
|**Silently requesting new tokens via a hidden iframe**|Load a silent Authorization Code flow in an invisible iframe, exploit an active SSO session/cookie to get a fresh code without user interaction, then exfiltrate the resulting tokens|

> [!tip] Silent re-authentication done legitimately. `/auth?scope=openid...&prompt=none` lets a **legitimate** client silently obtain a new access token if the user already has an active SSO session — the same mechanism attackers abuse above, used as intended.

---

## 23. Backend For Frontend (BFF) Pattern

```mermaid
flowchart TB
    subgraph TB1["Trust Boundary"]
        FE[Frontend]
        BE[Backend<br/>cookie-based session]
        FE <-->|"session cookie"| BE
    end

    BE -->|"① Run Authorization Code flow"| AS[Authorization Server]
    AS -->|"② Issue access + refresh token"| BE

    BE -->|"③ Proxies API requests,<br/>attaching its own access token"| API[API]
```

- The **frontend never sees or handles OAuth2 tokens at all** — it only holds a plain **cookie-based session** with its own backend.
- The **backend** is the actual OAuth2 client: it runs the Authorization Code flow, stores tokens **server-side**, and proxies API calls on the frontend's behalf, attaching the access token itself.
- **Why this is safer for SPAs**: tokens never touch browser-accessible JavaScript storage at all (no `localStorage`, no in-memory JS variables reachable by XSS), and the backend can apply all the best practices available to confidential clients (client authentication, sender-constrained tokens) that a pure browser-based SPA fundamentally cannot.

---

## 24. Keycloak Terminology & Sessions

Keycloak is a widely used, open-source **Authorization Server / OpenID Provider** implementation.

|Term|Meaning|
|---|---|
|**Realm**|A fully isolated **tenant** — its own configuration, users, and applications. Lets one Keycloak installation serve multiple, unrelated purposes (e.g. one realm for internal apps/employees, another for external customers)|
|**User**|A user within a specific realm|
|**Group**|A group of users — members **inherit** attributes (e.g. office address) and **roles** from the group|
|**Realm Role**|A global role, scoped to the whole realm|
|**Composite Role**|A role that itself **contains other roles** — analogous to method-security role hierarchies|
|**Client**|An OAuth2 client / OIDC relying party, configured via **Client Scopes**|
|**Client Scope**|Determines both what **claims** get included in the token, and what **APIs** the token can access|

### Keycloak's Two-Level Session Model

**Level 1 — SSO Session (a.k.a. User Session)**

- Created to track user activity **regardless of which client** they're using — the Keycloak equivalent of an `HttpSession`, used to maintain state across multiple requests from the same agent.

|Setting|Meaning|
|---|---|
|**SSO Session Max**|Absolute maximum session lifetime (e.g. 12 hours)|
|**SSO Session Idle**|How long a session can sit idle before premature expiry (e.g. 30 minutes) — refreshed every time the user interacts with Keycloak, directly or via a client refreshing tokens. If **Client Session Idle** isn't set, this value **is** the effective refresh token lifetime|
|**Client Session Idle**|Same idea, but overridable per client|
|**Client Session Max**|Same idea, but overridable per client|
|**Access Token Lifespan**|How long Access and ID tokens live|

**Level 2 — Client Session**

- Created per **client** the user is authenticated to, within a single SSO session — tracks activity for that specific client and is tied directly to the validity of the tokens that client is using.

---

## 25. Spring Security's OAuth 2.0 Support

Spring Security ships **three (really four)** distinct pieces of OAuth2/OIDC support, each solving a different role in the actor table above:

|Feature|Role Played|Use When|
|---|---|---|
|**OAuth 2.0 Login**|Client / Relying Party (login use case)|You want "Login with Google/GitHub"|
|**OAuth 2.0 Client**|Client (API-access use case)|Your app needs a token **on behalf of a user** to call another API secured with OAuth2|
|**Spring Authorization Server**|Authorization Server / OpenID Provider|You're building your **own** OAuth2/OIDC provider (implements the OAuth 2.1 + OpenID Connect 1.0 specs, built on top of Spring Security)|
|**Spring Security OAuth 2.0 Resource Server**|Resource Server|Your API needs to validate incoming Bearer tokens/JWTs|

### Resource Server Details

- Supports validating **JWTs** directly (verifying the signature against a JWK Set) as well as **opaque tokens** via introspection.
- Supports **multi-tenancy** — validating tokens from **multiple** Authorization Servers and/or **multiple token types** simultaneously, useful when an API must serve clients authenticated by different identity providers.

---

## 26. Common Web Attacks & Protections

### Clickjacking

An attacker embeds your page in an invisible/disguised `<iframe>`, tricking users into clicking something they can't actually see (e.g. "Claim your prize" overlaying a real "Confirm bank payment" button underneath).

```
Response Header: X-Frame-Options: DENY
```

Prevents your site from being framed by any other origin at all — this is one of the default headers Spring Security sends automatically.

### Cross-Site Request Forgery (CSRF)

A malicious site auto-submits a form to your bank while the victim's browser still holds a valid session cookie:

```html
<!-- Legitimate transfer form on YOUR site -->
<form method="post" action="/transfer">
  <input name="amount"><input name="routingNumber"><input name="account">
</form>
```

```html
<!-- Malicious form on an EVIL site, using hidden fields pre-filled with the attacker's data -->
<form method="post" action="https://bank.example.com/transfer">
  <input type="hidden" name="amount" value="100.00">
  <input type="hidden" name="routingNumber" value="evilsRoutingNumber">
  <input type="hidden" name="account" value="evilsAccountNumber">
  <input type="submit" value="Win Money!">
</form>
```

The victim clicks "Win Money!" and unknowingly transfers $100 — the evil site **can't read** the victim's cookies, but the **browser still automatically attaches** the bank's cookies to the request, since it's targeting the bank's own domain.

> [!important] CSRF is fundamentally a **cookie-based authentication** problem. It only matters if you use cookies as your authentication mechanism — directly relevant for SSR apps and the BFF pattern. Safe methods (`GET`, `HEAD`, `OPTIONS`, `TRACE`) should always be **read-only**, since CSRF protection is normally only enforced on state-changing methods.

### `SameSite` Cookie Attribute

|Term|Definition|
|---|---|
|**Origin**|Scheme + host + port — `http://blog.example.com:80` — everything counts, exact match required|
|**Site**|Registrable domain (TLD+1) — `blog.example.org`'s site is `example.org`, so a request from `docs.example.org` is considered **same-site** but still **cross-origin**|
|**`Lax`**|Sends the cookie on cross-site **read-only** (top-level navigation, `GET`) requests|
|**`Strict`**|Sends the cookie **only** on same-site requests|

> [!warning] `SameSite` alone doesn't fully solve CSRF. It helps, but doesn't protect against every Cross-Origin Request Forgery scenario on its own — the **Synchronizer Token Pattern** is still required, especially for SSR applications. Also: **Spring Security does not create the `HttpSession` for you** — you must explicitly configure session/cookie behavior yourself in SSR applications.

### Protecting Against CSRF

**SSR apps — Synchronizer Token Pattern (required)**

- Spring Security's `CsrfFilter` requires a valid, unguessable **CSRF token** on every state-changing request (`POST`/`PUT`/`PATCH`/`DELETE`) when using **session-based** authentication.
- The token is delivered to the client (via a cookie or a hidden form field) and must be echoed back in a request header/param — an attacker's cross-site form can't read it due to the browser's same-origin policy, so forged requests fail.

```html
<form method="post" action="/transfer">
  <input type="hidden" name="_csrf" value="4bfd1575-3ad1-4d21-96c7-4ef2d9f86721">
  <input type="text" name="amount">
  <input type="text" name="routingNumber">
  <input type="hidden" name="account">
  <input type="submit" value="Transfer">
</form>
```

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

**SPA + REST API apps**

- Rely on **`SameSite` cookies + CORS** (allow only your SPA's specific origin) — if configured correctly, this alone can be sufficient to prevent CSRF, making a separate token largely redundant.
- Alternatively, a **cookie-based synchronizer token** works for SPAs too.

> [!important] CSRF protection is usually **disabled** for stateless JWT APIs. If you're **not** using cookies/sessions for authentication (i.e. pure stateless Bearer-token APIs), CSRF protection is unnecessary and typically disabled: a token in an `Authorization` header can't be silently attached by a forged cross-site request the way a cookie can. **CSRF is fundamentally a cookie/session problem** — if there's no ambient credential the browser sends automatically, there's no CSRF vector to protect against.
> 
> ```java
> http.csrf(csrf -> csrf.disable());
> ```

### CORS

```
Web document (domain-a.com)
  ├── same-origin requests → domain-a.com's server         → ALWAYS allowed
  └── cross-origin requests → domain-b.com's server (image, │
      webfont, etc.)                                          → controlled by CORS
```

Main request (the page's own origin) always defines the origin; any resource fetched from a **different** origin is subject to CORS rules. Spring provides both a generic **Spring Security CORS support** (if not using Spring MVC) and **Spring MVC CORS support** (preferred, when available).

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://myfrontend.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

> [!warning] Never combine a wildcard `*` origin with `allowCredentials(true)` — browsers reject that combination outright, and it's a security anti-pattern anyway.

### Cross-Site Scripting (XSS)

Treating **data as code** — a vulnerable site returns attacker-supplied JavaScript to other users' browsers, which then executes with full access to that user's session (stealing passwords, sensitive data, triggering wire transfers, etc.).

|Type|Mechanism|
|---|---|
|**Reflected XSS**|The simplest form — an application takes data from the **current** request and reflects it back **unsafely** in the immediate response|
|**Stored XSS**|The application persists data from an untrusted source, then unsafely includes it in **later** responses to other users|

```
GET /status?message=<script>/*Bad stuff here...*/</script>
→ <p>Status: <script>/* Bad stuff here... */</script></p>   ← reflected directly into the page, unescaped
```

### Protecting Against XSS

1. **Sanitize input** — required if you must dynamically render user-generated HTML (use a library like **DOMPurify** to strip malicious patterns).
2. **Encode output** — preferred, general-purpose defense: render user content as **text**, never as raw HTML.

```jsx
// React automatically escapes user-generated content — safe by default
const title = response.potentiallyMaliciousInput;
const element = <h1>{title}</h1>;   // rendered as a harmless string, not executed as HTML/JS
```

3. **Content Security Policy (CSP)** — an HTTP header specifying which sources of scripts/styles/etc. the browser is allowed to execute at all, providing defense-in-depth even if an injection slips through.

### SQL Injection

Also fundamentally **treating data as code** — unsanitized input is concatenated directly into a SQL query string.

```java
// VULNERABLE — user input concatenated directly into the query
String query = "SELECT * FROM products WHERE category = '" + input + "'";
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(query);
```

An attacker supplying `' UNION SELECT username, password FROM users--` as `input` can make the query return **every username and password** in the database instead of the intended product list.

### Protecting Against SQL Injection

```java
// SAFE — prepared statement, parameter bound separately from the query structure
PreparedStatement statement = connection.prepareStatement("SELECT * FROM products WHERE category = ?");
statement.setString(1, input);
ResultSet resultSet = statement.executeQuery();
```

- A **prepared statement** is parsed by the database **first**; its `?` parameters are only ever treated as **data**, never as executable SQL — no matter what the input string contains.
- **Never** build SQL by string-concatenating code and parameters together — this is the single most important rule.

---

## 27. Quick Revision Cheat Sheet

### Interview-Style Questions

- Explain the Spring Security request lifecycle / filter chain.
- What is `SecurityFilterChain`?
- What is `UserDetailsService` and `UserDetails`?
- What is `PasswordEncoder` and why must passwords never be stored in plain text?
- What is the difference between Authentication and Authorization?
- What is JWT (JSON Web Token) and how does it work?
- What is the difference between stateful (session-based) and stateless (JWT-based) security?
- What is `@PreAuthorize` and `@Secured`?
- What is CSRF and when should you disable it?
- What is CORS and how do you configure it in Spring?
- What is OAuth2 and how does Spring Security support it?
- What is the role of `OncePerRequestFilter`?
- What is the difference between roles and authorities in Spring Security?
- What is method-level security and how do you enable it?
- How do you implement remember-me authentication?

### Concept Table

|Concept|Remember|
|---|---|
|Authentication vs Authorization|Who are you? (`401` if it fails) vs What can you do? (`403` if it fails)|
|Principal / Authority / Authentication / Authorization|Who you are / what you're permitted / proving identity / checking permission|
|RBAC / ABAC / ReBAC|Role-based / attribute-based (e.g. time of day) / relationship-based (e.g. "did you create this?")|
|Stateless (JWT)|No server session — scales without sticky sessions/distributed session store|
|`DelegatingFilterProxy`|Bridges a Spring bean `Filter` into the servlet container's filter chain — Virtual Proxy pattern|
|`FilterChainProxy`|Spring Security's actual registered filter — a whole chain, packed into one filter|
|`SecurityFilterChain`|One matchable, ordered list of filters — you can register several; configured via a `@Bean`|
|`ExceptionTranslationFilter`|Converts `AuthenticationException`/`AccessDeniedException` into the right HTTP response|
|`RequestCache`|Saves the original request so it can be replayed after login|
|`PasswordEncoder` / BCrypt|Salted, adaptive, one-way hash — never plaintext, never reversible encryption|
|`DelegatingPasswordEncoder`|Default encoder — supports migrating algorithms + legacy hashes via `{id}` prefix|
|`SecurityContextHolder`|Where the current `Authentication` lives — Spring Security doesn't care how it got populated|
|`ProviderManager`|Delegates to a list of `AuthenticationProvider`s; supports an optional parent for shared auth|
|`AuthenticationEntryPoint`|Requests credentials from an unauthenticated client (redirect / `WWW-Authenticate`)|
|`hasRole()` vs `hasAuthority()`|`hasRole("X")` auto-prefixes `ROLE_X`; `hasAuthority` matches the exact string|
|`GrantedAuthority`|One permission; `SimpleGrantedAuthority` is the concrete String-based implementation|
|`AuthorizationManager`|The final access-control decision-maker — request-based, method-based, message-based|
|`AuthorizationFilter` runs on|Every **dispatch type**, not just the original request|
|`permitAll()` over ignoring|Recommended since Spring Security 6 — session no longer pinged unless the rule needs it|
|`@PreAuthorize`/`@PostAuthorize`/`@PreFilter`/`@PostFilter`|Before-args / after-return / filter-args / filter-return|
|`@PreAuthorize` self-invocation|Method security is AOP-proxy-based — calling from within the same class bypasses the check entirely, same as `@Transactional`|
|Unannotated methods|NOT secured — always keep a catch-all `HttpSecurity` rule as a safety net|
|Method security = AOP|Subject to advisor ordering — `@PreFilter`=100, `@PreAuthorize`=200, transactions default to last|
|Encoding vs Hashing vs Encryption|Reversible/no secret vs one-way vs reversible-with-a-key|
|Digital Signature|Asymmetric — integrity + authenticity + non-repudiation|
|MAC|Symmetric — integrity + authenticity, no non-repudiation|
|JWT structure|Header.Payload.Signature — payload is encoded, **not encrypted**|
|JOSE = JWT + JWS + JWE + JWK + JWA|The full JSON crypto standards family|
|OAuth 2.0|Delegated **authorization**|
|OAuth2 vs OIDC|OAuth2 = authorization; OIDC = authentication (adds the ID Token)|
|Authorization Code flow|The recommended flow — code exchanged server-side, PKCE for public clients|
|Client Credentials flow|Machine-to-machine, no user, confidential clients only|
|Implicit / ROPC flows|Both deprecated — token-in-URL exposure / app handles raw credentials|
|PKCE|Protects public clients (mobile/SPA) that can't hold a secret; now recommended for everyone|
|Refresh token rotation|New token every refresh; reuse of an old one signals theft → whole chain revoked|
|BFF pattern|Tokens never touch the browser — backend holds them, frontend just gets a cookie session|
|Keycloak Realm|Fully isolated tenant|
|SSO Session vs Client Session|Per-user-across-all-clients vs per-client-within-that-user-session|
|CSRF|A cookie-auth problem — Synchronizer Token Pattern (SSR) or SameSite+CORS (SPA)|
|XSS|Sanitize input / encode output / CSP — treating data as code is the root cause|
|SQL Injection|Prepared statements only — never concatenate SQL and parameters|