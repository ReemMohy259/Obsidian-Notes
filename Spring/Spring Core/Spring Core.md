---
share_link: https://share.note.sx/5htcudmv#hTbGyvqd2i3vee+TXBrIzQ
share_updated: 2026-07-25T18:04:19+03:00
---
# Spring Framework — Core: Comprehensive Guide

## Table of Contents

- [[#1. IoC Container (BeanFactory / ApplicationContext)]]
- [[#2. Dependency Injection (DI)]]
- [[#3. Constructor / Setter / Field Injection]]
- [[#4. Circular Dependency & Resolution]]
- [[#5. Bean Lifecycle]]
- [[#6. Bean Scopes]]
- [[#7. Autowiring & @Qualifier / @Primary]]
- [[#8. Stereotype Annotations — @Component / @Service / @Repository / @Controller]]
- [[#9. `@Configuration` & `@Bean` Vs. `@Component`]]
- [[#10. @ComponentScan]]
- [[#11. @Value and Property Resolution]]
- [[#12. Background Bean Initialization (Spring 6.2+)]]
- [[#13. Spring 7 — Programmatic Bean Registration (BeanRegistrar)]]
- [[#14. Overloaded @Bean Methods]]
- [[#15. Property Source Precedence]]

---

# 1. IoC Container (BeanFactory / ApplicationContext) 

### What is Inversion of Control (IoC)?
- Instead of your code creating and wiring its own dependencies (`new Service()`), the **container** creates objects (beans), configures them, and manages their entire lifecycle.
- Control over object creation is **inverted**: from the application code → to the framework.

### `BeanFactory` vs `ApplicationContext`

|Feature|`BeanFactory`|`ApplicationContext`|
|---|---|---|
|Role|Root/basic IoC container interface|Superset (extends `BeanFactory`)|
|Bean instantiation|**Lazy** by default|**Eager** for singletons by default|
|Internationalization (i18n)|❌|✅ `MessageSource`|
|Event publishing|❌|✅ `ApplicationEventPublisher`|
|AOP integration|Limited|✅ Full support|
|Environment abstraction (profiles, property sources)|❌|✅|
|Typical usage|Rarely used directly|**Standard choice** in real apps|

> [!tip] In almost all modern Spring apps you interact with `ApplicationContext`, not raw `BeanFactory`. `BeanFactory` is the low-level contract `ApplicationContext` builds on.

### Common `ApplicationContext` Implementations

- `AnnotationConfigApplicationContext` — Java-config / annotation-based (most common today)
- `ClassPathXmlApplicationContext` — legacy XML config from classpath
- `FileSystemXmlApplicationContext` — legacy XML config from filesystem
- `WebApplicationContext` (and `AnnotationConfigWebApplicationContext`) — web-aware variant, has access to `ServletContext`

### Container Startup Flow (simplified)

```
Load bean definitions (scan / config classes / XML)
        ↓
Post-process bean definitions (BeanFactoryPostProcessor)
        ↓
Instantiate singleton beans (eager, by default)
        ↓
Populate properties / inject dependencies
        ↓
Apply BeanPostProcessor (before init)
        ↓
Call lifecycle init callbacks (@PostConstruct → afterPropertiesSet → init())
        ↓
Apply BeanPostProcessor (after init)
        ↓
Container is READY — beans available via getBean()
        ↓
On shutdown → destroy callbacks (@PreDestroy → destroy() → custom destroy)
```

### Key Interfaces to Know

- `BeanFactoryPostProcessor` — modifies **bean definitions** before any bean is instantiated (e.g. `PropertySourcesPlaceholderConfigurer`).
- `BeanPostProcessor` — intercepts bean **instances** before/after initialization (e.g. powers `@Autowired`, AOP proxying, `@Value` conversion).

---
# 2. Dependency Injection (DI)

### Definition
DI is the pattern Spring's IoC container uses to supply an object's dependencies **from the outside**, rather than the object creating them itself.

### Why DI?
- **Loose coupling** — classes depend on abstractions, not concrete implementations.
- **Testability** — dependencies can be mocked/stubbed easily.
- **Centralized configuration** — wiring logic lives in one place, not scattered across `new` calls.
- **Lifecycle management** — the container controls creation, scope, and destruction.

### DI vs. Dependency Lookup

||Dependency Injection|Dependency Lookup|
|---|---|---|
|Who asks for the dependency|Nobody — it's _pushed_ in|The object actively _pulls_ it (e.g. `context.getBean(...)`, JNDI lookup)|
|Coupling to container|None|Tight — code depends on the container API|
|Spring's preferred style|✅ Yes|Used only in edge cases (e.g. static contexts)|

### The Three Injection Types

Covered in detail in [[#3. Constructor / Setter / Field Injection]].

---

# 3. Constructor / Setter / Field Injection

### 3.1 Constructor Injection ✅ (Recommended)

```java
@Component
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

- Dependencies are **final** → immutable, thread-safe by design.
- Guarantees the object is **never in an incomplete state** (can't exist without required deps).
- Since Spring 4.3, `@Autowired` is **optional** if the class has a **single constructor**.
- Best for **mandatory** dependencies and easiest to unit test (no reflection/container needed — just call `new OrderService(mockGateway)`).

### 3.2 Setter Injection

```java
@Component
public class OrderService {
    private NotificationService notificationService;

    @Autowired
    public void setNotificationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

- Good for **optional** dependencies (can be left unset / reconfigured later).
- Object can exist in a partially-configured state — less safe than constructor injection.

### 3.3 Field Injection ⚠️ (Discouraged)

```java
@Component
public class OrderService {
    @Autowired
    private InventoryService inventoryService;
}
```

- Concise, but:
    - Requires **reflection** to set private fields → can't be `final`.
    - **Hides dependencies** — no explicit constructor signature listing what the class needs.
    - Harder to unit test without a DI container or reflection-based mocking.
    - Cannot enforce immutability.

> [!danger] Best Practice Prefer **constructor injection** for required dependencies. Use setter injection only for genuinely optional ones. Avoid field injection in production code.

---

# 4. Circular Dependency & Resolution

### What Is It?

Bean A depends on Bean B, and Bean B depends on Bean A (directly or through a chain).

```
BeanA → needs → BeanB
BeanB → needs → BeanA
```

### Constructor Injection Circular Dependency → FAILS

- Spring **cannot** resolve circular dependencies when **both** beans use constructor injection.
- Throws `BeanCurrentlyInCreationException` at startup.
- Reason: a fully-constructed instance is required to inject into the other's constructor, but neither can finish first.

### Setter / Field Injection Circular Dependency → Can Be Resolved

Spring resolves this for **singleton** beans using its **three-level cache** mechanism:

1. **Singleton objects cache** — fully initialized beans.
2. **Early singleton objects cache** — raw, not-yet-fully-initialized bean references.
3. **Singleton factories cache** — factories that can produce an early reference (used for proxying, e.g. AOP).

**Flow:**

```
Create BeanA (raw instance, not yet fully populated)
        ↓
Expose early reference of BeanA in cache
        ↓
Start populating BeanA's properties → needs BeanB
        ↓
Create BeanB (raw instance)
        ↓
BeanB needs BeanA → gets the EARLY reference from cache
        ↓
BeanB finishes initialization
        ↓
BeanA finishes initialization using completed BeanB
```

> [!warning] Limitation This early-reference trick only works for **setter/field injection** and only for **singleton scope**. It does **not** work for `prototype`-scoped beans or constructor injection.

### Breaking Circular Dependencies

- **Best practice: redesign** — circular dependencies usually indicate a design smell (extract shared logic into a third bean).
- `@Lazy` on one of the injection points defers resolution until first use, breaking the immediate cycle:

```java
@Component
public class BeanA {
    public BeanA(@Lazy BeanB beanB) { ... }
}
```

- Use setter injection instead of constructor injection as a (less clean) workaround.

---

# 5. Bean Lifecycle

### Initialization Order

When multiple lifecycle mechanisms are configured on the **same bean**, they are called in this order:

1. Methods annotated with `@PostConstruct`
2. `afterPropertiesSet()` — from the `InitializingBean` callback interface
3. A custom-configured `init()` method

### Destruction Order

Destroy methods follow the **same order**:

1. Methods annotated with `@PreDestroy`
2. `destroy()` — from the `DisposableBean` callback interface
3. A custom-configured `destroy()` method

> [!tip] Mnemonic Annotation → Interface → Custom config, for **both** init and destroy.

### Full Lifecycle Diagram (with post-processors)

```
Instantiate bean (constructor)
        ↓
Populate properties (DI)
        ↓
Aware interfaces (BeanNameAware, BeanFactoryAware, ApplicationContextAware, ...)
        ↓
BeanPostProcessor.postProcessBeforeInitialization()
        ↓
@PostConstruct → afterPropertiesSet() → custom init()
        ↓
BeanPostProcessor.postProcessAfterInitialization()  (AOP proxies applied here)
        ↓
Bean ready for use
        ↓
   ... application runs ...
        ↓
Container shutdown
        ↓
@PreDestroy → destroy() → custom destroy()
```

> Aware interfaces are some callback interfaces when the bean wants to know some information about the spring container that running inside it
---

# 6. Bean Scopes

|Scope|Description|Availability|
|---|---|---|
|`singleton` (default)|One shared instance per Spring container|Always|
|`prototype`|New instance every time the bean is requested|Always|
|`request`|One instance per HTTP request|Web-aware `ApplicationContext` only|
|`session`|One instance per HTTP session|Web-aware `ApplicationContext` only|
|`application`|One instance per `ServletContext`|Web-aware `ApplicationContext` only|
|`websocket`|One instance per WebSocket session|Web-aware `ApplicationContext` only|

```java
@Component
@Scope("prototype")
public class ReportGenerator { ... }
```


> [!NOTE] Prototype Bean Lifecycle
> 
> Unlike singleton beans, **Spring manages only the creation and initialization of prototype beans**. After a prototype bean is created and injected, **Spring no longer manages its lifecycle**.
> 
> ==Managed by Spring==
> - Instantiate the bean
> - Inject dependencies
> - Execute initialization callbacks (`@PostConstruct`, `InitializingBean`, `init-method`)
>   
> ==Not Managed by Spring==
>   - Destruction callbacks (`@PreDestroy`, `DisposableBean`, `destroy-method`) **are NOT invoked automatically**.
>     
> The client application is ==Responsible for cleanup==

### Injecting a Shorter-Lived Bean into a Longer-Lived One

Injecting a `prototype`/`request`/`session` bean directly into a `singleton` bean is a classic pitfall — the singleton captures **one instance forever**, defeating the shorter scope.

#### First Solution: Scoped Proxies

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserRequestContext { ... }
```

The singleton gets injected with a **proxy** that resolves the _real_ instance from the current scope on every method call. the proxy delegates the call to the currently active object.

|Proxy Mode|Mechanism|
|---|---|
|`ScopedProxyMode.TARGET_CLASS`|CGLIB (subclasses the target class)|
|`ScopedProxyMode.INTERFACE`|JDK dynamic proxy (requires an interface)|
#### Another Solutions:
- `@Lookup` (Method Injection)
- `ObjectProvider<T>`
- `ObjectFactory<T>`
- `jakarta.inject.Provider<T>`
- Inject `ApplicationContext` (least recommended)

##### ✅ Solution 1: `@Lookup` (Method Injection) 

Spring overrides the method and returns a **new prototype bean** on every call.

```java
@Component
@Scope("prototype")
class PrototypeBean {}

@Service
class UserService {

    public void execute() {
        PrototypeBean bean = getPrototypeBean();
    }

    @Lookup  // No implementation leave it to spring
    protected PrototypeBean getPrototypeBean() {}
    // This method must be public/protected not final and not static 
    // As the subclass proxy will impelemnt it
}
```

```
execute()
    │
    ▼
getPrototypeBean()
    │
    ▼
Spring Container
    │
    ▼
New PrototypeBean
```

**Best when:** You always need a fresh prototype instance.

##### ✅ Solution 2: `ObjectProvider<T>` 

Request the bean only when needed.

```java
@Service
class UserService {

    @Autowired
    private ObjectProvider<PrototypeBean> provider;

    public void execute() {
        PrototypeBean bean = provider.getObject();
    }
}
```

```
execute()
    │
    ▼
provider.getObject()
    │
    ▼
Spring Container
    │
    ▼
New PrototypeBean
```

**Best when:** You want lazy and programmatic access.

##### ✅ Solution 3: `ObjectFactory<T>`

Older alternative to `ObjectProvider`.

```java
@Service
class UserService {

    @Autowired
    private ObjectFactory<PrototypeBean> factory;

    public void execute() {
        PrototypeBean bean = factory.getObject();
    }
}
```

```
factory.getObject()
        │
        ▼
Spring Container
        │
        ▼
New PrototypeBean
```

**Use when:** Minimal functionality is sufficient.

##### ✅ Solution 4: `Provider<T>` (JSR-330)

Framework-independent solution.

```java
import jakarta.inject.Provider;

@Service
class UserService {

    @Autowired
    private Provider<PrototypeBean> provider;

    public void execute() {
        PrototypeBean bean = provider.get();
    }
}
```

```
provider.get()
      │
      ▼
Spring Container
      │
      ▼
New PrototypeBean
```

**Best when:** Using standard Jakarta dependency injection.

##### ✅ Solution 5: `ApplicationContext` (Least Recommended)

Manually retrieve the bean.

```java
@Service
class UserService {

    @Autowired
    private ApplicationContext context;

    public void execute() {
        PrototypeBean bean =
            context.getBean(PrototypeBean.class);
    }
}
```

```
execute()
    │
    ▼
context.getBean(...)
    │
    ▼
Spring Container
```

**Avoid when possible:** Couples your code to the Spring framework.


---
# 7. Autowiring & `@Qualifier` / `@Primary`

### How `@Autowired` Resolves Candidates

1. **By type** — if exactly one bean of the required type exists, inject it.
2. **By name** — if multiple candidates exist, Spring tries to match the **field/parameter name** to a bean name.
3. If still ambiguous → throws `NoUniqueBeanDefinitionException`, unless disambiguated with `@Qualifier` or `@Primary`.

> More details about [[@Autowired]]

### `@Qualifier` — Pick a Specific Bean by Name/Label

```java
public interface PaymentGateway {}

@Component("stripeGateway")
public class StripePaymentGateway implements PaymentGateway {}

@Component("paypalGateway")
public class PaypalPaymentGateway implements PaymentGateway {}

@Component
public class CheckoutService {
    private final PaymentGateway gateway;

    public CheckoutService(@Qualifier("stripeGateway") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

### `@Primary` — Default Choice Among Multiple Candidates

```java
@Component
@Primary
public class StripePaymentGateway implements PaymentGateway {}
```

- Used when one implementation should be the **default** unless explicitly overridden with `@Qualifier` at the injection point.
- `@Qualifier` at the injection site always wins over `@Primary`.

### Optional Dependencies (Solves the issue of no bean found)

```java
@Autowired(required = false)
private AuditLogger auditLogger; // may remain null

@Autowired
private Optional<AuditLogger> auditLogger; // Optional.empty() if absent

@Autowired
private ObjectProvider<AuditLogger> auditLoggerProvider; // lazy, supports getIfAvailable()
```

### Injecting All Implementations (Solves the issue of multiple beans found)

```java
@Autowired
private List<PaymentGateway> allGateways; // every PaymentGateway bean, in order

@Autowired
private Map<String, PaymentGateway> gatewaysByName; // keyed by bean name
```

Use `@Order` or `Ordered` interface to control collection ordering.

---

# 8. Stereotype Annotations — `@Component` / `@Service` / `@Repository` / `@Controller`

All are **meta-annotated** with `@Component`, meaning the component scanner treats them identically for bean registration purposes — the differences are **semantic** (and in `@Repository`'s case, functional).

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {
    // ...
}
```

|Annotation|Layer|Extra Behavior|
|---|---|---|
|`@Component`|Generic|None — the base stereotype|
|`@Service`|Business/service layer|None (purely semantic marker)|
|`@Repository`|Persistence/DAO layer|✅ **Automatic exception translation** — native persistence exceptions (JDBC `SQLException`, JPA exceptions, etc.) are translated into Spring's unified `DataAccessException` hierarchy|
|`@Controller`|Web/MVC layer|Detected by `DispatcherServlet` as a request-handling component|
|`@RestController`|Web/REST layer|**Composed annotation** = `@Controller` + `@ResponseBody` (every handler method's return value is written directly to the response body)|

### Composed Annotations (Meta-Annotations)

A **meta-annotation** is an annotation applied to another annotation, letting you bundle behavior. `@RestController` is the classic example: it's composed of `@Controller` and `@ResponseBody`, so you don't have to add `@ResponseBody` to every method.

---

# 9. `@Configuration` & `@Bean` Vs. `@Component`

### Why the CGLIB Proxy Matters

The **CGLIB proxy** on `@Configuration` classes preserves Spring's bean lifecycle and scope semantics when one `@Bean` method calls another **within the same class**.

Instead of a plain Java method call, CGLIB **intercepts** the call and delegates to the Spring container:

- If the bean already exists (e.g., singleton) → returns the **existing managed instance**.
- Otherwise → creates, registers, and returns a new bean.

**Without CGLIB (plain `@Component`):**

```
@Bean method → Normal Java call → new Object()
```

**With CGLIB (`@Configuration`):**

```
@Bean method
   ↓
Intercepted by CGLIB
   ↓
Spring Container
   ↓
Existing bean?
   ├── Yes → Return managed bean
   └── No  → Create, register, and return bean
```

> [!important] Why `@Configuration` exists as a distinct annotation It guarantees that inter-`@Bean`-method calls respect the IoC container: bean scopes, lifecycle callbacks, and AOP proxies. Inside a plain `@Component`, `@Bean` methods are just ordinary Java calls — **no interception**, so calling another `@Bean` method directly creates a **new**, unmanaged object instead of returning the container-managed singleton.

### `@Configuration(proxyBeanMethods = false)`

- Introduced for performance: skips CGLIB proxying when you **don't** need inter-bean-method calls to go through the container (common in Spring Boot auto-configuration for faster startup).
- If you rely on calling one `@Bean` method from another to get the managed instance, you **must** keep `proxyBeanMethods = true` (the default).

### Gotchas

> [!warning] Method modifiers `@Bean` methods in a `@Configuration` class **cannot be `private` or `final`** — CGLIB can't override them. Use `protected` or `public`.

> [!warning] Static methods bypass CGLIB Calls to **static** `@Bean` methods are **never intercepted** by the container, even inside `@Configuration` classes. (This is also why `PropertySourcesPlaceholderConfigurer` factory methods must be `static` — they need to run very early, before normal bean post-processing.)

---

# 10. `@ComponentScan`

### Basic Usage

```java
@Configuration
@ComponentScan(basePackages = "com.example.app")
public class AppConfig { }
```

- Tells Spring where to look for classes annotated with `@Component` (and its meta-annotated variants: `@Service`, `@Repository`, `@Controller`, `@Configuration`, etc.) to register as beans.
- `@SpringBootApplication` implicitly includes `@ComponentScan` on the package of the main class (and its sub-packages).

### Filtering What Gets Scanned

```java
@ComponentScan(
    basePackages = "com.example.app",
    includeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = CustomAnnotation.class),
    excludeFilters = @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = LegacyService.class)
)
```

Common `FilterType` values: `ANNOTATION`, `ASSIGNABLE_TYPE`, `ASPECTJ`, `REGEX`, `CUSTOM`.


### Side Note
#### Class Inheritance ≠ Bean Metadata Inheritance

- Spring reads `@Scope` **only** from:
    - the concrete bean class (component-scanned beans), or
    - the `@Bean` factory method.
- Java class inheritance does **not** imply inheritance of Spring bean metadata.
- **Bean definition inheritance** exists only in **XML configuration**, not in annotation-based configuration.

---
# 11. `@Value` and Property Resolution

- Spring provides a **default lenient embedded value resolver**.
    - If a property can't be resolved, the **literal placeholder** (e.g. `${catalog.name}`) is injected as the value instead of failing.
- To enforce **strict resolution** (fail fast if a placeholder is missing), declare a `PropertySourcesPlaceholderConfigurer` bean:

```java
@Configuration
public class AppConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

> [!warning] Must be `static` When configuring `PropertySourcesPlaceholderConfigurer` via JavaConfig, the `@Bean` method **must be `static`**.

> [!info] Spring Boot default Spring Boot auto-configures a `PropertySourcesPlaceholderConfigurer` that reads from `application.properties` / `application.yml`.

### Customizing Placeholder Syntax

Available setters:

- `setPlaceholderPrefix()`
- `setPlaceholderSuffix()`
- `setValueSeparator()`
- `setEscapeCharacter()`

The default escape character can also be changed/disabled globally via the `spring.placeholder.escapeCharacter.default` JVM system property (or `SpringProperties` mechanism).

### Type Conversion for `@Value`

A `BeanPostProcessor` uses a `ConversionService` behind the scenes to convert `String` → target type.

To support custom types, register your own `ConversionService`:

```java
@Configuration
public class AppConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
        conversionService.addConverter(new MyCustomConverter());
        return conversionService;
    }
}
```

---

# 12. Background Bean Initialization (Spring 6.2+)

- `@Bean(bootstrap = BACKGROUND)` allows singling out specific beans for **background initialization**, covering the entire bean-creation step on context startup.
- Dependent beans with **non-lazy** injection points automatically **wait** for the background bean to complete.
- All regular background initializations are **forced to complete** by the end of context startup.
- Only beans additionally marked `@Lazy` may complete **later**, up until first actual access.

> [!tip] Pairs well with `@Lazy` Background initialization is typically combined with `@Lazy` (or `ObjectProvider`) injection points in dependent beans — otherwise the main bootstrap thread blocks when an early injection needs the background-initialized instance.

---

# 13. Spring 7 — Programmatic Bean Registration (`BeanRegistrar`)

### Purpose

- Introduced in **Spring Framework 7**.
- A **first-class, type-safe API** for registering beans programmatically.
- Replaces older, more complex APIs like `ImportBeanDefinitionRegistrar` for most use cases.

### When to Use

Use `BeanRegistrar` for **dynamic** registration:

- Based on active profiles
- Using loops
- From external configuration
- For infrastructure/library beans
- With conditional logic in plain Java

For **static** bean definitions, prefer `@Component` / `@Bean`.

### Registration Flow

```
ApplicationContext starts
        ↓
Process @Configuration
        ↓
Process @Import(MyBeanRegistrar.class)
        ↓
Call register(...)
        ↓
Register Bean Definitions
        ↓
Instantiate Beans
        ↓
Dependency Injection
        ↓
@PostConstruct
```

> [!important] `BeanRegistrar` registers **bean definitions**, not bean instances.

### Examples

**Basic registration**

```java
registry.registerBean(UserService.class);
```

Equivalent to:

```java
@Bean
UserService userService() {
    return new UserService();
}
```

**Custom bean name**

```java
registry.registerBean("foo", Foo.class);
```

Equivalent to:

```java
@Bean("foo")
Foo foo() {
    return new Foo();
}
```

**Customize bean metadata**

```java
registry.registerBean(
    Bar.class,
    spec -> spec
        .prototype()
        .lazyInit()
        .description("Custom description")
);
```

Configurable: scope, lazy init, description, supplier, qualifiers, etc.

**Custom instance creation (supplier)**

```java
registry.registerBean(
    Bar.class,
    spec -> spec.supplier(
        context -> new Bar(context.bean(Foo.class))
    )
);
```

Equivalent to:

```java
@Bean
Bar bar(Foo foo) {
    return new Bar(foo);
}
```

`context.bean(Foo.class)` ≈ `applicationContext.getBean(Foo.class)`

**Conditional registration**

```java
if (env.matchesProfiles("dev")) {
    registry.registerBean(DevService.class);
}
```

Equivalent to `@Profile`, but allows arbitrary Java logic (`if`, `for`, `switch`, ...).

**Registration in a loop**

```java
for (String tenant : tenants) {
    registry.registerBean(
        tenant + "Service",
        TenantService.class
    );
}
```

One of the biggest advantages over `@Bean` methods.

### AOT Support

Fully compatible with:

- Spring AOT
- JVM execution
- GraalVM Native Images

### Key Takeaways

- Introduced in Spring Framework 7.
- Registers bean **definitions** programmatically.
- Imported via `@Import(MyBeanRegistrar.class)`.
- Supports dynamic registration with plain Java (`if`, `for`, `switch`).
- Configures scope, lazy, supplier, qualifiers, etc.
- `context.bean(...)` retrieves other Spring-managed beans during registration.
- Runs **before** bean instantiation during container init.
- Best for dynamic, conditional, or library/infrastructure beans; use `@Component`/`@Bean` for regular app code.

---

# 14. Overloaded `@Bean` Methods

```java
@Bean
methodA()

@Bean
methodA(Dependency dependency)
```

Spring treats these as **one bean definition**, choosing between them via the **factory method/constructor resolution algorithm** based on available dependencies.

> [!danger] `@Profile` is NOT part of that resolution process Never use `@Profile` to distinguish overloaded `@Bean` methods — it won't work as expected.

### Correct Way to Define Profile-Specific Bean Variants

1. Use **different Java method names**.
2. Give them the **same bean name** via `@Bean("myBean")`.
3. Put **different `@Profile` conditions** on each method.

---

# 15. Property Source Precedence

### General Rule

The search is **hierarchical**. By default, **system properties** take precedence over **environment variables**.

- If `my-property` is set in both, `env.getProperty("my-property")` returns the **system property** value.
- Property values are **not merged** — a higher-precedence source **completely overrides** a lower one.

### General Spring Boot Precedence Table (highest → lowest)

| Precedence | Property Source                                            | Example                                          |
| ---------- | ---------------------------------------------------------- | ------------------------------------------------ |
| 1          | Command-line arguments                                     | `java -jar app.jar --server.port=9090`           |
| 2          | Java System Properties                                     | `-Dserver.port=9090`                             |
| 3          | OS Environment Variables                                   | `SERVER_PORT=9090`                               |
| 4          | `application.properties`/`.yml` outside the jar            | `/config/application.properties`                 |
| 5          | `application.properties`/`.yml` inside the jar (classpath) | `src/main/resources/application.properties`      |
| 6          | `@PropertySource`                                          | `@PropertySource("classpath:custom.properties")` |
| 7          | Default properties                                         | `SpringApplication.setDefaultProperties(...)`    |

---
