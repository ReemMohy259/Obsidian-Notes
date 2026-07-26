## Table of Contents

- [[#1. Auto-Configuration (@SpringBootApplication)]]
- [[#2. Starters]]
- [[#3. Profiles (application-{profile}.yml)]]
- [[#4. External Configuration & @ConfigurationProperties]]
- [[#5. Spring Boot Actuator]]
- [[#6. Embedded Servers (Tomcat / Jetty / Undertow)]]
- [[#7. Packaging & Running (Fat Jar, Layered Jars)]]
- [[#8. Spring Boot DevTools]]
- [[#9. Logging]]
- [[#10. Testing in Spring Boot]]
- [[#11. Spring Boot CLI & Build Tool Plugins]]
- [[#12. Banner & Startup Customization]]
- [[#13. `spring.jpa.open-in-view`]]
- 14. [[#14. How do you externalize configuration in Spring Boot?]]

---

# 1. Auto-Configuration (`@SpringBootApplication`) 

### What Is Auto-Configuration?

Spring Boot inspects what's on the **classpath** and what beans **already exist** in the context, then automatically registers sensible-default beans for you — e.g. add `spring-boot-starter-data-jpa` + a `DataSource` on the classpath → Boot auto-configures a `DataSource`, `EntityManagerFactory`, and `TransactionManager` without you writing a single `@Bean`.

### `@SpringBootApplication` = Three Annotations in One

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

It is a **composed/meta-annotation** equivalent to:

| Annotation                 | Purpose                                                                       |
| -------------------------- | ----------------------------------------------------------------------------- |
| `@SpringBootConfiguration` | A specialization of `@Configuration` — marks this as the primary config class |
| `@EnableAutoConfiguration` | Turns on Spring Boot's auto-configuration mechanism                           |
| `@ComponentScan`           | Scans the package of this class (and sub-packages) for `@Component` beans     |

### How Auto-Configuration Actually Works

1. `@EnableAutoConfiguration` triggers Spring Boot to load candidate configuration classes listed in:
    - `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+/3.x)
    - _(Legacy: `META-INF/spring.factories` in Boot 2.x and earlier)_
2. Each candidate is an `@Configuration` class guarded by **`@Conditional`** annotations, so it only activates when appropriate.

### Key `@Conditional` Annotations Used Internally

| Annotation                     | Activates When                                                                                 |
| ------------------------------ | ---------------------------------------------------------------------------------------------- |
| `@ConditionalOnClass`          | A specific class is present on the classpath                                                   |
| `@ConditionalOnMissingClass`   | A specific class is **absent**                                                                 |
| `@ConditionalOnBean`           | A bean of a given type **already exists** in the context                                       |
| `@ConditionalOnMissingBean`    | No bean of that type exists yet — **the classic "back off if user defined their own" pattern** |
| `@ConditionalOnProperty`       | A specific property is set (optionally to a specific value)                                    |
| `@ConditionalOnWebApplication` | The app is a web application                                                                   |
| `@ConditionalOnResource`       | A specific resource (file) exists                                                              |

> [!important] The Golden Rule of Auto-Configuration Auto-configured beans are almost always annotated `@ConditionalOnMissingBean`. **If you define your own bean of the same type, Boot's default backs off automatically** — you never need to disable auto-config manually in the common case.

### Excluding Specific Auto-Configurations

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

or in `application.properties`:

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### Debugging What Got Auto-Configured

```properties
debug=true
```

Or run with `--debug`. Prints a full report of **positive matches** (applied) and **negative matches** (why something didn't apply) for every auto-configuration class — extremely useful when something "isn't working".

---

# 2. Starters

### What Are They?

**Starters** are curated dependency descriptors (just Maven/Gradle POMs with no code) that pull in a coherent, version-compatible set of libraries for a given purpose. They solve "dependency hell" by letting Spring Boot's **BOM (Bill of Materials)** manage compatible versions for you.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

This one line transitively brings in Spring MVC, an embedded Tomcat, Jackson (JSON), and validation — all at tested, mutually compatible versions.

### Common Starters

|Starter|Brings In|
|---|---|
|`spring-boot-starter-web`|Spring MVC + embedded Tomcat + Jackson|
|`spring-boot-starter-webflux`|Reactive stack (Project Reactor + Netty)|
|`spring-boot-starter-data-jpa`|Spring Data JPA + Hibernate + JDBC|
|`spring-boot-starter-data-mongodb`|Spring Data MongoDB|
|`spring-boot-starter-security`|Spring Security|
|`spring-boot-starter-test`|JUnit 5, Mockito, AssertJ, Spring Test|
|`spring-boot-starter-actuator`|Production monitoring endpoints|
|`spring-boot-starter-validation`|Bean Validation (Jakarta Validation / Hibernate Validator)|
|`spring-boot-starter-thymeleaf`|Thymeleaf templating engine|
|`spring-boot-starter-cache`|Caching abstraction support|

### `spring-boot-starter-parent` vs BOM Import

- **`spring-boot-starter-parent`** (Maven): inherit as your project's `<parent>` — gives you version management, sensible default plugin config, and resource filtering.
- **BOM import** (when you can't use a parent POM, or in Gradle):

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.x.x</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Writing a Custom Starter

Convention: `your-name-spring-boot-starter`. Typically two modules:

- `xxx-spring-boot-autoconfigure` — the actual `@Configuration` classes + `@ConditionalOn...` logic + `AutoConfiguration.imports` file.
- `xxx-spring-boot-starter` — an empty POM that just depends on the autoconfigure module (+ any required libraries).

---

# 3. Profiles (`application-{profile}.yml`)

### Purpose

Profiles let you maintain **environment-specific configuration** (dev, test, staging, prod) and swap between them without code changes.

### File Naming Convention

```
application.yml                  ← always loaded (common/default config)
application-dev.yml               ← loaded only when "dev" profile is active
application-prod.yml              ← loaded only when "prod" profile is active
```

Profile-specific files **override** matching keys from `application.yml`; anything not overridden falls back to the base file.

### Activating a Profile

```properties
# in application.yml
spring.profiles.active=dev
```

```bash
# command line (highest precedence — see property precedence table)
java -jar app.jar --spring.profiles.active=prod

# environment variable
export SPRING_PROFILES_ACTIVE=prod
```

Multiple profiles can be active at once: `--spring.profiles.active=prod,metrics`

> In **multiple profile case**, if we have the same property in multiple profiles, **last profile wins**
### Profile Groups (Spring Boot 2.4+)

```yaml
spring:
  profiles:
    group:
      production: "prod,prod-db,prod-cache"
```

Activating `production` activates the whole group.

### `@Profile` on Beans

```java
@Component
@Profile("dev")
public class FakePaymentGateway implements PaymentGateway { }

@Component
@Profile("!dev")   // negation — active on any profile EXCEPT dev
public class RealPaymentGateway implements PaymentGateway { }
```

> [!warning] Reminder from the Spring Core note `@Profile` cannot be used to disambiguate **overloaded `@Bean` methods** — see [[Spring Core#14. Overloaded @Bean Methods|Overloaded @Bean Methods]].

---

# 4. External Configuration & `@ConfigurationProperties`

### The Configuration Sources Hierarchy

Spring Boot layers many external configuration sources. Boot adds a few of its own on top: command-line args → JNDI → System properties → OS env vars → profile-specific files → base `application.yml`/`.properties` → `@PropertySource` → defaults.

### `@Value` vs `@ConfigurationProperties`

||`@Value("${...}")`|`@ConfigurationProperties`|
|---|---|---|
|Style|One property per field, scattered|**Type-safe, grouped** binding of a whole prefix|
|Relaxed binding|❌|✅ (`my-prop`, `myProp`, `MY_PROP` all bind to the same field)|
|Validation (`@Validated`)|❌|✅|
|Nested objects / Lists / Maps|Awkward|✅ Native support|
|IDE metadata / autocomplete|❌|✅ (with `spring-boot-configuration-processor`)|

### Example

```yaml
app:
  mail:
    host: smtp.example.com
    port: 587
    retries: 3
    recipients:
      - alice@example.com
      - bob@example.com
```

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {

    @NotBlank
    private String host;
    private int port;
    private int retries = 1; // default value
    private List<String> recipients = new ArrayList<>();

    // getters & setters (or use a record in Boot 3.x)
}
```

### Registering `@ConfigurationProperties` Classes

```java
@Configuration
@EnableConfigurationProperties(MailProperties.class)
public class MailConfig { }
```

Or, simpler — scan-based registration:

```java
@ConfigurationPropertiesScan  // put on @SpringBootApplication class or a @Configuration
```

Or mark the properties class itself as a component:

```java
@Component
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties { }
```

### Immutable Configuration Properties (Java Records, Boot 3.x)

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(String host, int port, List<String> recipients) { }
```

Constructor binding is used automatically when there's a single parameterized constructor (or use `@ConstructorBinding` explicitly pre–Boot 3.0).

### Enabling IDE Metadata & Autocomplete

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

Generates `META-INF/spring-configuration-metadata.json`, which powers autocomplete + docs in your IDE for custom properties.

---

# 5. Spring Boot Actuator

### What It Is

A production-readiness toolkit that exposes operational information about your running application — health, metrics, environment, mappings — over HTTP (and/or JMX).

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Key Endpoints

| Endpoint                             | Purpose                                                                       |
| ------------------------------------ | ----------------------------------------------------------------------------- |
| `/actuator/health`                   | Application health (UP/DOWN), aggregates health indicators (DB, disk, custom) |
| `/actuator/info`                     | Arbitrary app info you configure (`info.*` properties or `git.properties`)    |
| `/actuator/metrics`                  | Micrometer-backed metrics (JVM, HTTP requests, custom)                        |
| `/actuator/env`                      | All resolved `Environment` properties (⚠️ sensitive — secure it)              |
| `/actuator/beans`                    | Every bean in the `ApplicationContext`                                        |
| `/actuator/mappings`                 | All `@RequestMapping` routes                                                  |
| `/actuator/loggers`                  | View & **change log levels at runtime** without restart                       |
| `/actuator/threaddump` / `/heapdump` | Diagnostics for debugging live issues                                         |
| `/actuator/shutdown`                 | Gracefully shuts down the app (disabled by default)                           |

### Exposing Endpoints (most are hidden by default over HTTP)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
```

> [!danger] Security Never expose `env`, `beans`, `heapdump`, or `shutdown` publicly — they leak secrets or allow DoS. Secure Actuator endpoints separately (different port + Spring Security) in production.

### Custom Health Indicators

```java
@Component
public class DownstreamServiceHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean up = checkDownstream();
        return up ? Health.up().build()
                   : Health.down().withDetail("reason", "timeout").build();
    }
}
```

### Metrics via Micrometer

Actuator's `/metrics` is backed by **Micrometer**, a vendor-neutral metrics facade. Add `micrometer-registry-prometheus` (or another registry) to ship metrics to Prometheus, Datadog, CloudWatch, etc.

---

# 6. Embedded Servers (Tomcat / Jetty / Undertow)

### Why Embedded Servers?

Spring Boot apps are typically **executable JARs**, not WARs deployed to an external server — the HTTP server runs **inside** your JVM process. This is what enables `java -jar app.jar` to "just work".

### Default & Alternatives

|Server|Notes|
|---|---|
|**Tomcat** (default with `spring-boot-starter-web`)|Mature, most widely used, servlet-based|
|**Jetty**|Lightweight, good for WebSocket-heavy or resource-constrained apps|
|**Undertow**|Non-blocking, lower memory footprint, used often with reactive/high-throughput apps|
|**Netty** (with `spring-boot-starter-webflux`)|The reactive-stack default, not servlet-based at all|

### Switching Servers (Maven example)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

### Common Server Configuration

```yaml
server:
  port: 8443
  servlet:
    context-path: /api
  compression:
    enabled: true
  tomcat:
    threads:
      max: 200
    max-http-form-post-size: 2MB
```

### Servlet Stack vs Reactive Stack

||Servlet (`spring-boot-starter-web`)|Reactive (`spring-boot-starter-webflux`)|
|---|---|---|
|Model|Thread-per-request, blocking|Event-loop, non-blocking|
|Default server|Tomcat|Netty|
|Programming model|Spring MVC (`@Controller`)|Spring WebFlux (`Mono`/`Flux`)|
|Best for|Typical CRUD apps, blocking DB drivers|High-concurrency, streaming, backpressure-sensitive workloads|


---
# 7. Packaging & Running (Fat Jar, Layered Jars)

### Fat JAR vs Normal JAR

| Feature                       | Normal JAR | Fat (Uber/Executable) JAR |
| ----------------------------- | ---------- | ------------------------- |
| Contains application classes  | ✅          | ✅                         |
| Contains project dependencies | ❌          | ✅                         |
| Can run independently         | ❌          | ✅                         |
| Requires external classpath   | ✅          | ❌                         |
| Commonly used by              | Plain Java | Spring Boot               |

#### Normal JAR
Contains **only your application's compiled classes and resources**.

```
my-app.jar
├── com/example/...
└── application.properties
```

Run with external dependencies:
```bash
java -cp "my-app.jar:libs/*" com.example.Main
```

**Pros**
- Smaller file size.
- Dependencies managed separately.

**Cons**
- Must distribute dependency JARs alongside the application.

### Fat (Uber/Executable) JAR
`spring-boot-maven-plugin` / `spring-boot-gradle-plugin` repackage your app + **all dependencies** + an embedded server into a single runnable JAR:

```bash
mvn clean package
java -jar target/myapp-0.0.1-SNAPSHOT.jar
```

Internally it uses a nested-JAR loader (`JarLauncher`) so dependency JARs are embedded as-is inside `BOOT-INF/lib/`, not merged/shaded.

```
my-app.jar
├── BOOT-INF/classes
├── BOOT-INF/lib
│   ├── spring-core.jar
│   ├── spring-context.jar
│   ├── ...
└── META-INF
```

Run directly:
```bash
java -jar my-app.jar
```

**Pros**
- Single file deployment.
- No external dependencies required.
- Ideal for Docker and cloud deployment.

**Cons**
- Larger file size.
- Dependencies are bundled inside the JAR.

### Layered JARs (for Docker caching)

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers><enabled>true</enabled></layers>
    </configuration>
</plugin>
```

Splits the JAR into layers (dependencies, resources, application code) so Docker can cache the rarely-changing dependency layer separately from your frequently-changing code layer → much faster rebuilds.

```bash
java -Djarmode=layertools -jar app.jar extract
```

### Building Native Images (GraalVM)

```bash
mvn -Pnative native:compile
```

Produces a native executable with near-instant startup and lower memory — requires `spring-boot-starter-parent` + the `native-maven-plugin`, and code compatible with Spring AOT processing.

---
# 8. Spring Boot DevTools

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

- **Automatic restart** on classpath changes (uses two classloaders — a base one for libraries, a restart one for your code — so restarts are fast).
- **LiveReload** — auto-refreshes the browser on static resource changes.
- Disables template/cache in dev (Thymeleaf caching off, etc.) for instant feedback.
- Automatically **disabled in production** (detected via packaged JAR — no extra config needed).

---
# 9. Logging

### Default Setup

Spring Boot uses **Commons Logging** as the facade, backed by **Logback** by default (via `spring-boot-starter-logging`, pulled in transitively by every starter).

### Configuring via `application.yml`

```yaml
logging:
  level:
    root: INFO
    com.example.app: DEBUG
    org.springframework.web: WARN
  file:
    name: logs/app.log
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n"
```

### Switching Logging Framework (e.g. to Log4j2)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

### Runtime Log Level Changes

Via Actuator (see [[#5. Spring Boot Actuator]]):

```bash
curl -X POST localhost:8080/actuator/loggers/com.example.app \
     -H "Content-Type: application/json" \
     -d '{"configuredLevel":"DEBUG"}'
```

---

# 10. Testing in Spring Boot

|Annotation|Loads|Use Case|
|---|---|---|
|`@SpringBootTest`|Full `ApplicationContext`|Integration tests|
|`@WebMvcTest(MyController.class)`|Only web layer (MVC), mocks service layer|Controller-focused tests|
|`@DataJpaTest`|Only JPA/repository layer + in-memory DB|Repository tests|
|`@JsonTest`|Just JSON serialization components|Jackson mapping tests|
|`@MockBean`|Replaces a bean in the context with a Mockito mock|Isolating a unit within an integration test|
|`@TestConfiguration`|Extra/override beans just for tests|Custom test wiring|

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderControllerIT {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldReturnOrder() {
        var response = restTemplate.getForEntity("/orders/1", OrderDto.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

`spring-boot-starter-test` bundles: **JUnit 5**, **Mockito**, **AssertJ**, **Spring Test**, **JSONassert**, and **Hamcrest**.

---

# 11. Spring Boot CLI & Build Tool Plugins

- **Spring Boot CLI** — lets you run raw Groovy scripts as Spring Boot apps without boilerplate (`spring run app.groovy`); rarely used in production, mostly for quick prototyping/demos.
- **`spring-boot-maven-plugin`** — repackaging (`repackage` goal), running (`spring-boot:run`), building images (`spring-boot:build-image` — uses Cloud Native Buildpacks to produce a Docker image **without a Dockerfile**).
- **`spring-boot-gradle-plugin`** — Gradle equivalent (`bootRun`, `bootJar`, `bootBuildImage`).

```bash
mvn spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest
```

---

# 12. Banner & Startup Customization

- Custom ASCII banner: drop a `banner.txt` in `src/main/resources` (supports placeholders like `${application.version}`, `${spring-boot.version}`).
- Disable the banner:

```yaml
spring:
  main:
    banner-mode: "off"
```

- `SpringApplication` customization (headless mode, custom `ApplicationListener`s, lazy initialization globally):

```java
SpringApplicationBuilder builder = new SpringApplicationBuilder(MyApp.class);
builder.headless(true).run(args);
```

- **Global lazy initialization** (Boot 2.2+) — defers bean creation until first use, speeding up startup for apps with many beans:

```yaml
spring:
  main:
    lazy-initialization: true
```

---

# 13. `spring.jpa.open-in-view` 

`spring.jpa.open-in-view` controls the **Open Session in View (OSIV)** pattern.

When enabled, Spring keeps the **Hibernate `Session` / JPA `EntityManager` open for the entire HTTP request**, including the view rendering phase.

Normally:
```text
HTTP Request
      │
      ▼
Controller
      │
      ▼
Service (@Transactional)
      │
      ▼
Transaction Ends
      │
      ▼
EntityManager Closes
```

With **OSIV enabled**:
```text
HTTP Request
      │
      ▼
Controller
      │
      ▼
Service (@Transactional)
      │
      ▼
Transaction Ends
      │
      ▼
EntityManager remains open
      │
      ▼
View/JSON Serialization
      │
      ▼
EntityManager Closes
```

### Why is it useful?

It allows **lazy-loaded associations** to be initialized **outside the service layer**.

Example:

```java
@Entity
class User {

    @OneToMany(fetch = FetchType.LAZY)
    private List<Order> orders;
}
```

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

Without OSIV this will throws `LazyInitializationException`
### Drawbacks
- Database session remains open for the whole request.
- Can execute SQL queries during view rendering or JSON serialization.
- May hide poor application design (entities leaking outside the service layer).
- Can increase database connection usage.
- May cause unexpected N+1 query problems.
### Best Practice

For REST APIs:

```properties
spring.jpa.open-in-view=false
```

Instead:
- Fetch required data inside the service layer.
- Use DTOs.
- Use `JOIN FETCH` or `@EntityGraph` when needed.

---
# 14. How do you externalize configuration in Spring Boot?

External configuration allows you to **change application behavior without modifying or recompiling the code**.

Instead of hardcoding values:
```java
private String host = "localhost";
```

Use configuration:
```properties
database.host=localhost
```

### Common Configuration Sources (Highest → Lowest)
1. Command-line arguments (`--server.port=8081`)
2. JVM system properties (`-Dserver.port=8081`)
3. Environment variables (`SERVER_PORT=8081`)
4. External `application.properties` / `application.yml`
5. Internal `application.properties` / `application.yml`
6. `@PropertySource`
7. Default properties
### Ways to Access Configuration

* `@Value`
* `@ConfigurationProperties`
* `Environment`
---