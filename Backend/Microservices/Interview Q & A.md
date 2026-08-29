---
share_link: https://share.note.sx/pwbhsym4#plgHFcFZAeaxX6LCGLjbOQ
share_updated: 2026-08-01T22:24:05+03:00
---
## Fundamentals

**1. What is a microservices architecture?** An architectural style where an application is built as a collection of small, independent services, each responsible for a specific business capability, communicating over the network (usually HTTP/REST or messaging). Each service can be developed, deployed, and scaled independently.

**2. What's the difference between a monolith and microservices?**

- **Monolith**: a single deployable unit containing all functionality — one codebase, one database, one deployment.
- **Microservices**: functionality split into multiple independently deployable services, each often with its own database, communicating over the network.

**3. What are the main advantages of microservices?**

- Independent deployment — teams can ship one service without redeploying the whole system
- Independent scaling — scale only the services under heavy load
- Technology flexibility — each service can use a different language/stack if needed
- Fault isolation — a failure in one service doesn't necessarily bring down the whole system
- Smaller, more focused codebases — easier for teams to own and understand

**4. What are the main downsides/challenges of microservices?**

- Increased operational complexity — many services to deploy, monitor, and manage
- Network reliability becomes a concern — calls that used to be in-process method calls are now network calls that can fail or be slow
- Data consistency is harder — no single database/transaction spanning everything
- Distributed debugging/tracing is harder — a single user request may span multiple services
- Testing is more complex — need integration/contract tests, not just unit tests

**5. When would you NOT recommend microservices?** For small applications, small teams, or early-stage products where the added operational overhead outweighs the benefits. Many experienced engineers recommend starting with a well-structured **monolith** and splitting into microservices later, once real scaling or team-boundary pain points appear ("monolith-first" approach).

**6. What is a "distributed monolith," and why is it a problem?** A system that's technically split into multiple deployable services, but they're so tightly coupled (shared database, synchronous chains of calls, deploy-together dependencies) that it has all the downsides of microservices (network overhead, complexity) without the actual benefits (independent deployability, fault isolation).

---

## Service Design & Communication

**7. How do you decide service boundaries in a microservices system?** Typically around **business capabilities** or **bounded contexts** (a concept from Domain-Driven Design) — e.g., "Order Service," "Payment Service," "Inventory Service" — rather than splitting by technical layer (like "all controllers" vs "all repositories"). Each service should own its data and have a clear, cohesive responsibility.

**8. What is Domain-Driven Design (DDD), and how does it relate to microservices?** An approach to software design that models the software around the real business domain, using concepts like **bounded contexts** (clear boundaries where a specific model/language applies) and **aggregates**. Microservices architectures often align service boundaries with DDD bounded contexts, since each tends to represent a natural, independent unit of business logic and data.

**9. What's the difference between synchronous and asynchronous communication between services?**

- **Synchronous** (e.g., REST/HTTP, gRPC): the caller waits for a direct response before continuing. Simpler to reason about, but couples the caller to the callee's availability and latency.
- **Asynchronous** (e.g., message queues, event streaming — Kafka, RabbitMQ): the caller sends a message/event and continues without waiting for a response. More resilient to temporary failures and better for decoupling, but harder to reason about and debug.

**10. What is REST, and why is it commonly used for microservice communication?** An architectural style for designing HTTP APIs around resources (URLs) and standard HTTP verbs (GET, POST, PUT, DELETE). It's widely used because it's simple, human-readable, uses standard HTTP tooling, and most languages/frameworks support it out of the box.

**11. What is gRPC, and when might you choose it over REST?** A high-performance RPC framework using HTTP/2 and Protocol Buffers (a binary serialization format) instead of JSON. You'd choose it for internal service-to-service communication where performance/latency matters more than human-readability — it's faster and produces smaller payloads than REST/JSON, but is less convenient for browser clients or quick debugging.

**12. What's the difference between orchestration and choreography in microservices?**

- **Orchestration**: a central coordinator (e.g., an orchestrator service or workflow engine) explicitly tells each service what to do and in what order — easier to understand and monitor, but introduces a central point of coupling.
- **Choreography**: each service reacts to events independently (e.g., via a message broker) without a central coordinator — more decoupled, but harder to trace the overall flow of a business process.

**13. What is an API Gateway, and why is it used?** A single entry point that sits in front of your microservices, routing external client requests to the appropriate backend service. It commonly also handles cross-cutting concerns like authentication, rate limiting, request logging, and SSL termination — so individual services don't need to duplicate that logic. In the Spring ecosystem, **Spring Cloud Gateway** is the standard choice.

**14. What is the Backend for Frontend (BFF) pattern?** A variation of the API Gateway pattern where you create a **separate gateway per client type** (e.g., one BFF for the mobile app, one for the web app) — each tailored to that client's specific data and formatting needs, rather than one generic gateway serving everyone.

---

## Service Discovery & Load Balancing

