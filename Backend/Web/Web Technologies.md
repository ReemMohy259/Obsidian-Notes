---
share_link: https://share.note.sx/ljmgu30f#B+9z3t1pndGpRGRNB/pVWA
share_updated: 2026-07-11T05:43:25+03:00
---

## Table of Contents

- [[#1. HTTP Fundamentals]]
- [[#2. HTTP Methods]]
- [[#3. Headers, Cookies, and Sessions]]
- [[#4. Session vs Cookie]]
- [[#5. HTTP Status Codes (1xx–5xx)]]
- [[#6. REST Principles — Statelessness & Idempotency]]
- [[#7. REST vs RESTful APIs]]
- [[#8. Richardson Maturity Model]]
- [[#9. HATEOAS]]
- [[#10. API Versioning & Content Negotiation]]
- [[#11. SOAP — XML, WSDL]]
- [[#12. REST vs SOAP]]
- [[#13. Servlet Lifecycle]]
- [[#14. Filters vs Listeners (vs Interceptors)]]
- [[#15. Web Server vs Application Server]]
- [[#16. WebSockets vs HTTP vs Server-Sent Events (SSE)]]
- [[#17. Common Web Attacks (Web Security 101)]]
- [[#18. Quick Revision Cheat Sheet]]

---

# 1. HTTP Fundamentals

**HTTP (HyperText Transfer Protocol)** is a stateless, text-based (originally) request/response protocol built on TCP (or QUIC for HTTP/3), used to transfer resources between clients and servers.

### HTTP Versions at a Glance

|Version|Key Characteristic|
|---|---|
|HTTP/1.0|One request per TCP connection (connection closed after each response)|
|HTTP/1.1|Persistent connections (`Keep-Alive`), pipelining, chunked transfer encoding|
|HTTP/2|Binary framing, **multiplexing** multiple requests over one connection, header compression (HPACK), server push|
|HTTP/3|Runs over **QUIC** (UDP-based) instead of TCP — eliminates head-of-line blocking, faster connection setup|

### Anatomy of a Request/Response

```
GET /api/orders/1 HTTP/1.1          ← request line: method, path, version
Host: example.com                    ← headers
Accept: application/json
Authorization: Bearer eyJ...

                                      ← blank line
(no body for GET)
```

```
HTTP/1.1 200 OK                      ← status line
Content-Type: application/json       ← headers
Content-Length: 42

{"id": 1, "name": "Widget"}          ← body
```

---

# 2. HTTP Methods

|Method|Purpose|Idempotent?|Safe?|Has Body?|
|---|---|---|---|---|
|`GET`|Retrieve a resource|✅|✅|No (conventionally)|
|`POST`|Create a resource / trigger an action|❌|❌|Yes|
|`PUT`|Replace a resource entirely|✅|❌|Yes|
|`PATCH`|Partially update a resource|❌ (usually)|❌|Yes|
|`DELETE`|Remove a resource|✅|❌|Rarely|
|`HEAD`|Like `GET` but headers only, no body|✅|✅|No|
|`OPTIONS`|Discover allowed methods/CORS preflight|✅|✅|No|

> [!info] **Safe** vs **Idempotent** — don't confuse them
> 
> - **Safe**: the method doesn't change server state at all (read-only). Only `GET`, `HEAD`, `OPTIONS`.
> - **Idempotent**: calling it once or 100 times **leaves the server in the same state** as calling it once — but it can still _change_ state on the first call. `PUT id=5 {name: "X"}` run 3 times still results in the same final resource, so it's idempotent even though it's not safe.
> - `POST` is neither: each call typically creates a **new** resource (three `POST`s → three new records).
> - `PATCH` is usually **not** idempotent in principle (e.g. "increment counter by 1" applied twice ≠ applied once), though many real-world PATCH implementations are written to behave idempotently anyway.

---

# 3. Headers, Cookies, and Sessions

### Common Request Headers

|Header|Purpose|
|---|---|
|`Host`|Target domain (enables virtual hosting)|
|`Accept`|Media types the client can handle (drives content negotiation)|
|`Authorization`|Credentials (`Bearer <token>`, `Basic <base64>`)|
|`Content-Type`|Media type of the request body|
|`If-None-Match` / `If-Modified-Since`|Conditional requests for caching|
|`Cookie`|Sends previously stored cookies back to the server|

### Common Response Headers

|Header|Purpose|
|---|---|
|`Content-Type`|Media type of the response body|
|`Set-Cookie`|Instructs the client to store a cookie|
|`Cache-Control`|Caching directives (`no-store`, `max-age=3600`)|
|`ETag`|Version identifier for a resource (caching/concurrency)|
|`Location`|Where a newly created resource lives (with `201 Created`)|
|`WWW-Authenticate`|Tells the client how to authenticate (with `401`)|

### Cookies

- Small key-value pairs the **server** asks the **client** to store (via `Set-Cookie`) and automatically resend on subsequent requests to the same domain (via `Cookie`).
- Key attributes:

|Attribute|Meaning|
|---|---|
|`Expires` / `Max-Age`|Lifetime of the cookie|
|`HttpOnly`|Not accessible via JavaScript (`document.cookie`) — mitigates XSS token theft|
|`Secure`|Only sent over HTTPS|
|`SameSite`|`Strict`/`Lax`/`None` — controls cross-site sending, main defense against CSRF|
|`Domain` / `Path`|Scope of where the cookie is sent|

### Sessions

- Since HTTP is stateless, a **session** is a server-side mechanism to remember a user across multiple requests.
- Typical flow: server creates a session object + a unique **session ID**, sends that ID to the client as a cookie (e.g. `JSESSIONID`), and the client echoes it back on every request so the server can look up the associated session data.

---

# 4. Session vs Cookie

||Cookie|Session|
|---|---|---|
|**Storage location**|Client-side (browser)|Server-side (memory, DB, Redis, etc.)|
|**What's actually sent over the wire**|The full data (unless just an ID)|Only a **session ID** — the cookie is the _pointer_, not the data|
|**Size limit**|~4KB per cookie|Effectively unlimited (bounded by server storage)|
|**Security exposure**|Data itself can be read/tampered with client-side unless encrypted/signed|Actual data never leaves the server — only the opaque ID does|
|**Lifetime control**|Set by `Expires`/`Max-Age`; lives even after browser closes if persistent|Typically tied to inactivity timeout; can be invalidated server-side anytime|
|**Scalability**|Stateless from the server's point of view — scales trivially|Needs **session replication** or a shared store (Redis) across multiple server instances|
|**Typical use**|Remembering preferences, tracking, small non-sensitive flags|Authentication state, shopping carts, anything sensitive/large|

> [!important] The relationship, not a rivalry Cookies and sessions aren't competing alternatives — in the classic model, **the session lives on the server, and a cookie is simply the delivery mechanism carrying the session ID back and forth.** You could also store session-like data entirely in a signed/encrypted cookie (stateless sessions, e.g. JWT-in-cookie) — that trades server-side storage for putting the burden of integrity/size on the cookie itself.

---

# 5. HTTP Status Codes (1xx–5xx)

|Range|Category|Common Codes|
|---|---|---|
|**1xx**|Informational|`100 Continue`, `101 Switching Protocols` (used in the WebSocket handshake)|
|**2xx**|Success|`200 OK`, `201 Created`, `202 Accepted`, `204 No Content`|
|**3xx**|Redirection|`301 Moved Permanently`, `302 Found`, `304 Not Modified`, `307 Temporary Redirect`|
|**4xx**|Client Error|`400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `405 Method Not Allowed`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests`|
|**5xx**|Server Error|`500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`|

> [!tip] `401` vs `403` — the classic mix-up
> 
> - **`401 Unauthorized`** — you are **not authenticated** (no/invalid credentials). Should include `WWW-Authenticate`.
> - **`403 Forbidden`** — you **are** authenticated, but you don't have **permission** for this resource/action.

---

# 6. REST Principles — Statelessness & Idempotency 

**REST (REpresentational State Transfer)**, defined by Roy Fielding's dissertation, is an **architectural style** for distributed systems built around a small set of constraints:

|Constraint|Meaning|
|---|---|
|**Client–Server**|Clear separation of concerns; either can evolve independently|
|**Statelessness**|Every request must contain **all** information needed to process it — the server stores **no client session state** between requests|
|**Cacheability**|Responses must define themselves as cacheable or not (`Cache-Control`, `ETag`)|
|**Uniform Interface**|Resources identified by URIs, manipulated via a standard set of methods/representations|
|**Layered System**|Client can't tell if it's talking directly to the server or through intermediaries (proxies, gateways, load balancers)|
|**Code-on-Demand** (optional)|Server can extend client behavior by shipping executable code (e.g. JavaScript)|

### Statelessness — Deep Dive

- Nothing about "who the client is" or "what they did last request" is remembered server-side between calls.
- Any needed context (auth token, pagination cursor, etc.) travels **with every request** — typically as a header (`Authorization: Bearer ...`), not a server-side session.
- **Why it matters**: enables horizontal scaling — any server instance can handle any request, since no instance is holding session state that others lack. This is the direct architectural tension with the classic servlet-session model in [[#4. Session vs Cookie]] — a "RESTful" API and a stateful HTTP session are philosophically at odds; most real systems compromise (stateless REST APIs + a stateful browser session for the web UI, or JWTs to keep even the "session" stateless).

### Idempotency — Deep Dive (recap from [[#2. HTTP Methods]])

- Critical for **safe retries**: if a client times out waiting for a response to a `PUT` and retries, an idempotent endpoint guarantees no duplicate side effects.
- `POST` is the risky one — many APIs add an **idempotency key** header (`Idempotency-Key: <uuid>`) so clients can safely retry non-idempotent creates without risking duplicate resources; the server deduplicates by that key.

---

# 7. REST vs RESTful APIs

||REST|"RESTful API" (common industry usage)|
|---|---|---|
|Definition|The **architectural style** itself — a set of constraints|An API that **claims** to follow REST conventions in practice|
|Strictness|All 6 constraints, including HATEOAS|Usually only _some_ constraints are actually honored|
|Reality in the wild|Rare — true REST-per-Fielding is uncommon|Most "REST APIs" are really just **HTTP + JSON CRUD APIs** using resource-style URLs, but often skip HATEOAS entirely|

> [!important] The uncomfortable truth Fielding himself has publicly complained that most APIs calling themselves "RESTful" are not actually REST, because they omit **hypermedia controls (HATEOAS)** — see next section. In casual industry usage, "RESTful API" essentially just means: _uses HTTP verbs semantically, resource-oriented URLs, and JSON/XML payloads._ This is "good enough" for the vast majority of real systems, but it's technically a looser, more pragmatic definition than Fielding's original one.

---

# 8. Richardson Maturity Model

A scale (by Leonardo Richardson) describing **how RESTful** an API actually is, in four levels:

|Level|Name|Characteristics|
|---|---|---|
|**Level 0**|The Swamp of POX|Single URI endpoint, single HTTP method (usually `POST`), the "protocol" is really just XML/JSON-over-HTTP tunneling (like classic SOAP-over-HTTP)|
|**Level 1**|Resources|Multiple URIs, one per resource (e.g. `/orders/1`, `/customers/5`) — but still often just one HTTP method (`POST`) for everything|
|**Level 2**|HTTP Verbs|Proper use of HTTP methods (`GET`/`POST`/`PUT`/`DELETE`) and status codes to represent actions and outcomes — **this is where the vast majority of "REST APIs" in production actually sit**|
|**Level 3**|Hypermedia Controls (HATEOAS)|Responses include **links** describing available next actions/related resources, so clients can navigate the API dynamically instead of hardcoding URLs — this is Fielding's "true REST"|

```
Level 0 ──► Level 1 ──► Level 2 ──► Level 3
 (POX)      (Resources)  (Verbs)     (HATEOAS)
                                        │
                              Only here is it "fully RESTful"
                              by Fielding's original definition
```

---

# 9. HATEOAS

**HATEOAS = Hypermedia As The Engine Of Application State.**

The idea: API responses shouldn't just return raw data — they should include **links** telling the client what it can do _next_, the same way a web page has clickable links. The client doesn't need to hardcode URL structures; it **discovers** them from the response.

### Example Without HATEOAS

```json
{
  "id": 1,
  "status": "PENDING",
  "total": 49.99
}
```

The client must already "know" (out-of-band, from documentation) that it can `PUT /orders/1/cancel` to cancel this order.

### Example With HATEOAS

```json
{
  "id": 1,
  "status": "PENDING",
  "total": 49.99,
  "_links": {
    "self": { "href": "/orders/1" },
    "cancel": { "href": "/orders/1/cancel", "method": "PUT" },
    "customer": { "href": "/customers/42" }
  }
}
```

Now the client can navigate purely by following links returned in responses — if the server later changes URL structure, well-built HATEOAS clients keep working, because they never hardcoded the path.

### In Spring

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
@GetMapping("/orders/{id}")
public EntityModel<OrderDto> getOrder(@PathVariable Long id) {
    OrderDto order = orderService.findById(id);
    return EntityModel.of(order,
        linkTo(methodOn(OrderApiController.class).getOrder(id)).withSelfRel(),
        linkTo(methodOn(OrderApiController.class).cancelOrder(id)).withRel("cancel"));
}
```

> [!tip] Why most APIs skip it HATEOAS adds real complexity (client and server both need hypermedia-aware code) for a benefit — loose coupling to URL structure — that many teams don't actually need, since frontend and backend are usually deployed and versioned together anyway. This is _why_ most production "REST APIs" stop at Richardson Level 2.

---

# 10. API Versioning & Content Negotiation

### Versioning Strategies

|Strategy|Example|Trade-off|
|---|---|---|
|**URI path**|`/api/v1/orders`|Simple, visible, but "pollutes" the URI (arguably breaks the idea that a URI identifies _a resource_, not _a version of an API_)|
|**Query parameter**|`/api/orders?version=1`|Easy to default, easy to miss/forget|
|**Custom header**|`X-API-Version: 1`|Keeps URIs clean, but less discoverable/less cache-friendly|
|**`Accept` header (media type versioning)**|`Accept: application/vnd.myapp.v1+json`|Most "correct" per content negotiation principles, but least common in practice due to complexity|

### Content Negotiation (recap from [[Spring MVC#12. Content Negotiation]])

- The client states what it wants via `Accept` (response format) and `Content-Type` (request format); the server picks a matching representation, or replies `406 Not Acceptable` if it can't satisfy the request.

---

# 11. SOAP — XML, WSDL

**SOAP (Simple Object Access Protocol)** is ==an XML-based messaging protocol specification used to exchange structured information in the implementation of **strongly-typed, secure web services**==. It establishes a strict, platform-independent communication contract between applications, allowing a Java backend to interact seamlessly with systems written in other languages like .NET or C++.

> It's build on top of other TCP protocols, While it is overwhelmingly used over HTTP/HTTPS, SOAP is actually transport-independent. It can also be transmitted over other protocols like SMTP (email) or JMS (Java Message Service).

### Anatomy of a SOAP Message

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <!-- optional metadata: auth, transaction info -->
  </soap:Header>
  <soap:Body>
    <GetOrderRequest xmlns="http://example.com/orders">
      <OrderId>1</OrderId>
    </GetOrderRequest>
  </soap:Body>
</soap:Envelope>
```

### WSDL (Web Services Description Language)

- An XML document that **formally describes** a SOAP service: available operations, expected input/output message structures, data types, and the endpoint address.
- Acts as a strict, machine-readable **contract** — client tooling can generate strongly-typed client stubs directly from a WSDL file.

### Key SOAP Characteristics

- Protocol-**agnostic** in theory (can run over HTTP, SMTP, JMS), though HTTP dominates in practice.
- Built-in extensibility standards: **WS-Security** (message-level encryption/signing), **WS-ReliableMessaging**, **WS-AtomicTransaction**.
- Strongly typed via **XML Schema (XSD)**.

---

# 12. REST vs SOAP

||REST|SOAP|
|---|---|---|
|Message format|Typically JSON (also XML)|XML only|
|Contract|Informal (OpenAPI/Swagger, optional)|Formal, mandatory (WSDL)|
|Protocol binding|HTTP-native, uses HTTP verbs/status codes|Protocol-agnostic (usually over HTTP, ignores HTTP semantics)|
|Statelessness|Core constraint|Not inherent — can be stateful|
|Built-in security/transactions|No — bring your own (OAuth2, TLS)|Yes — WS-Security, WS-AtomicTransaction standards|
|Performance|Lighter payloads, faster to parse|Heavier due to XML envelope overhead|
|Typical use today|Public APIs, mobile/web backends, microservices|Enterprise/legacy systems, banking, telecom — where strict contracts and built-in transactional guarantees matter|

---

# 13. Servlet Lifecycle

A **Servlet** is the low-level Java class handling HTTP requests inside a servlet container (Tomcat, Jetty, Undertow — see [[#15. Web Server vs Application Server]]). `DispatcherServlet` (from [[Spring MVC]]) is itself just one big servlet.

### Lifecycle Phases

```
Container starts
      │
      ▼
 Servlet class loaded
      │
      ▼
new Servlet() (default constructor)
      │
      ▼
   init(ServletConfig)     ← called ONCE, before serving any request
      │
      ▼
┌─────────────────────────┐
│  service(request, resp) │ ←── called on EVERY incoming request
│   (dispatches internally│       (multiple threads may call this
│    to doGet/doPost/etc.)│        concurrently for the SAME instance)
└─────────────────────────┘
      │
      ▼
   destroy()                ← called ONCE, when container shuts down
                               or servlet is unloaded
```

|Method|Called|Purpose|
|---|---|---|
|`init(ServletConfig)`|Once, at startup (or first request if lazily loaded)|One-time setup — DB connections, resource loading|
|`service(req, res)`|Every request|Dispatches to `doGet`, `doPost`, `doPut`, `doDelete`, etc. based on HTTP method|
|`destroy()`|Once, at shutdown|Cleanup — closing resources, flushing caches|

> [!warning] Servlets are singletons by default A servlet container typically creates **one instance** of each servlet and reuses it across all requests, dispatching each request to a separate **thread**. This means servlet instance fields are **shared state across concurrent requests** — a classic thread-safety pitfall for anyone writing raw servlets (Spring insulates you from this by default, since controllers are typically stateless singletons that don't hold per-request mutable fields).

---

# 14. Filters vs Listeners (vs Interceptors)

### Filters

- Intercept requests/responses **before** they reach a servlet (and after, on the way out) — operate at the **servlet container** level, framework-agnostic.
- Chainable: multiple filters can process the same request in sequence via `FilterChain.doFilter(...)`.

```java
public class AuthFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        // pre-processing
        chain.doFilter(req, res); // continue the chain
        // post-processing
    }
}
```

Typical uses: authentication, logging, compression, CORS, encoding.

### Listeners

- **React to lifecycle events** of the web application/session/request — they don't sit "in the flow" of a request the way filters do; they get **notified** when something happens.

|Listener Interface|Fires On|
|---|---|
|`ServletContextListener`|Application startup (`contextInitialized`) / shutdown (`contextDestroyed`)|
|`HttpSessionListener`|Session created / destroyed|
|`ServletRequestListener`|Request starts / ends|
|`HttpSessionAttributeListener`|Attribute added/removed/replaced in a session|

```java
public class AppStartupListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("Application starting up...");
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("Application shutting down...");
    }
}
```

### The Three-Way Distinction

||Filter|Listener|Interceptor|
|---|---|---|---|
|Triggers on|Every matching request (in the request flow)|Lifecycle **events** (app/session/request start/end)|Requests reaching `DispatcherServlet`|
|Can modify request/response?|Yes|No|Yes (via `ModelAndView`)|
|Framework awareness|None (pure servlet spec)|None (pure servlet spec)|Spring MVC-aware|
|Typical use|Security, logging, compression|Resource init/cleanup on app or session events|Auth per-route, locale setup, logging with handler context|

---

# 15. Web Server vs Application Server

||Web Server|Application Server|
|---|---|---|
|**Primary job**|Serve **static content** (HTML, CSS, JS, images) and handle HTTP protocol details|Run **application/business logic**, often including a full servlet/EJB container|
|**Protocol scope**|HTTP/HTTPS only|HTTP plus often additional protocols (JMS, RMI, CORBA in the Java EE world)|
|**Dynamic content**|Limited — typically via delegation (CGI, reverse-proxying to an app server)|Native — executes servlets, JSPs, EJBs, manages transactions|
|**Examples**|Nginx, Apache HTTP Server, Microsoft IIS|Full Java EE app servers: WildFly, WebLogic, WebSphere, GlassFish|
|**Where does Tomcat fit?**|Tomcat is often called a "**servlet container**" — technically it's a lightweight in-between: it serves HTTP and runs servlets/JSPs, but it does **not** implement the full Java EE spec (no EJB container, no JMS, no distributed transactions) the way WildFly/WebLogic do||

### Typical Production Topology

```
Client
   │
   ▼
Reverse Proxy / Web Server (Nginx)   ← TLS termination, static assets, load balancing
   │
   ▼
Application Server / Servlet Container (embedded Tomcat inside your Spring Boot jar)
   │
   ▼
Your Application Code (Controllers → Services → Repositories)
```

> [!tip] Where Spring Boot fits A Spring Boot app **embeds** a lightweight servlet container (Tomcat/Jetty/Undertow) directly inside its executable jar. You typically still put a real web server (Nginx, or a cloud load balancer) in front of it for TLS termination, static file caching, and load balancing across multiple instances.

---

# 16. WebSockets vs HTTP vs Server-Sent Events (SSE)

||HTTP (request/response)|Server-Sent Events (SSE)|WebSockets|
|---|---|---|---|
|**Direction**|Client → Server → Client (one round trip per request)|**Server → Client only**, one-way stream|**Full duplex** — both directions, simultaneously|
|**Connection**|New connection per request (or reused via `Keep-Alive`, but still request-driven)|Single long-lived HTTP connection, server keeps pushing|Single long-lived connection **upgraded** from HTTP via `101 Switching Protocols`|
|**Protocol**|Plain HTTP|Plain HTTP (`Content-Type: text/event-stream`)|Its own protocol (`ws://`/`wss://`) after the HTTP handshake|
|**Client can push mid-stream?**|No — must initiate a new request|No|Yes|
|**Browser API**|`fetch`/`XMLHttpRequest`|`EventSource`|`WebSocket`|
|**Built-in reconnection**|N/A|✅ Automatic in `EventSource`|❌ Must implement manually|
|**Typical use**|Standard CRUD APIs|Live feeds, notifications, stock tickers, progress updates (server → client only)|Chat apps, multiplayer games, collaborative editing — anything needing low-latency bidirectional messaging|
|**Overhead**|Header overhead per request|Low — one connection, text-based|Lowest per-message overhead once connected|
|**Firewall/proxy friendliness**|Best (it's just HTTP)|Good (it's just HTTP)|Can be blocked by strict proxies/firewalls not expecting protocol upgrades|

### Visual Comparison

```
HTTP:
Client ──request──► Server
Client ◄─response── Server
(repeat per interaction)

SSE:
Client ──single request──► Server
Client ◄──event 1───────── Server
Client ◄──event 2───────── Server
Client ◄──event 3───────── Server   (connection stays open, one direction)

WebSocket:
Client ──HTTP Upgrade────► Server
Client ◄─101 Switching────  Server
Client ◄──message──────►  Server   (either side can send, anytime)
Client ◄──message──────►  Server
```

### In Spring

- **SSE**: return `Flux<ServerSentEvent<T>>` (WebFlux) or use `SseEmitter` in Spring MVC.
- **WebSockets**: `spring-boot-starter-websocket`, `@ServerEndpoint`/`WebSocketHandler`, often paired with **STOMP** for pub/sub messaging semantics over WebSockets.

---

# 17. Common Web Attacks (Web Security 101)

|Attack|What Happens|Primary Defense|
|---|---|---|
|**XSS (Cross-Site Scripting)**|Attacker injects malicious JS into a page viewed by other users (e.g. via an unescaped comment field)|Escape/encode all user-supplied output; `Content-Security-Policy` header; `HttpOnly` cookies|
|**CSRF (Cross-Site Request Forgery)**|A malicious site tricks a logged-in user's browser into submitting a request to your app, using the user's own cookies|CSRF tokens; `SameSite=Strict/Lax` cookies; check `Origin`/`Referer`|
|**SQL Injection**|Unsanitized input is concatenated into a SQL query, letting an attacker manipulate/exfiltrate data|Parameterized queries / prepared statements; ORM usage (JPA); never string-concatenate SQL|
|**Clickjacking**|Your page is embedded in an invisible `<iframe>` on an attacker's site, tricking users into clicking hidden buttons|`X-Frame-Options: DENY`; `Content-Security-Policy: frame-ancestors 'none'`|
|**Session Hijacking**|Attacker steals a valid session ID (via XSS, network sniffing, etc.) and impersonates the user|`HttpOnly`+`Secure` cookies, HTTPS everywhere, short session lifetimes, regenerate session ID after login|
|**Man-in-the-Middle (MITM)**|Attacker intercepts traffic between client and server|TLS/HTTPS everywhere, HSTS header|
|**CORS Misconfiguration**|Overly permissive `Access-Control-Allow-Origin: *` combined with credentials exposes APIs to any origin|Explicit allowed-origins list, never combine wildcard origin with `allowCredentials(true)`|
|**Broken Authentication**|Weak password policies, missing rate limiting, predictable session IDs|Strong hashing (bcrypt/argon2), MFA, rate limiting/lockouts|
|**IDOR (Insecure Direct Object Reference)**|`/api/orders/5` returns someone else's order because the server never checks ownership|Always verify the authenticated user owns/can access the requested resource server-side|
|**DDoS (Distributed Denial of Service)**|Overwhelm the server with traffic to make it unavailable|Rate limiting, CDN/WAF, autoscaling, circuit breakers|
|**Command Injection**|User input is passed unsanitized into a system shell command|Avoid shelling out to OS commands with user input; use safe APIs/allow-lists|

---

# 18. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|Safe method|Never changes server state (`GET`, `HEAD`, `OPTIONS`)|
|Idempotent method|Same result no matter how many times you call it (`GET`, `PUT`, `DELETE`) — `POST` is neither|
|Cookie|Client-side storage; server asks for it via `Set-Cookie`|
|Session|Server-side storage; cookie just carries the session **ID**|
|`401`|Not authenticated|
|REST statelessness|No server-side client state between requests — everything travels with the request|
|Richardson Level 2|Where most "REST APIs" actually live (proper verbs + status codes, no hypermedia)|
|Richardson Level 3 / HATEOAS|Responses include links to available next actions — true REST per Fielding|
|REST vs SOAP|REST = lightweight/HTTP-native/JSON; SOAP = strict XML contract (WSDL) + built-in enterprise features|
|Servlet lifecycle|`init()` once → `service()` per request → `destroy()` once|
|Filter|Servlet-level, sits in the request flow|
|Listener|Reacts to lifecycle **events**, not in the request flow|
|Web server|Static content + HTTP (Nginx, Apache)|
|App server|Runs your app logic, servlet container or full Java EE stack|
|WebSocket|Full-duplex, persistent, needs protocol upgrade|
|SSE|Server → client only, plain HTTP, auto-reconnect|
|XSS|Inject script into a page other users view|
|CSRF|Forge a request using a victim's existing session/cookies|
|IDOR|Forgetting to check resource ownership server-side|
