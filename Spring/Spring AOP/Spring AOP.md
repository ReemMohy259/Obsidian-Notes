---
share_link: https://share.note.sx/elvmnhan#SMHrOgp1onmHDkD/tybmyA
share_updated: 2026-07-09T05:38:18+03:00
---
## Table of Contents

- [[#1. What is AOP?]]
- [[#2. Why AOP? (Importance & Benefits)]]
- [[#3. Proxies — The Mechanism Behind Spring AOP]]
- [[#4. JDK Dynamic Proxy vs CGLIB Proxy]]
- [[#5. Core Vocabulary — Aspect, JoinPoint, Pointcut, Weaving]]
- [[#6. Weaving — Compile-Time / Load-Time / Runtime]]
- [[#7. Advice Types]]
- [[#8. Order of Advice Execution]]
- [[#9. Pointcut Expressions & Declarations]]
- [[#10. AOP vs Servlet Filters]]
- [[#11. Common Use Cases — Logging, Auditing, Transactions]]
- [[#12. The Self-Invocation Problem & Self Injection]]
- [[#13. Quick Revision Cheat Sheet]]

---
# 1. What is AOP?

**Aspect-Oriented Programming (AOP)** is a programming paradigm that separates **cross-cutting concerns** — logic that applies across many unrelated parts of an application — from core business logic.

Typical cross-cutting concerns:

- Logging
- Security
- Transactions
- Exception handling
- Auditing
- Performance monitoring

### Without AOP

```
Service A
   logging
   business code

Service B
   logging
   business code

Service C
   logging
   business code
```

Every service repeats the same boilerplate.

### With AOP

```
        Logging Aspect
              │
   ┌──────────┼──────────┐
   │          │          │
   A          B          C
```

Business classes contain **only business logic** — the cross-cutting logic lives in one place (an **Aspect**) and is woven into the relevant methods automatically.

---

# 2. Why AOP? (Importance & Benefits)

### Without AOP — Duplicated Concerns

```
Service A
    logging
    security
    validation

Service B
    logging
    security
    validation
```

### With AOP — Centralized Concerns

```
Logging Aspect
Security Aspect
Validation Aspect
        │
   ┌────┼────┐
   │    │    │
   A    B    C
```

### Benefits

✅ Separation of concerns 
✅ Code reuse
✅ Cleaner business logic 
✅ Easier maintenance 
✅ Centralized logging/security/transactions

### Typical Use Cases

- Logging
- Transactions
- Security
- Auditing
- Performance monitoring
- Exception handling

---

# 3. Proxies — The Mechanism Behind Spring AOP

### What is a Proxy?

A **proxy** is an object that sits between the client and the target object, intercepting method calls to run extra logic (logging, security, transactions, etc.) **before or after** the real method executes.

```
Client
   │
   ▼
Proxy
   │
   ▼
Target Object
```

> [!important] **Spring AOP works entirely using proxies.** There is no bytecode manipulation of your original class — a _wrapper_ object is created around it, and the container injects the **proxy**, not the raw target, wherever the bean is referenced.

### With an Interface (JDK Dynamic Proxy)

```java
interface UserService {
    void save();
}

class UserServiceImpl implements UserService {
    public void save() {}
}
```

Spring creates:

```
Client
   │
JDK Proxy (implements UserService)
   │
UserServiceImpl
```

### Without an Interface (CGLIB Proxy)

```java
class UserService {
    void save() {}
}
```

Spring creates:

```
Client
   │
CGLIB Proxy (extends UserService)
   │
UserService
```

---

# 4. JDK Dynamic Proxy vs CGLIB Proxy

||JDK Dynamic Proxy|CGLIB Proxy|
|---|---|---|
|Mechanism|Java Reflection|Generates a **subclass** at runtime|
|Requirement|At least one interface|Works even **without** interfaces|
|What it proxies|Interface methods only|All non-final methods|
|Creation cost|Faster|Slightly heavier|
|Default behavior|Used when the bean **implements an interface**|Used when there's **no interface**, or `proxyTargetClass=true` is set|

### Limitations

**JDK Proxy**

- Only intercepts **interface** methods.
- Cannot proxy class-only (non-interface) methods.

**CGLIB** Cannot intercept:

- `final` classes
- `final` methods
- `private` methods
- `static` methods

> [!tip] Why this matters in practice If a colleague calls a method that isn't declared on the interface, or marks a class/method `final`, AOP advice **silently won't apply** to it — a very common source of "why isn't my `@Transactional`/`@Cacheable` working?" bugs.

### Forcing CGLIB Even With an Interface

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

Spring Boot actually defaults to CGLIB (`proxyTargetClass=true`) for **all** proxying since Boot 2.x, even when interfaces exist — for consistency across the app.

---

# 5. Core Vocabulary — Aspect, JoinPoint, Pointcut, Weaving

### Aspect

A class containing cross-cutting logic.

```java
@Aspect
@Component
public class LoggingAspect {
}
```

### Advice

The action performed at a specific point — _before_, _after_, or _around_ method execution. (Full breakdown in [[#7. Advice Types]].)

### Join Point

A point during program execution where advice **can** run.

> [!info] Spring AOP supports **method execution join points only**. Full AspectJ (outside Spring AOP) additionally supports field access, constructor calls, static initializers, and more — Spring deliberately restricts itself to method execution for simplicity and proxy-based implementation.

### Pointcut

An **expression** that selects _which_ join points (methods) advice should apply to.

```java
execution(* com.app.service.*.*(..))
```

### Target Object

The original object being advised.

```
Logging Proxy
      │
Target Object
```

### Proxy

The object Spring creates that wraps the target and applies advice at matched join points.

### Weaving

The process of connecting **aspects** to **target objects** to create the advised (proxied) object. Covered in depth next.

---

# 6. Weaving — Compile-Time / Load-Time / Runtime

**Weaving** is _when and how_ the aspect logic actually gets attached to the target code. There are three general strategies (across the AOP world, not all used by Spring AOP itself):

|Weaving Type|When It Happens|Used By|
|---|---|---|
|**Compile-time weaving**|Aspect code is woven directly into `.class` bytecode during compilation|Full AspectJ (via `ajc` compiler)|
|**Load-time weaving (LTW)**|Bytecode is woven when classes are **loaded** by the JVM, via a special class loader/agent|Full AspectJ, can be enabled in Spring via `@EnableLoadTimeWeaving`|
|**Runtime weaving**|No bytecode modification at all — a **proxy** object is created at runtime that wraps the target|**Spring AOP (default & most common)**|

> [!important] Spring AOP = Runtime Weaving Only Spring performs weaving **at runtime**, using dynamic proxies (JDK or CGLIB). This is simpler and requires no special build step or agent, but it comes with the proxy limitations from [[#4. JDK Dynamic Proxy vs CGLIB Proxy]] (no field/constructor join points, no advising `final`/`private`/`static` methods, and the [[#12. The Self-Invocation Problem & Self Injection|self-invocation problem]]).
> 
> If you need compile-time or load-time weaving (e.g. to advise field access or private methods), you need **full AspectJ**, not plain Spring AOP — Spring can integrate with it via `spring-aspects` + `@EnableLoadTimeWeaving`, but that's a different, heavier mechanism.

---

# 7. Advice Types

### `@Before`

Runs **before** method execution.

```java
@Before("execution(* com.app.service.*.*(..))")
public void beforeAdvice() {
    System.out.println("Before method");
}
```

Use cases: logging, validation, authentication.

### `@After`

Runs after method execution **regardless of outcome** (success or exception) — like a `finally` block.

```
method executes
      │
success or exception
      │
   @After
```

```java
@After("execution(* com.app.service.*.*(..))")
public void afterAdvice() {
    System.out.println("Always executes");
}
```

### `@AfterReturning`

Runs **only if** the method completes successfully.

```java
@AfterReturning(
    pointcut = "execution(* com.app.service.*.*(..))",
    returning = "result"
)
public void afterReturning(Object result) {
    System.out.println(result);
}
```

### `@AfterThrowing`

Runs **only when** an exception is thrown.

```java
@AfterThrowing(
    pointcut = "execution(* com.app.service.*.*(..))",
    throwing = "ex"
)
public void afterThrowing(Exception ex) {
    System.out.println(ex.getMessage());
}
```

### `@Around` 

Can:
- execute logic **before** the method
- execute logic **after** the method
- **modify arguments**
- **modify the return value**
- **stop execution completely** (skip calling the target method)

```java
@Around("execution(* com.app.service.*.*(..))")
public Object around(ProceedingJoinPoint pjp) throws Throwable {

    System.out.println("Before");

    Object result = pjp.proceed(); // must call this to continue the chain

    System.out.println("After");

    return result;
}
```

> [!danger] Don't forget `pjp.proceed()` If you omit the call to `proceed()`, the target method **never executes** — this is easy to forget and a common bug source with `@Around`.

### Summary Table

|Advice|Executes|
|---|---|
|`@Before`|Before method|
|`@After`|Always after method (success or failure)|
|`@AfterReturning`|After successful execution only|
|`@AfterThrowing`|After an exception only|
|`@Around`|Before **and** after — full control, can short-circuit|

---

# 8. Order of Advice Execution

### Successful Execution

```
@Around (before half)
        │
     @Before
        │
   Target Method
        │
  @AfterReturning
        │
      @After
        │
@Around (after half)
```

### Exception Case

```
@Around (before half)
        │
     @Before
        │
Target Method throws Exception
        │
  @AfterThrowing
        │
      @After
        │
@Around (after half)
```

### Ordering Multiple Aspects

```java
@Order(1)
@Component
@Aspect
class LoggingAspect {}

@Order(2)
@Component
@Aspect
class SecurityAspect {}
```

**Smaller `@Order` value = higher priority** (runs first on the "before" side, last on the "after" side — like nested wrapping).

```
Logging Before
    │
Security Before
    │
Target Method
    │
Security After
    │
Logging After
```

---

# 9. Pointcut Expressions & Declarations

### Basic Syntax

```java
execution(modifiers-pattern? return-type-pattern declaring-type-pattern? method-pattern(parameters-pattern))
```

(`?` = optional field)

### Example

```java
execution(* com.app.service.*.*(..))
```

Reads as: any return type, any class in the `service` package, any method name, with any arguments.

### Wildcard Cheat Sheet

|Symbol|Meaning|
|---|---|
|`*`|Any single return type / any single method name / any single class|
|`..`|Any number of parameters, **or** any number of sub-packages (when used in a package path)|

### Other Common Pointcut Designators (beyond `execution`)

|Designator|Matches|
|---|---|
|`within(com.app.service..*)`|Any join point within the given package/subpackages|
|`@annotation(com.app.Loggable)`|Methods annotated with a specific annotation|
|`@within(org.springframework.stereotype.Service)`|Methods in classes annotated with a specific annotation|
|`args(String,..)`|Methods whose arguments match a given pattern|
|`target(com.app.PaymentGateway)`|Join points where the target object is of the given type|

### Reusable Pointcut Declarations

Instead of repeating the same expression:

```java
@Before("execution(* com.app.service.*.*(..))")
@After("execution(* com.app.service.*.*(..))")
```

Declare it once:

```java
@Pointcut("execution(* com.app.service.*.*(..))")
public void serviceLayer() {}
```

And reference it by name:

```java
@Before("serviceLayer()")
public void before() {}

@After("serviceLayer()")
public void after() {}
```

### Combining Pointcuts

```java
@Pointcut("execution(* com.app.service.*.*(..))")
public void serviceLayer() {}

@Pointcut("@annotation(com.app.Loggable)")
public void loggableMethods() {}

@Before("serviceLayer() && loggableMethods()")
public void beforeLoggableServiceCall() {}
```

Logical operators `&&`, `||`, `!` combine pointcuts just like boolean expressions.

---

# 10. AOP vs Servlet Filters

|AOP|Servlet Filter|
|---|---|
|Intercepts method execution|Intercepts HTTP requests/responses|
|Works inside Spring beans|Works before reaching Spring MVC|
|Business logic level|Web layer level|
|Can intercept service/repository methods|Cannot intercept service methods|
|Uses proxies|Uses the servlet container|

### Request Flow

```
HTTP Request
      │
Servlet Filter
      │
DispatcherServlet
      │
  Controller
      │
   Service
      │
  AOP Proxy
      │
Target Method
```

### When to Use Which

**Filter**

- Authentication
- CORS
- Request logging
- Encoding
- Rate limiting

**AOP**

- Logging service methods
- Transactions
- Performance monitoring
- Auditing
- Security at method level

---

# 11. Common Use Cases — Logging, Auditing, Transactions

### Logging & Auditing Aspects

```java
@Aspect
@Component
public class AuditLogAspect {

    @Pointcut("@annotation(com.app.Auditable)")
    public void auditableMethods() {}

    @AfterReturning(pointcut = "auditableMethods()", returning = "result")
    public void logAudit(JoinPoint jp, Object result) {
        String method = jp.getSignature().getName();
        Object[] args = jp.getArgs();
        System.out.printf("AUDIT: %s called with %s -> returned %s%n", method, args, result);
    }
}
```

Combine with `@annotation(...)` pointcuts so only explicitly marked methods get audited — keeps the aspect's scope intentional rather than blanket-matching a whole package.

### Transaction Management (`@Transactional`)

- `@Transactional` is itself implemented as **AOP advice** — Spring wraps the annotated bean in a proxy that starts a transaction **before** the method and commits/rolls back **after**, based on success or exception.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryService.reserveStock(order);
    }
}
```

- Because this is proxy-based, all the same limitations apply: calling an `@Transactional` method **from within the same class** (self-invocation) bypasses the proxy and the transaction **will not start** — see next section.
- Default rollback behavior: rolls back on unchecked exceptions (`RuntimeException`, `Error`); commits through checked exceptions unless configured with `rollbackFor`.

### Performance Monitoring Example

```java
@Around("serviceLayer()")
public Object logExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    Object result = pjp.proceed();
    long elapsed = System.currentTimeMillis() - start;
    System.out.printf("%s executed in %dms%n", pjp.getSignature(), elapsed);
    return result;
}
```

---

# 12. The Self-Invocation Problem & Self Injection

### The Problem

Since Spring AOP is **proxy-based** (see [[#6. Weaving — Compile-Time / Load-Time / Runtime]]), advice only applies when a call comes **through the proxy** — i.e., from outside the class. A method calling **another method on `this`** bypasses the proxy entirely.

```java
@Service
public class UserService {

    public void registerUser() {
        saveUser();   // direct "this" call
    }

    @Transactional
    public void saveUser() {
        // ...
    }
}
```

```
registerUser()
      │
      ▼
this.saveUser()
      │
      ▼
Bypasses Spring Proxy
      │
      ▼
@Transactional NOT applied
```

The transaction silently never starts — one of the most common real-world Spring AOP bugs.

### Fix #1: Self Injection

**Self injection** means a bean injects a reference to **its own Spring-managed instance** (i.e., its proxy) and calls methods through that reference instead of `this`.

```java
@Service
public class UserService {

    @Autowired
    private UserService self;

    public void registerUser() {
        self.saveUser();
    }

    @Transactional
    public void saveUser() {
        // ...
    }
}
```

```
registerUser()
      │
      ▼
self.saveUser()
      │
      ▼
 Spring Proxy
      │
      ▼
Transaction Starts
      │
      ▼
 saveUser()
```

Now the call goes **through** the proxy, so `@Transactional` (and equally `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize`) is correctly applied.

> [!info] Self Injection Is a Fallback — Lowest Precedence Spring's dependency resolution order when injecting is:
> 
> 1. Other matching beans
> 2. `@Primary` beans
> 3. `@Qualifier` matches
> 4. Bean-name matches
> 5. **Self reference (last resort)**
> 
> Self injection only kicks in when nothing else resolves the dependency — Spring explicitly documents it as a fallback mechanism, not a first-class pattern.

### Fix #2 (Recommended): Extract to Another Bean

Spring's own guidance is to **avoid relying on self injection** where possible, and instead move the proxied method into a separate bean:

```java
@Service
public class UserService {

    private final UserTransactionService txService;

    public UserService(UserTransactionService txService) {
        this.txService = txService;
    }

    public void registerUser() {
        txService.saveUser();
    }
}

@Service
public class UserTransactionService {

    @Transactional
    public void saveUser() {
        // ...
    }
}
```

The call now naturally passes through a real inter-bean call (and therefore through the proxy), with no special-casing needed.

### Key Points

- **Self injection** = a bean injecting a reference to its own Spring-managed instance (its proxy).
- Exists to solve the **self-invocation problem** with proxy-based annotations (`@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize`).
- It is a **fallback** dependency-resolution mechanism with the **lowest precedence**.
- Prefer **extracting functionality into a separate bean** as the cleaner, more idiomatic fix.

---

# 13. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|Proxy|Object that intercepts method calls|
|JDK Proxy|Interface-based|
|CGLIB|Subclass-based|
|AOP|Separates cross-cutting concerns|
|Aspect|Class containing advice|
|Advice|Action executed at a join point|
|Join Point|Method execution (Spring AOP only)|
|Pointcut|Expression selecting methods|
|Weaving|Connecting aspect to target — Spring does this **at runtime** via proxies|
|Filter|HTTP request/response level|
|AOP|Spring bean/method level|
|Advice Order|`@Around`(before) → `@Before` → Method → `@AfterReturning`/`@AfterThrowing` → `@After` → `@Around`(after)|
|`*`|Any return type/method|
|`..`|Any number of parameters or subpackages|
|`@Pointcut`|Reusable pointcut expression|
|Self-invocation problem|Calling a proxied method via `this` bypasses advice|
|Self injection|Fallback fix: inject own proxy and call through it|

---