**15. What problem does service discovery solve?** In a dynamic environment (containers/instances scaling up and down, IPs changing), services can't hardcode each other's network addresses. Service discovery lets services **register themselves** on startup and **look up** other services by logical name at runtime, rather than by fixed IP/port.

**16. What is Eureka, and what's its role?** Netflix's service registry, integrated into Spring Cloud as **Spring Cloud Netflix Eureka**. Services register themselves with Eureka on startup and periodically send heartbeats; other services query Eureka to find available instances of a service by name instead of hardcoding URLs. It's the standard choice for non-Kubernetes Spring Cloud deployments — in Kubernetes environments, Kubernetes's own built-in service discovery is often used instead.

**17. What is client-side load balancing, and how is it done in Spring Cloud?** Instead of routing all requests through a central load balancer, the calling service itself fetches the list of available instances from the service registry and picks one (e.g., round-robin). In Spring Cloud, this is done with **Spring Cloud LoadBalancer** (Netflix Ribbon's now-deprecated predecessor), typically by annotating a `RestClient`/`WebClient`/`RestTemplate` bean with `@LoadBalanced` and calling it using the logical service name in the URL (e.g., `http://order-service/api/orders`).

**18. What is OpenFeign, and why is it used?** A declarative HTTP client — instead of manually building REST calls, you define a Java interface with annotations (`@GetMapping`, `@PostMapping`, etc.), and Spring generates the implementation at runtime. It integrates with service discovery and load balancing so you can call other services by their logical name with minimal boilerplate.

```java
@FeignClient(name = "order-service")
public interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id);
}
```

---

## Resilience & Fault Tolerance

**19. Why is resilience especially important in microservices?** Because services communicate over the network, any downstream service can be slow, unavailable, or overloaded — and without safeguards, a failure in one service can **cascade** and take down others that depend on it (a "cascading failure").

**20. What is the Circuit Breaker pattern?** A pattern that monitors calls to a downstream service and "opens" the circuit (stops making calls, failing fast with a fallback) after a threshold of failures is reached — preventing a struggling downstream service from being overwhelmed further and preventing the caller from wasting resources on calls likely to fail. After a cooldown period, it moves to a "half-open" state to test whether the downstream service has recovered.

**21. What is Resilience4j, and how does it relate to Hystrix?** **Resilience4j** is the current standard resilience library for Java/Spring microservices, providing circuit breakers, retries, rate limiting, bulkheads, and time limiters. It replaced **Netflix Hystrix**, which is now in maintenance mode/deprecated. Spring Cloud Circuit Breaker provides a common abstraction over Resilience4j.

**22. What is a "bulkhead" in the context of resilience patterns?** A pattern (named after ship bulkheads that contain flooding to one section) that limits the number of concurrent calls/resources allocated to a specific dependency, so that if one downstream service is slow or misbehaving, it can't exhaust the thread pool or resources needed to serve calls to other, healthy dependencies.

**23. What's the difference between a retry and a circuit breaker?**

- **Retry**: automatically re-attempts a failed call, useful for transient/temporary failures.
- **Circuit breaker**: stops making calls altogether after repeated failures, to avoid making things worse. They're often used together — but blindly retrying without a circuit breaker can actually worsen an overloaded downstream service by adding more load.

**24. What is a timeout, and why does it matter in microservices?** A limit on how long a service will wait for a response from another service before giving up. Without proper timeouts, a slow downstream service can cause calling threads to pile up waiting, exhausting resources and causing cascading slowdowns across the system.

**25. What is graceful degradation / a fallback response?** Providing a reasonable default or reduced-functionality response when a dependency is unavailable, instead of failing the entire request — e.g., showing cached or default product recommendations if the recommendation service is down, rather than failing the whole page load.

---

## Data Management

**26. Why does each microservice typically have its own database?** To maintain **loose coupling** and independent deployability — if services shared a single database, a schema change for one service could break others, and it becomes harder to scale or evolve services independently. This is often called the "database per service" pattern.

**27. What challenge does having separate databases per service create, and how is it addressed?** It makes it hard to run **transactions across services** (no more single ACID transaction spanning multiple databases) and to query data that spans services. This is commonly addressed with:

- **Saga pattern** — a sequence of local transactions coordinated via events or orchestration, with compensating actions to "undo" prior steps if something fails partway through
- **CQRS (Command Query Responsibility Segregation)** — separating write models from read models, often with a denormalized read-optimized store built from events
- **Event-driven data replication** — services publish events when their data changes, and other services maintain their own local read-optimized copies of data they need

**28. What is the Saga pattern, in more detail?** A way to maintain data consistency across multiple services without a distributed transaction — a business process is broken into a series of local transactions, each one in a different service, with the next step triggered upon success. If a step fails, previously completed steps are undone via **compensating transactions** (e.g., "cancel reservation" to undo "reserve inventory"). Sagas can be coordinated via **orchestration** (a central saga coordinator) or **choreography** (each service reacts to events from the previous step).

**29. What is eventual consistency, and why is it common in microservices?** A consistency model where, after an update, different parts of the system may be temporarily out of sync but will converge to a consistent state eventually (e.g., once an event has propagated). It's common in microservices because enforcing strict, immediate consistency across independently-owned databases is often impractical — many systems accept eventual consistency in exchange for availability and loose coupling.

**30. What is Event Sourcing?** An approach where, instead of storing just the current state of an entity, you store the full sequence of events that led to that state (e.g., "OrderCreated," "ItemAdded," "OrderShipped"). The current state can always be rebuilt by replaying events — useful for auditability and works well alongside CQRS and the Saga pattern.

---

## Configuration, Observability & Deployment

**31. What is centralized configuration management, and why is it needed?** Instead of each service managing its own configuration files independently, a **Config Server** (e.g., Spring Cloud Config, backed by a Git repo) provides configuration to all services centrally — making it easier to manage settings consistently across many services and environments, and to update config without redeploying.

**32. What is distributed tracing, and why is it important in microservices?** A technique for tracking a single request as it flows across multiple services, using a shared **trace ID** propagated through each call. It's essential because a bug or slowdown might only be visible when you can see the full chain of calls a request went through — checking one service's logs alone often isn't enough. Common tools: **Zipkin**, **Jaeger**, using **Micrometer Tracing** (the current standard in Spring, replacing the now-deprecated Spring Cloud Sleuth), often exported via OpenTelemetry.

**33. What is centralized/aggregated logging, and why do microservices need it?** Since logs are spread across many independent service instances, you need a way to collect and search them in one place (e.g., the ELK stack — Elasticsearch, Logstash, Kibana — or Grafana Loki) rather than SSH-ing into individual machines. Correlating logs across services usually relies on a shared **trace/correlation ID** included in each log line.

**34. What is a health check, and how is it used in microservices?** An endpoint (e.g., Spring Boot Actuator's `/actuator/health`) that reports whether a service instance is running correctly. Orchestrators (Kubernetes) and service registries (Eureka) use health checks to decide whether to route traffic to an instance, restart it, or remove it from the pool.

**35. What role does containerization (Docker) play in microservices?** Packaging each service with its dependencies into a portable, consistent container image — ensuring it runs the same way across dev, test, and production environments, and making it easy to deploy many independent services consistently.

**36. What role does Kubernetes play in a microservices architecture?** An orchestration platform that automates deploying, scaling, healing (restarting failed instances), and networking containerized services — handling much of the operational complexity that comes with running many independent microservices at scale. It also provides its own built-in service discovery and load balancing.

**37. What is a Service Mesh (e.g., Istio, Linkerd)?** Infrastructure that manages service-to-service communication concerns (load balancing, retries, timeouts, mutual TLS, observability) at the network/proxy level (often via sidecar proxies), rather than requiring each service's application code to implement these concerns itself.

---

## Practical / Scenario Questions

**38. How would you handle a scenario where Service A calls Service B, and Service B is down?**

- Use a **timeout** so Service A doesn't wait forever
- Use a **circuit breaker** (Resilience4j) to stop repeatedly calling a failing service
- Provide a **fallback** response if appropriate (cached data, default value, or a clear error)
- Ensure the failure doesn't cascade — Service A should still function (perhaps in a degraded way) for functionality that doesn't depend on Service B

**39. How would you design an "Order placed → Inventory updated → Payment processed" flow across three separate services?** Frame it as a **Saga**: each step is a local transaction in its own service. Options:

- **Choreography**: Order Service publishes an "OrderPlaced" event → Inventory Service consumes it, reserves stock, publishes "InventoryReserved" → Payment Service consumes it and processes payment. If payment fails, a compensating event ("PaymentFailed") triggers Inventory Service to release the reserved stock.
- **Orchestration**: A central Saga orchestrator explicitly calls each service in sequence and handles compensations if a step fails.

**40. How would you avoid one slow/failing microservice bringing down your whole system?** Combine: timeouts, circuit breakers, bulkheads (isolating resource pools per dependency), fallback responses, and asynchronous communication (queues) where synchronous coupling isn't necessary — plus good observability (tracing, health checks) to detect issues quickly.

**41. If asked to design a simple e-commerce microservices system, what services might you propose?** A typical breakdown: **User/Auth Service, Product Service, Order Service, Inventory Service, Payment Service, Notification Service**, fronted by an **API Gateway**, with a **Service Registry** (Eureka) for discovery, a **Config Server** for centralized settings, and **Resilience4j** for fault tolerance between them.

---

## Quick tips for the interview

- For a junior role, expect more focus on **conceptual understanding** — monolith vs microservices trade-offs, REST communication, what a circuit breaker/API gateway/service discovery is for — rather than deep distributed-systems theory (CAP theorem nuances, consensus algorithms) or hands-on Kubernetes/service mesh experience.
- It's a strong signal to mention that microservices come with real trade-offs and aren't automatically "better" than a monolith — showing you understand _when_ to use them (not just _how_) reads as more mature engineering judgment.
- If you've only built a small personal project with 2–3 services (e.g., using Eureka + Feign + a Gateway), that's a great concrete example to walk through if asked — much stronger than reciting definitions.