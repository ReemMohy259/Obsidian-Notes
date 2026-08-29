---
share_link: https://share.note.sx/9yhnqqei#ZvKaPVuxR+1T1WzGU+bUPg
share_updated: 2026-08-03T13:31:07+03:00
---
## Table of Contents

- [[#1. What Are Microservices?]]
- [[#2. Monolith vs Microservices Trade-offs]]
- [[#3. Service Discovery (Eureka)]]
- [[#4. API Gateway]]
- [[#5. Circuit Breaker (Resilience4j)]]
- [[#6. Config Server]]
- [[#7. Distributed Tracing & Centralized Logging]]
- [[#8. Messaging: RabbitMQ vs Kafka]]
- [[#9. SAGA Pattern]]
- [[#10. Event-Driven Architecture]]
- [[#11. Sync vs Async Service Communication]]
- [[#12. Database per Service]]
- [[#13. CAP Theorem & Resilience Patterns]]
- [[#14. Additional Essentials]]
- [[#15. Quick Revision Cheat Sheet]]

---

## 1. What Are Microservices?

**Microservices** is an architectural style where an application is composed of small, **independently deployable** services, each owning a specific business capability, communicating over a network.

```
┌─────────┐   ┌───────────┐   ┌───────────┐   ┌────────────┐
│ Orders  │   │ Inventory │   │ Customers │   │  Payments  │
│ Service │   │  Service  │   │  Service  │   │  Service   │
├─────────┤   ├───────────┤   ├───────────┤   ├────────────┤
│ own DB  │   │  own DB   │   │  own DB   │   │  own DB    │
└─────────┘   └───────────┘   └───────────┘   └────────────┘
     ▲              ▲               ▲                ▲
     └──────────────┴────────┬──────┴────────────────┘
                             │
                       API Gateway
                             │
                          Clients
```

### Defining Characteristics

- **Independently deployable** — each service ships on its own schedule.
- **Independently scalable** — scale only the service under load, not the whole system.
- **Owns its own data** — see [[#12. Database per Service]].
- **Organized around business capability**, not technical layers.
- **Polyglot-friendly** — different services _can_ use different languages/stacks (though most orgs limit this in practice for operational sanity).
- **Decentralized governance** — teams own their service end-to-end (a strong link to **Conway's Law**: system architecture tends to mirror organizational communication structure, so team boundaries and service boundaries should generally align).

---

## 2. Monolith vs Microservices Trade-offs

||Monolith|Microservices|
|---|---|---|
|Deployment|One unit, all-or-nothing|Independent per service|
|Scaling|Whole app scales together|Scale exactly the bottlenecked service|
|Data consistency|Easy — single DB, ACID transactions|Hard — distributed data, eventual consistency, [[#9. SAGA Pattern]]
|Operational complexity|Low|High — needs [[#3. Service Discovery (Eureka)]]
|Team autonomy|Low — shared codebase, shared release train|High — teams own services end-to-end|
|Failure isolation|Poor — one bug can crash everything|Better — one service failing (ideally) doesn't take down others, if resilience patterns are used correctly|
|Latency|In-process calls (nanoseconds)|Network calls (milliseconds+), serialization overhead|
|Testing|Simpler — everything in-process|Harder — needs contract testing, service virtualization, integration environments|
|Best for|Small-medium teams, early-stage, unclear domain|Large orgs, well-understood domains, need for independent scaling/deployment|

> [!danger] The Distributed Monolith anti-pattern If services can't be deployed independently because they're all tightly coupled (shared database, synchronous chains of calls, lock-step versioning), you've paid microservices' operational cost while getting **none** of the benefits. boundaries must come from real domain seams, not arbitrary splits.

---

## 3. Service Discovery (Eureka)

### The Problem

In a dynamic environment (containers restarting, autoscaling, IPs changing), services can't hardcode each other's network locations — "where is the Inventory Service right now?" changes constantly.

### The Solution — A Service Registry

Each service instance **registers itself** with a central registry on startup and sends periodic **heartbeats**. Other services **query the registry** to discover available instances, instead of using hardcoded URLs.

```
      ┌─────────────────────┐
      │  Eureka Server      │  ← the registry
      │  (Service Registry) │
      └─────────────────────┘
        ▲       ▲       ▲
   register  register  register (+ heartbeats)
        │       │       │
  ┌─────────┐┌─────────┐┌─────────┐
  │ Orders  ││Orders   ││Inventory│
  │ instance││instance ││ instance│
  │   #1    ││  #2     ││   #1    │
  └─────────┘└─────────┘└─────────┘
    ▲
    │ "where is Inventory Service?"
┌────────┐
│Payments│(client-side discovery — asks Eureka, then calls the instance directly)
│ Service│
└────────┘
```

### Netflix Eureka (Spring Cloud Netflix)

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@EnableEurekaServer
@SpringBootApplication
public class DiscoveryServerApp { ... }
```

```xml
<!-- Eureka Client (every microservice) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
spring:
  application:
    name: order-service
```

Calling another service by its registered name (via `RestTemplate`/`WebClient` with `@LoadBalanced`, or Spring Cloud's `DiscoveryClient`) resolves to an actual instance URL at call time.

### Client-Side vs Server-Side Discovery

||Client-Side Discovery|Server-Side Discovery|
|---|---|---|
|Who queries the registry|The calling service itself|A load balancer/gateway in front of the services|
|Example|Netflix Eureka + Ribbon/Spring Cloud LoadBalancer|Kubernetes Services + `kube-proxy`, AWS ALB|
|Trade-off|More logic in each client|Simpler clients, but relies on an external component|

> [!tip] Kubernetes changes the calculus In a Kubernetes environment, **DNS-based service discovery** (built into K8s Services) often replaces Eureka entirely — this is why Eureka is more associated with "classic" Spring Cloud Netflix stacks than modern cloud-native/K8s deployments. Know both; know which one your target environment actually uses.

---

## 4. API Gateway

### The Problem

Without a gateway, every client (web, mobile) must know the network location of **every** microservice, handle cross-cutting concerns (auth, rate limiting, logging) in every client, and deal with N different services' worth of CORS/TLS/versioning individually.

### The Solution

A single entry point that sits in front of all services, providing:

- **Routing** — forwards each request to the correct backend service.
- **Authentication/Authorization** — centralize token validation instead of duplicating it per service.
- **Rate limiting & throttling**.
- **Request/response transformation**.
- **Load balancing** across service instances.
- **Cross-cutting resilience**: can integrate circuit breakers, retries, timeouts at the edge.
- **API composition** — sometimes aggregates results from multiple services into one response for the client (though this can also be its own **Backend-for-Frontend / BFF** layer instead).

```
                     ┌────────────────────┐
Client ────request──►│    API Gateway     │
                     │  auth, rate limit, │
                     │  routing, logging  │
                     └────────────────────┘
                        │       │       │
                        ▼       ▼       ▼
                    Orders  Inventory Payments
                    Service  Service   Service
```

### Spring Cloud Gateway (Modern Choice)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service  # lb:// = load-balanced via service discovery
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
        - id: inventory-service
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
```

### Other Common Gateway Options

|Gateway|Notes|
|---|---|
|**Spring Cloud Gateway**|Reactive (WebFlux-based), the modern Spring-ecosystem default|
|**Netflix Zuul**|Older, largely superseded by Spring Cloud Gateway|
|**Kong / NGINX / Envoy**|Language/framework-agnostic, common in polyglot or K8s environments|
|**AWS API Gateway**|Managed, cloud-native option|

> [!warning] Don't let the Gateway become a bottleneck or a "smart" monolith The Gateway should stay focused on **cross-cutting, generic** concerns (auth, routing, rate limiting). If business logic starts creeping into gateway filters, you've recreated a monolith at the edge — and a single point of failure with it (mitigate with redundancy/horizontal scaling of the gateway itself).

---

## 5. Circuit Breaker (Resilience4j)

### The Problem

In a synchronous call chain (`Order Service → Payment Service → Bank API`), if the Payment Service is slow or down, callers pile up waiting on it — threads block, resources exhaust, and the failure **cascades** upstream, potentially taking down the entire system even though only one downstream dependency actually failed.

### The Solution — Circuit Breaker Pattern

Modeled after an electrical circuit breaker: monitor calls to a dependency, and if failures exceed a threshold, **stop calling it entirely for a while** ("open" the circuit), failing fast instead of waiting on a doomed call — giving the failing service room to recover instead of being hammered further.

### The Three States

```
     failures exceed threshold
CLOSED ───────────────────────► OPEN
  ▲                               │
  │                               │ wait duration elapses
  │      success rate OK          ▼
  └──────────────────────── HALF_OPEN
                    │
                    │ a trial request fails
                    └──────────► OPEN
```

|State|Behavior|
|---|---|
|**CLOSED**|Normal operation — calls pass through, failures are counted|
|**OPEN**|Calls **fail immediately** without even attempting the network call (often returning a fallback)|
|**HALF_OPEN**|After a wait period, allows a limited number of **trial calls** through to test if the dependency has recovered|

### Resilience4j (Modern Choice — Replaced Netflix Hystrix)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

```java
@Service
public class PaymentClient {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResult charge(Order order) {
        return restClient.post("/charge", order, PaymentResult.class);
    }

    public PaymentResult paymentFallback(Order order, Throwable t) {
        return PaymentResult.pending("Payment service unavailable, queued for retry");
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-size: 10
        failure-rate-threshold: 50     # % of calls failing to trip the breaker
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
```

### Related Resilience Patterns (Resilience4j Also Provides)

|Pattern|Purpose|
|---|---|
|**Retry**|Automatically retries a failed call (with backoff) — useful for transient failures|
|**Rate Limiter**|Caps the number of calls allowed in a time window|
|**Bulkhead**|Isolates resources (thread pools) per dependency, so one slow dependency can't exhaust threads needed by others|
|**Time Limiter**|Enforces a timeout on a call, failing fast rather than waiting indefinitely|

> [!tip] Combine, don't choose one A production-grade call to a downstream service typically wraps **Time Limiter → Circuit Breaker → Retry → Bulkhead**, layered together, rather than relying on just one pattern.

---

## 6. Config Server

### The Problem

Each microservice has its own configuration (DB URLs, feature flags, third-party API keys), which changes per-environment (dev/staging/prod). Managing dozens of `application.yml` files scattered across services, and redeploying just to change a config value, doesn't scale.

### The Solution — Centralized Configuration

A dedicated **Config Server** serves configuration to all services from a single source of truth (commonly a Git repo), letting you:

- Change configuration **without rebuilding/redeploying** the service (when combined with refresh mechanisms).
- Version-control configuration changes just like code.
- Manage per-environment and per-service config from one place.

```
   Git Repo (config files per service/environment)
              │
              ▼
      ┌────────────────┐
      │  Config Server │
      └────────────────┘
        ▲       ▲       ▲
   fetch config at startup (or on refresh)
        │       │       │
  ┌─────────┐┌─────────┐┌─────────┐
  │ Orders  ││Inventory││Payments │
  │ Service ││ Service ││ Service │
  └─────────┘└─────────┘└─────────┘
```

### Spring Cloud Config

```xml
<!-- Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApp { ... }
```

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
```

```yaml
# Each microservice's bootstrap config
spring:
  application:
    name: order-service
  config:
    import: "optional:configserver:http://localhost:8888"
```

### Refreshing Config Without a Restart

```java
@RefreshScope       // beans annotated here re-read config when refreshed
@Component
public class FeatureFlags { ... }
```

```bash
curl -X POST http://order-service/actuator/refresh
```

For refreshing **many** services at once, Spring Cloud Bus (backed by RabbitMQ/Kafka) broadcasts a refresh event to all instances simultaneously instead of hitting each one individually.

---

## 7. Distributed Tracing & Centralized Logging

### The Problem

A single user request in a microservices system might touch 5+ services. When something goes wrong, you need to **reconstruct the full journey** of that request across service boundaries — impossible if each service's logs live only on its own machine with no shared context.

### Distributed Tracing

- Each request is tagged with a **Trace ID** (shared across every service it touches) and each hop within that trace gets its own **Span ID**.
- As the request flows `Gateway → Order Service → Payment Service → Bank API`, every service propagates the same Trace ID forward (typically via HTTP headers), letting tooling reconstruct the entire call tree.

```
Trace ID: abc-123
├─ Span: Gateway           (2ms)
│   └─ Span: Order Service  (45ms)
│       └─ Span: Payment Service (38ms)
│           └─ Span: Bank API      (30ms)
```

**Common Tools:**

|Tool|Role|
|---|---|
|**Micrometer Tracing** (formerly Spring Cloud Sleuth)|Instruments Spring apps to generate/propagate trace & span IDs|
|**Zipkin**|Collects and visualizes distributed traces|
|**Jaeger**|Alternative to Zipkin, CNCF project, popular in Kubernetes/cloud-native stacks|
|**OpenTelemetry**|Vendor-neutral standard/API for traces, metrics, and logs — increasingly the common instrumentation layer underneath tools like Zipkin/Jaeger|

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

### Centralized Logging

- Since logs are scattered across many service instances/containers, ship them all to a **central aggregator** rather than SSH-ing into individual machines.

```
Service A logs ──┐
Service B logs ──┼──► Log Shipper (Filebeat/Fluentd) ──► Central Store ──► Search/Visualize
Service C logs ──┘                                      (Elasticsearch)     (Kibana)
```

**Common Stack: ELK / EFK**

|Component|Role|
|---|---|
|**Elasticsearch**|Stores and indexes log data for fast search|
|**Logstash** / **Fluentd**|Collects, parses, and ships logs|
|**Kibana**|Visualization/dashboarding UI|
|**Grafana Loki**|Lightweight alternative log aggregation stack, often paired with **Grafana** + **Prometheus** for metrics|

> [!tip] Correlate logs with traces Best practice: include the **Trace ID** in every log line. Then, from a trace in Zipkin/Jaeger showing a slow/failing span, you can jump straight to the exact log lines from that specific request across every service it touched — instead of grepping blindly.

---

## 8. Messaging: RabbitMQ vs Kafka

Both are message brokers enabling **asynchronous** communication between services (see [[#11. Sync vs Async Service Communication]]), but they're built for different use cases.

||RabbitMQ|Kafka|
|---|---|---|
|**Model**|Traditional **message broker** — smart broker, dumb consumer|Distributed **event streaming log** — dumb broker, smart consumer|
|**Message retention**|Removed once consumed/acknowledged (by default)|Retained for a configurable period **regardless of consumption** — consumers can re-read history|
|**Ordering guarantee**|Per-queue|Per-**partition** (strong ordering within a partition)|
|**Throughput**|High, but generally lower than Kafka|Extremely high — built for massive event streams|
|**Routing flexibility**|Rich routing (direct, topic, fanout, headers exchanges)|Simpler — topic + partition based|
|**Protocol**|AMQP (also STOMP, MQTT)|Custom binary protocol over TCP|
|**Best for**|Task queues, RPC-style messaging, complex routing needs, lower-latency point-to-point work distribution|Event sourcing, event-driven architectures, log aggregation, streaming analytics, replay-able event history|
|**Consumer model**|Broker pushes/distributes messages to consumers, tracks acknowledgment|Consumers pull and track their own **offset** (position) in the log|

```
RabbitMQ (Queue model):
Producer ──► Exchange ──► Queue ──► Consumer (message removed once acked)

Kafka (Log model):
Producer ──► Topic (Partition 0, 1, 2...) ──► durable, ordered log
                    ▲                    ▲
             Consumer Group A      Consumer Group B
             (independent offset)  (independent offset, can re-read same events)
```

### In Spring

```xml
<!-- RabbitMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```java
@RabbitListener(queues = "order.created")
public void handleOrderCreated(OrderCreatedEvent event) { ... }
```

```xml
<!-- Kafka -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```java
@KafkaListener(topics = "order-events", groupId = "inventory-service")
public void handleOrderEvent(OrderEvent event) { ... }
```

---

## 9. SAGA Pattern

### The Problem

In a monolith, a multi-step business transaction (place order → reserve inventory → charge payment → confirm shipment) is one ACID database transaction — all-or-nothing. In microservices, each step lives in a **different service with its own database** — you can't wrap them in a single traditional transaction (see [[#12. Database per Service]] and [[#13. CAP Theorem & Resilience Patterns]]).

### The Solution — SAGA

A **SAGA** is a sequence of **local transactions**, one per service, where each step publishes an event/message that triggers the next step. If a step fails, previously completed steps are undone via **compensating transactions** — the SAGA equivalent of a rollback.

### Two Coordination Styles

**1. Choreography** — no central coordinator; each service reacts to events from others.

```
Order Service ──OrderCreated──► Inventory Service ──StockReserved──► Payment Service
     ▲                                                                     │
     └───────────────────── PaymentFailed (compensate!) ───────────────────┘
                                        │
                              Inventory Service releases stock
                                        │
                              Order Service cancels order
```

- ✅ Simple for a small number of steps, no single point of failure.
- ❌ Hard to see/reason about the overall flow as it grows (logic is smeared across every participant); risk of cyclic dependencies between services.

**2. Orchestration** — a central orchestrator explicitly tells each service what to do, step by step.

```
                    ┌───────────────────┐
                    │ SAGA Orchestrator │
                    └───────────────────┘
                     │       │       │
             "reserve"  "charge"  "ship"
                     ▼       ▼       ▼
              Inventory  Payment  Shipping
               Service   Service   Service
```

- ✅ Centralized, explicit, easier to visualize/debug/monitor the whole transaction.
- ❌ The orchestrator becomes a critical component, and can accumulate business logic that arguably belongs in the services themselves.

### Compensating Transactions Example

```
Step 1: Reserve inventory       → Compensate: Release inventory
Step 2: Charge payment          → Compensate: Refund payment
Step 3: Confirm shipment        → Compensate: Cancel shipment
```

If Step 3 fails, run compensations for Steps 2 and 1, in reverse order.

> [!warning] SAGAs give up atomicity for availability There's no instant of true global consistency — for a window of time, the system is in an **intermediate state** (e.g. inventory reserved, but payment not yet confirmed). This is the practical, code-level manifestation of choosing **eventual consistency** — directly tied to [[#13. CAP Theorem & Resilience Patterns]].

---

## 10. Event-Driven Architecture

### Core Idea

Services communicate by **publishing events** ("something happened") rather than directly calling each other ("do this now"). Interested services **subscribe** and react independently.

```
Order Service ──publishes──► OrderPlaced event
                                    │
                    ┌───────────────┼────────────────┐
                    ▼               ▼                ▼
          Inventory Service   Notification Service   Analytics Service
          (reserves stock)    (sends confirmation)    (updates dashboard)
```

### Benefits

✅ **Loose coupling** — the publisher doesn't know (or care) who's listening. 
✅ **Extensibility** — adding a new subscriber requires **zero changes** to the publisher. 
✅ Naturally supports **async** communication ([[#11. Sync vs Async Service Communication]]) and drives patterns like [[#9. SAGA Pattern|SAGA choreography]].

### Trade-offs

❌ Harder to trace a single business flow across many decoupled subscribers (mitigated by [[#7. Distributed Tracing & Centralized Logging|distributed tracing]]). 
❌ Eventual consistency is inherent — subscribers react _after_ the fact, not instantly. 
❌ Event schema evolution needs careful versioning discipline — many independent consumers depend on a shared event contract.

### Event Notification vs Event-Carried State Transfer

|Style|Event Contains|Consumer Must...|
|---|---|---|
|**Event Notification**|Minimal info (e.g. just an ID: "Order #5 changed")|Call back to the source service to fetch full details|
|**Event-Carried State Transfer**|The **full relevant state** (e.g. the whole order payload)|Can act entirely from the event itself, no callback needed — but risks staleness/duplication of data across services|

### Event Sourcing (Related, Not Identical)

- A more specific pattern: instead of storing just the **current state** of an entity, store the **full sequence of events** that led to it, and derive current state by replaying them.
- Gives a complete audit trail "for free" and enables rebuilding state at any point in time — but adds real complexity (event schema versioning, snapshotting for performance) and is only justified when audit/replay/temporal-query needs are strong.

---

## 11. Sync vs Async Service Communication

||Synchronous|Asynchronous|
|---|---|---|
|**Mechanism**|Direct call, caller **waits** for a response (REST/HTTP, gRPC)|Caller sends a message/event and **moves on**; response (if any) arrives later, if at all|
|**Coupling**|Tighter — caller needs the callee to be **up and responsive right now**|Looser — broker/queue buffers messages even if the consumer is temporarily down|
|**Failure impact**|Failure/slowness propagates immediately up the call chain (needs [[#5. Circuit Breaker (Resilience4j)|circuit breakers]])|
|**Latency perception**|Client waits for the full round trip|Client can respond immediately ("request accepted") while work happens in the background|
|**Complexity**|Simpler to reason about — request/response is linear|Harder to reason about — need to think in terms of eventual consistency, retries, idempotent consumers, dead-letter queues|
|**Typical tech**|REST/HTTP, gRPC, GraphQL|RabbitMQ, Kafka, SQS|
|**Best for**|Queries needing an immediate answer (e.g. "get my order status")|Commands/workflows that can tolerate a delay (e.g. "process this order," "send this email")|

```
Synchronous:
Client ──request──► Service A ──request──► Service B
Client ◄─response── Service A ◄─response── Service B
(Client is blocked the whole time, and A is coupled to B's uptime)

Asynchronous:
Client ──request──► Service A ──publish event──► [Queue/Topic]
Client ◄─"202 Accepted"── Service A
                                          Service B consumes whenever it's ready
```

> [!tip] Most real systems mix both Use **synchronous** calls for reads/queries where the client genuinely needs an immediate answer, and **asynchronous** messaging for writes/commands and cross-service side effects that can tolerate eventual consistency. This split underlies both [[#9. SAGA Pattern|SAGA]] and general [[#10. Event-Driven Architecture|event-driven design]].

---

## 12. Database per Service

### The Principle

Each microservice **owns its own database**, and no other service is allowed to access it directly — all access goes through the owning service's API.

```
┌─────────┐        ┌───────────┐        ┌───────────┐
│ Orders  │        │ Inventory │        │ Customers │
│ Service │        │  Service  │        │  Service  │
└────┬────┘        └─────┬─────┘        └─────┬─────┘
     │                   │                    │
     ▼                   ▼                    ▼
┌─────────┐        ┌────────────┐        ┌────────────┐
│Orders DB│        │Inventory DB│        │Customers DB│
└─────────┘        └────────────┘        └────────────┘
     ✗ Inventory Service directly querying Orders DB — FORBIDDEN
```

### Why

- Keeps services **independently deployable** — a schema change in one service can't silently break another that was querying its tables directly.
- Enforces the Bounded Context boundary at the data layer, not just the code layer.
- Enables **polyglot persistence** — Orders might use PostgreSQL, Inventory might use a specialized time-series DB, Search might use Elasticsearch — whatever fits each service's access patterns.

### The Cost: No More Cross-Service JOINs or Transactions

This is the single biggest mental shift coming from monolith development:

- **No SQL JOIN across services' tables** — if you need combined data, either call the other service's API and combine in memory, or maintain a **read-optimized local copy** (see below).
- **No multi-service ACID transactions** — this is _exactly_ why [[#9. SAGA Pattern|SAGA]] exists.

### Handling Cross-Service Queries

|Pattern|Approach|
|---|---|
|**API Composition**|Call each relevant service's API and join the results in the calling layer (e.g. in a BFF or the API Gateway)|
|**CQRS + Read Replicas**|Maintain a separate, denormalized read model built by consuming events from the owning services — trades storage/complexity for fast, join-free reads|

---

## 13. CAP Theorem & Resilience Patterns

### CAP Theorem

In a **distributed system**, when a network partition occurs, you can only guarantee **two** of these three simultaneously:

|Property|Meaning|
|---|---|
|**Consistency (C)**|Every read receives the most recent write (or an error) — all nodes see the same data at the same time|
|**Availability (A)**|Every request receives a (non-error) response, even if some nodes are down|
|**Partition Tolerance (P)**|The system continues operating despite network partitions (nodes unable to communicate)|

> [!important] P is not optional in real distributed systems Network partitions **will** happen — the real choice CAP forces on you is between **C and A** _during_ a partition. This is why CAP is more usefully framed as: "when a partition occurs, do you prioritize Consistency or Availability?"

```
        CAP Theorem — pick 2 of 3
              (P is a given)

    CP: Consistency + Partition Tolerance
        → some nodes return errors/timeouts rather than stale data
        → e.g. traditional RDBMS clusters with strong consistency, ZooKeeper

    AP: Availability + Partition Tolerance
        → all nodes keep responding, but some may return stale data temporarily
        → e.g. Cassandra, DynamoDB (eventually consistent mode), most microservices systems
```

### Where This Shows Up in Microservices

- Choosing **AP** (favor availability) is the practical default for most microservices systems — hence the pervasiveness of **eventual consistency**, [[#9. SAGA Pattern|SAGA]] (instead of distributed ACID transactions), and asynchronous, event-driven communication.
- Choosing **CP** matters more for specific components needing strong consistency guarantees (e.g. a service managing financial ledger balances, or a leader-election/coordination service like ZooKeeper/etcd).

### Resilience Patterns Recap (tie back to [[#5. Circuit Breaker (Resilience4j)]])

|Pattern|Defends Against|
|---|---|
|**Circuit Breaker**|Cascading failures from a struggling downstream dependency|
|**Retry (with backoff)**|Transient/temporary failures|
|**Timeout**|Indefinitely hanging calls|
|**Bulkhead**|One dependency exhausting shared resources (threads/connections) needed by others|
|**Rate Limiter**|Being overwhelmed by traffic, or overwhelming a downstream dependency|
|**Fallback**|Providing a degraded-but-functional response when a dependency is unavailable, instead of a hard failure|
|**Idempotent Consumers**|Safely handling duplicate message delivery (a fact of life in most messaging systems — most guarantee **at-least-once**, not exactly-once, delivery)|

---

## 14. Additional Essentials

A few things every comprehensive microservices treatment should include beyond your checklist:

### Deployment: Containers & Orchestration

- Microservices are almost always deployed as **containers** (Docker) managed by an **orchestrator** (Kubernetes) — providing scheduling, self-healing (restart crashed instances), scaling, and often service discovery/config out of the box (potentially replacing Eureka/Config Server with K8s-native mechanisms).

### API Contracts & Versioning

- **Contract testing** (e.g. Pact) verifies that a service's API still satisfies what its consumers expect, **without** spinning up the actual consumer/provider together — critical when dozens of teams depend on each other's APIs independently.
- Version APIs deliberately (see [[Web-Technologies-Notes#10. API Versioning & Content Negotiation]]) since you can't force every consumer to upgrade in lock-step.

### Observability — The Three Pillars

|Pillar|Answers|Tooling|
|---|---|---|
|**Logs**|What happened, in detail, at a point in time?|ELK/EFK, Loki|
|**Metrics**|What's the aggregate health/behavior over time?|Micrometer, Prometheus, Grafana|
|**Traces**|How did this specific request flow across services?|Zipkin, Jaeger, OpenTelemetry|

### Security Between Services

- **mTLS (mutual TLS)** for service-to-service authentication within the mesh.
- **OAuth2/JWT propagation** — passing the authenticated user's identity/claims through the call chain, often validated once at the [[#4. API Gateway|Gateway]] and re-verified (or trusted via a service mesh) downstream.
- **Service Mesh** (Istio, Linkerd) — a dedicated infrastructure layer handling mTLS, retries, circuit breaking, and observability at the network level, **outside** application code — an alternative to embedding all of Resilience4j/tracing libraries into every service.

---

## 15. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|Microservices|Independently deployable services, each owning a business capability + its own data|
|Distributed Monolith|Services split without real boundaries — worst of both worlds|
|Service Discovery (Eureka)|Services register + query a central registry instead of hardcoding locations|
|API Gateway|Single entry point: routing, auth, rate limiting, cross-cutting concerns|
|Circuit Breaker|CLOSED → OPEN → HALF_OPEN; fail fast instead of cascading failures|
|Config Server|Centralized, version-controlled configuration, refreshable without redeploy|
|Distributed Tracing|Trace ID + Span ID reconstruct a request's path across services|
|Centralized Logging|Ship all service logs to one searchable store (ELK/EFK)|
|RabbitMQ|Smart broker, queue-based, great for routing/task distribution|
|Kafka|Durable event log, great for streaming/replay/high throughput|
|SAGA|Sequence of local transactions + compensating transactions instead of distributed ACID|
|Choreography vs Orchestration|Event-reactive (decentralized) vs explicit coordinator (centralized)|
|Event-Driven Architecture|Publish events, subscribers react — loose coupling, eventual consistency|
|Sync communication|Direct call, caller waits, tighter coupling|
|Async communication|Message/event, caller moves on, looser coupling|
|Database per Service|No cross-service JOINs/transactions — API composition or CQRS instead|
|CAP Theorem|Partition happens → choose Consistency or Availability|
|Most microservices choose|AP (availability) + eventual consistency|
|Resilience patterns|Circuit Breaker, Retry, Timeout, Bulkhead, Rate Limiter, Fallback|
