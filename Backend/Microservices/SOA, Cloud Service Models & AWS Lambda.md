---
share_link: https://share.note.sx/5j3u6vd9#xpI0JrQETzdjbeyZnPNQDw
share_updated: 2026-08-03T13:31:51+03:00
---
## Table of Contents

- [[#1. SOA (Service-Oriented Architecture)]]
- [[#2. SOA vs Microservices]]
- [[#3. Cloud Service Models — IaaS, PaaS, SaaS, FaaS]]
- [[#4. AWS Lambda]]
- [[#5. Quick Revision Cheat Sheet]]

---

## 1. SOA (Service-Oriented Architecture)

**SOA** structures an application as a collection of services communicating over a network, typically integrated through a central **Enterprise Service Bus (ESB)**.

```
Service A ──┐
Service B ──┼──► Enterprise Service Bus (ESB) ──► routing, transformation, orchestration, protocol mediation
Service C ──┘
```

### Key Characteristics

- **Enterprise-wide reuse** — services are designed to be shared across _many_ applications org-wide, not owned by one product team.
- **ESB-centric** — the bus handles routing, message transformation, protocol translation (SOAP↔REST, etc.), and often business logic/orchestration.
- **Heavier protocols** — commonly **SOAP** + **WSDL** contracts (recap: [[Web-Technologies-Notes#11. SOAP — XML, WSDL]]), though later SOA implementations also used REST.
- **Shared/centralized data** — services more often share a common database or data layer.
- **Centralized governance** — one architecture team typically owns service contracts and the ESB.

---

## 2. SOA vs Microservices

||SOA|Microservices|
|---|---|---|
|**Integration**|Centralized ESB|Decentralized — direct calls or lightweight messaging ([[Microservices-Notes#8. Messaging: RabbitMQ vs Kafka|
|**Service scope**|Coarse-grained, enterprise-wide reusable services|Fine-grained, one business capability per service|
|**Data**|Often shared database|[[Microservices-Notes#12. Database per Service|
|**Governance**|Centralized (one team owns contracts/ESB)|Decentralized — each team owns its service end-to-end|
|**Protocols**|Heavier — SOAP/WSDL common|Lightweight — REST/JSON, gRPC, events|
|**Deployment**|Services often still deployed together / less independent|Independently deployable ([[Architecture-Foundations-Notes#3. Modular Monolith|
|**Failure isolation**|ESB can be a single point of failure|Failures isolated per service (with [[Microservices-Notes#13. CAP Theorem & Resilience Patterns|
|**Origin era**|Early/mid-2000s enterprise integration|~2011+, cloud-native, DevOps-driven|

> [!important] The one-sentence distinction **Microservices are SOA with the ESB removed and the boundaries drawn much smaller** — decentralizing what SOA centralized (integration logic, governance, and often data), trading a single smart bus for many small, independently deployable, dumb-pipes-smart-endpoints services. This is the same "smart broker vs smart consumer" trade-off pattern seen in [[Microservices-Notes#8. Messaging: RabbitMQ vs Kafka|RabbitMQ vs Kafka]].

> [!tip] Not a strict evolution SOA isn't "wrong" — it fits well when genuine **enterprise-wide service reuse** across many disparate applications is the actual goal. Microservices fit better when independent, per-team deployability and scaling are the priority. Many large orgs still run legitimate SOA for cross-cutting enterprise integration, with microservices inside individual product teams.

---

## 3. Cloud Service Models — IaaS, PaaS, SaaS, FaaS

```
                Who manages what?
┌────────────────┬───────────┬───────────┬────────────┬───────────┐
│                │  IaaS     │  PaaS     │  FaaS      │  SaaS     │
├────────────────┼───────────┼───────────┼────────────┼───────────┤
│ Application    │   YOU     │   YOU     │   YOU      │ PROVIDER  │
│ Runtime        │   YOU     │ PROVIDER  │ PROVIDER   │ PROVIDER  │
│ OS             │   YOU     │ PROVIDER  │ PROVIDER   │ PROVIDER  │
│ Virtualization │ PROVIDER  │ PROVIDER  │ PROVIDER   │ PROVIDER  │
│ Servers/Storage│ PROVIDER  │ PROVIDER  │ PROVIDER   │ PROVIDER  │
│ Networking     │ PROVIDER  │ PROVIDER  │ PROVIDER   │ PROVIDER  │
└────────────────┴───────────┴───────────┴────────────┴───────────┘
```

| Model                                  | You Manage                         | Provider Manages                                                                               | Example                                             |
| -------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| **IaaS** (Infrastructure as a Service) | OS, runtime, app, scaling          | Physical hardware, virtualization, network                                                     | AWS EC2, Azure VMs, Google Compute Engine           |
| **PaaS** (Platform as a Service)       | Application code + config          | OS, runtime, patching, scaling infrastructure                                                  | Heroku, AWS Elastic Beanstalk, Google App Engine    |
| **FaaS** (Function as a Service)       | Just the function's business logic | Everything else, **including the running server itself** — provisions on demand per invocation | AWS Lambda, Azure Functions, Google Cloud Functions |
| **SaaS** (Software as a Service)       | Nothing — just use it              | The entire application                                                                         | Gmail, Salesforce, Slack, Notion                    |

### The Abstraction Ladder

```
IaaS ──► PaaS ──► FaaS ──► SaaS
(most control,          (least control,
 most ops burden)         zero ops burden)
```

- **IaaS**: you're still managing servers (patching, scaling, OS) — think [[Spring-Boot-Notes#6. Embedded Servers (Tomcat / Jetty / Undertow)|running your own Tomcat]] on a rented VM.
- **PaaS**: you push code; the platform builds/runs/scales it — closer to `git push heroku main`.
- **FaaS**: you write a single function; the platform runs it **only when triggered**, for exactly the duration it executes, then tears it down — see [[#4. AWS Lambda]].
- **SaaS**: you're the _end user_, not the developer — no code at all.

> [!tip] Where microservices/SOA fit into this Both SOA and microservices are **architectural styles** — orthogonal to _where_ you deploy them. A microservice can run on IaaS (your own EC2 instances + Kubernetes), PaaS (Elastic Beanstalk), or even be individually decomposed further into FaaS functions per endpoint.

---

## 4. AWS Lambda

**AWS Lambda** is Amazon's FaaS offering — you upload a function; AWS handles provisioning, scaling, and teardown automatically, **per invocation**.

### Core Model

```
Event Source (API Gateway, S3 upload, SQS message, schedule, ...)
        │
        ▼
   AWS Lambda ──► spins up an execution environment ──► runs your handler function ──► returns result
        │
   environment is FROZEN (reused for a bit) or DESTROYED (after idle timeout)
```

### Key Characteristics

|Property|Detail|
|---|---|
|**Trigger-based**|Invoked by **events** — API Gateway (HTTP), S3 (file upload), SQS/SNS (messages), EventBridge (schedules/events), DynamoDB Streams|
|**Pay-per-use**|Billed per invocation + execution duration (milliseconds), **not** for idle time — the defining FaaS economic model|
|**Auto-scaling**|AWS runs as many concurrent instances of your function as needed, automatically, with no capacity planning|
|**Stateless**|Each invocation should be independent — no reliable in-memory state between calls (though the _execution environment_ itself may be briefly reused, this isn't guaranteed)|
|**Execution limits**|Max execution time (15 minutes), memory (up to 10GB, tunable, which also scales CPU proportionally), deployment package size limits|
|**Languages**|Java, Node.js, Python, Go, .NET, Ruby, or any language via a **custom runtime**|

### Cold Start — The Classic FaaS Trade-off

```
First invocation (cold):  provision environment → load runtime/dependencies → run handler   (slower)
Subsequent invocations (warm): environment reused → run handler directly                      (fast)
```

- A **cold start** happens when Lambda must spin up a brand-new execution environment (first invocation, or after scaling up, or after an idle timeout tears the old one down).
- **JVM-based languages (Java) suffer more from cold starts** than Node/Python/Go, due to JVM startup + classloading overhead — directly relevant if deploying a Spring Boot app as a Lambda function (mitigations: **Spring Cloud Function**, **GraalVM native images** for near-instant startup, or **Provisioned Concurrency** to keep warm instances ready).

### When Lambda Fits Well

✅ Event-driven, bursty, or unpredictable workloads (image processing on upload, webhook handlers, scheduled cleanup jobs). ✅ Workloads with **long idle periods** — you pay nothing when it's not running, unlike an always-on server. ✅ Simple, short-lived, stateless operations.

### When Lambda Fits Poorly

❌ **Long-running** processes (hard 15-minute execution ceiling). ❌ **Latency-sensitive**, consistently high-traffic APIs where cold starts and per-invocation overhead outweigh the benefit — a normal always-on server (or container on ECS/Kubernetes) is often both cheaper and more predictable at sustained high load. ❌ Workloads needing **persistent in-memory state** across requests (e.g. a warm cache local to one instance) — Lambda's stateless-by-design model fights this.

### Lambda in a Microservices Context

Lambda functions are a natural fit for the **very fine-grained end** of the [[Microservices-Notes#1. What Are Microservices?|microservices spectrum]] — sometimes called **"nanoservices"** when taken to the extreme (one function per single operation). Often paired with **API Gateway** ([[Microservices-Notes#4. API Gateway|API Gateway pattern]]) as the front door routing HTTP requests to the right function, and **Step Functions** for orchestrating multi-step workflows across several Lambdas (echoing [[Microservices-Notes#9. SAGA Pattern|SAGA orchestration]]).

---

## 5. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|SOA|Enterprise-wide, coarse-grained services, integrated via a central ESB|
|Microservices vs SOA|Decentralized integration + governance, fine-grained, database-per-service, lightweight protocols|
|IaaS|You manage OS + runtime + app — rented hardware/VMs|
|PaaS|You manage just the app — platform handles OS/runtime/scaling|
|FaaS|You manage just a function — provider handles everything, including the server, per invocation|
|SaaS|You manage nothing — you're the end user of a finished app|
|AWS Lambda|Event-triggered, pay-per-invocation, auto-scaling, stateless, 15-min max runtime|
|Cold start|First/new execution environment spin-up — slower; JVM/Java hit harder than Node/Python|
|Lambda best fit|Bursty, event-driven, short-lived, stateless workloads|
|Lambda poor fit|Long-running, latency-critical high-throughput, or stateful workloads|

---
