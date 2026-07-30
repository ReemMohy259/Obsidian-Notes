## Table of Contents

- [[#1. Testing Fundamentals & the Test Pyramid]]
- [[#2. JUnit 5 (Jupiter) — Architecture]]
- [[#3. Core Test Annotations]]
- [[#4. Test Lifecycle]]
- [[#5. Assertions — Every Expression You'll Use]]
- [[#6. Assumptions]]
- [[#7. Parameterized Tests]]
- [[#8. Nested Tests, Tags & Test Suites]]
- [[#9. Test Doubles — Dummy, Fake, Stub, Spy, Mock]]
- [[#10. Mockito — Mocking & Stubbing]]
- [[#11. Mockito — Verification & Argument Matchers]]
- [[#12. AssertJ — Fluent Assertions]]
- [[#13. Spring Testing — Overview of Test Slices]]
- [[#14. Spring Testing — @SpringBootTest]]
- [[#15. Spring Testing — Web Layer (@WebMvcTest, MockMvc)]]
- [[#16. Spring Testing — Data Layer (@DataJpaTest)]]
- [[#17. Spring Testing — @MockBean vs @Mock, @SpyBean]]
- [[#18. Spring Testing — Testcontainers]]
- [[#19. Code Coverage & Test Quality]]
- [[#20. Quick Revision Cheat Sheet]]

---

## 1. Testing Fundamentals & the Test Pyramid

```
        ▲  Fewer, slower, more realistic
        │        ┌─────────────┐
        │        │   E2E/UI    │
        │        ├─────────────┤
        │        │ Integration │
        │        ├─────────────┤
        │        │  Unit Tests │
        │        └─────────────┘
        ▼  Many, fast, isolated
```

| Level           | Scope                                                  | Speed                  | Dependencies                                                        |
| --------------- | ------------------------------------------------------ | ---------------------- | ------------------------------------------------------------------- |
| **Unit**        | One class/method in isolation                          | Milliseconds           | Everything else **mocked**                                          |
| **Integration** | Multiple real components together (DB, Spring context) | Seconds                | Some real infra (often via [[#18. Spring Testing — Testcontainers]] |
| **E2E**         | The whole running application, as a user would use it  | Slow (seconds–minutes) | Real deployed system                                                |

> [!important] The core testing principle A **unit test** verifies one unit of behavior with every collaborator replaced by a [[#9. Test Doubles — Dummy, Fake, Stub, Spy, Mock|test double]] — this is what makes it fast, deterministic, and pinpoint-accurate about _what_ broke. Reach for more integration/E2E tests only for what genuinely needs multiple real components interacting (e.g. "does my repository's query actually work against a real database").

---

## 2. JUnit 5 (Jupiter) — Architecture

JUnit 5 is split into three sub-projects:

|Module|Role|
|---|---|
|**JUnit Platform**|The foundation — discovers and runs tests, launches any "engine" (including JUnit 4's, or others)|
|**JUnit Jupiter**|The modern **programming model** — the annotations/assertions you actually write (`@Test`, `assertEquals`, ...)|
|**JUnit Vintage**|A compatibility engine that runs **old JUnit 3/4** tests on the Platform|

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

Spring Boot's `spring-boot-starter-test` pulls this in automatically, along with Mockito, AssertJ, and Spring Test

---

## 3. Core Test Annotations

|Annotation|Purpose|
|---|---|
|`@Test`|Marks a method as a test case|
|`@DisplayName("...")`|Custom, human-readable name shown in test reports|
|`@Disabled("reason")`|Skips the test, with a required/recommended reason|
|`@RepeatedTest(n)`|Runs the same test `n` times — useful for flaky/timing-sensitive checks|
|`@Timeout(value = 2, unit = TimeUnit.SECONDS)`|Fails the test if it exceeds the given duration|
|`@Tag("slow")`|Categorizes tests for selective execution (see [[#8. Nested Tests, Tags & Test Suites]])|

```java
@Test
@DisplayName("should throw when withdrawing more than the balance")
void withdraw_insufficientFunds_throws() {
    Account account = new Account(100);
    assertThrows(InsufficientFundsException.class, () -> account.withdraw(200));
}
```

> [!tip] Naming convention `methodUnderTest_condition_expectedResult` (or similar) keeps test names scannable in a failure report without opening the file — you can see _what_ broke and _under what condition_ directly from the test runner output.

---

## 4. Test Lifecycle

|Annotation|Runs|Typical Use|
|---|---|---|
|`@BeforeAll`|**Once**, before all tests in the class (method must be `static`, unless the class uses `@TestInstance(PER_CLASS)`)|Expensive one-time setup (e.g. starting a shared resource)|
|`@BeforeEach`|Before **every** test method|Resetting/creating fresh test fixtures — the far more common choice|
|`@AfterEach`|After **every** test method|Per-test cleanup|
|`@AfterAll`|**Once**, after all tests (also `static` by default)|Tearing down shared resources|

```java
class AccountTest {

    private Account account;

    @BeforeEach
    void setUp() {
        account = new Account(100);       // fresh instance for EVERY test — avoids test interdependence
    }

    @Test
    void deposit_increasesBalance() {
        account.deposit(50);
        assertEquals(150, account.getBalance());
    }
}
```

```
@BeforeAll (once)
   │
   ├─ @BeforeEach → @Test #1 → @AfterEach
   ├─ @BeforeEach → @Test #2 → @AfterEach
   └─ @BeforeEach → @Test #3 → @AfterEach
   │
@AfterAll (once)
```

> [!warning] Tests must be independent Each test should pass or fail **regardless of execution order** or of any other test having run first. Shared mutable state between tests (a static field, a field not reset in `@BeforeEach`) is a classic source of tests that pass individually but fail when run together.

---

## 5. Assertions — Every Expression You'll Use

All from `org.junit.jupiter.api.Assertions` (static import: `import static org.junit.jupiter.api.Assertions.*;`).

### Equality & Identity

```java
assertEquals(expected, actual);
assertEquals(expected, actual, "custom failure message");
assertEquals(3.14159, actual, 0.001);           // delta — for floating-point comparisons
assertNotEquals(unexpected, actual);
assertSame(obj1, obj2);                           // reference equality (==)
assertNotSame(obj1, obj2);
```

### Boolean & Null Checks

```java
assertTrue(condition);
assertFalse(condition);
assertNull(value);
assertNotNull(value);
```

### Arrays & Collections

```java
assertArrayEquals(expectedArray, actualArray);
assertIterableEquals(expectedIterable, actualIterable);
assertLinesMatch(expectedLines, actualLines);      // for comparing lists of text lines, supports regex per-line
```

### Exceptions

```java
Exception ex = assertThrows(InsufficientFundsException.class, () -> account.withdraw(200));
assertEquals("Insufficient funds", ex.getMessage());

assertDoesNotThrow(() -> account.withdraw(50));
```

### Grouped / Multiple Assertions

```java
assertAll(
    () -> assertEquals("Alice", person.getName()),
    () -> assertEquals(30, person.getAge()),
    () -> assertTrue(person.isActive())
);
```

> [!important] `assertAll` runs every assertion, even if one fails Unlike writing separate `assertX` calls sequentially (where the **first** failure aborts the test immediately, hiding any later assertions), `assertAll` executes **all** of them and reports **every** failure together — much more useful when several related properties need checking on one test run.

### Timeouts (as an assertion, alternative to `@Timeout`)

```java
assertTimeout(Duration.ofMillis(100), () -> slowMethod());
assertTimeoutPreemptively(Duration.ofMillis(100), () -> slowMethod());  // also aborts execution on timeout
```

---

## 6. Assumptions

`Assumptions` (from `org.junit.jupiter.api.Assumptions`) **skip** a test (rather than fail it) when a precondition isn't met — useful for environment-dependent tests.

```java
@Test
void onlyOnCI() {
    assumeTrue("CI".equals(System.getenv("ENV")));
    // rest of the test only runs if the assumption holds
}

@Test
void conditionalAssertion() {
    assumingThat(isDatabaseAvailable(),
        () -> assertEquals(5, repository.count()));
}
```

Difference from an assertion: a **failed assumption** marks the test as **skipped** (neutral), not **failed** (a real problem) — the test report distinguishes these outcomes.

---

## 7. Parameterized Tests

Run the same test logic against **many inputs** without duplicating the test method.

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 5, 8, 13})
void isFibonacci_returnsTrue(int number) {
    assertTrue(FibonacciChecker.isFibonacci(number));
}
```

| Source Annotation                                      | Provides                                                                                      |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `@ValueSource(ints/strings/...)`                       | A simple array of literal values                                                              |
| `@CsvSource({"1,1", "2,4", "3,9"})`                    | Multiple comma-separated arguments per invocation                                             |
| `@MethodSource("methodName")`                          | A static method returning a `Stream`/`Collection` of arguments — for complex/object arguments |
| `@EnumSource(MyEnum.class)`                            | Every constant of an enum                                                                     |
| `@NullSource` / `@EmptySource` / `@NullAndEmptySource` | Explicitly test `null`/empty-string edge cases                                                |

```java
@ParameterizedTest
@CsvSource({
    "2, 3, 5",
    "10, 20, 30",
    "-1, 1, 0"
})
void add_returnsSum(int a, int b, int expected) {
    assertEquals(expected, Calculator.add(a, b));
}
```

```java
@ParameterizedTest
@MethodSource("provideOrders")
void isValid_ordersWithPositiveTotal(Order order, boolean expected) {
    assertEquals(expected, order.isValid());
}

static Stream<Arguments> provideOrders() {
    return Stream.of(
        Arguments.of(new Order(100), true),
        Arguments.of(new Order(-5), false)
    );
}
```

---

## 8. Nested Tests, Tags & Test Suites

### `@Nested` — Grouping Related Tests

```java
class AccountTest {

    @Nested
    class WhenBalanceIsZero {
        @Test
        void withdraw_throws() { /* ... */ }
    }

    @Nested
    class WhenBalanceIsPositive {
        @Test
        void withdraw_succeeds() { /* ... */ }
    }
}
```

Produces clearer, hierarchical test reports ("AccountTest > WhenBalanceIsZero > withdraw_throws") and lets each context share its own `@BeforeEach` setup.

### `@Tag` — Selective Execution

```java
@Tag("slow")
@Test
void expensiveIntegrationCheck() { /* ... */ }
```

Build tools (Maven/Gradle) can then run/exclude tests by tag (e.g. skip `slow` tests during a fast local build, run them only in CI).

### `@TestMethodOrder` — Controlling Order (Use Sparingly)

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderedTest {
    @Test @Order(1) void first() { }
    @Test @Order(2) void second() { }
}
```

> [!warning] Order-dependent tests are usually a smell Needing explicit ordering often signals hidden shared state between tests — revisit [[#4. Test Lifecycle]]'s independence principle before reaching for this.

---

## 9. Test Doubles — Dummy, Fake, Stub, Spy, Mock

A test double is ==a generic term for any object that replaces a real production object during software testing==, similar to a stunt double in a movie. The primary types are dummies, fakes, stubs, spies, and mocks, which help isolate code to make tests fast and reliable

|Type|Behavior|Example|
|---|---|---|
|**Dummy**|Passed around to satisfy a parameter list, never actually used|`new Order(null)` where the constructor requires _something_ but the test doesn't touch it|
|**Fake**|A working, simplified implementation — not production-grade, but behaves correctly|An in-memory `Map`-backed repository standing in for a database|
|**Stub**|Returns **pre-programmed** answers to calls, no real logic|`when(repo.findById(1)).thenReturn(order)`|
|**Spy**|A real object, wrapped so calls can be **observed/verified**, while still delegating to real behavior by default|`spy(realObject)` — see [[#10. Mockito — Mocking & Stubbing]]|
|**Mock**|A double that **verifies interactions** — did this method get called, how many times, with what arguments|`verify(mock).save(order)`|

#### Why Use Test Doubles
- **Isolation**: Separates the code you are testing from external services, networks, or slow databases.
- **Speed**: Removes heavy operations so your unit tests run fast.
- **Determinism**: Ensures tests always get the exact data they expect without random or changing external factors.

> [!tip] The distinction that matters day-to-day **Stub** = "control what it returns." **Mock** = "verify what was called on it." Mockito's `mock()` objects can do both simultaneously — the "stub vs mock" label really describes **how you're using** the double in a given test, not two different Mockito APIs.

---

## 10. Mockito — Mocking & Stubbing

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
</dependency>
```

### Creating Mocks

```java
OrderRepository repository = mock(OrderRepository.class);
```

Or, with the extension enabled (`@ExtendWith(MockitoExtension.class)`):

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository repository;      // auto-created mock

    @InjectMocks
    private OrderService orderService;         // real object, with @Mock fields injected automatically
}
```

### Stubbing Behavior

```java
when(repository.findById(1L)).thenReturn(Optional.of(order));
when(repository.findById(99L)).thenReturn(Optional.empty());
when(repository.save(any())).thenThrow(new DataAccessException("DB down"));

when(repository.findById(anyLong()))
    .thenReturn(Optional.of(order1))
    .thenReturn(Optional.of(order2));          // successive calls return different values, in order
```

### Stubbing `void` Methods

```java
doNothing().when(emailService).send(any());
doThrow(new RuntimeException()).when(emailService).send(any());
```

`when(...).thenReturn(...)` doesn't work for `void` methods — Mockito needs the `do...().when(mock)....` form instead, since there's no return value to hang the stub off of.

### Spies — Partial Mocking

```java
List<String> spyList = spy(new ArrayList<>());
spyList.add("real item");                      // actually executes on the real object
verify(spyList).add("real item");                // AND can be verified

doReturn(99).when(spyList).size();                // override just ONE method's behavior
```

---

## 11. Mockito — Verification & Argument Matchers

### Verifying Interactions

```java
verify(repository).save(order);                     // called exactly once, with this exact argument
verify(repository, times(2)).save(any());
verify(repository, never()).delete(any());
verify(repository, atLeastOnce()).findById(any());
verify(repository, atLeast(2)).save(any());
verify(repository, atMost(3)).save(any());

verifyNoInteractions(repository);                     // NOTHING was called on this mock at all
verifyNoMoreInteractions(repository);                 // no OTHER calls beyond ones already verified
```

### Argument Matchers

```java
verify(repository).save(argThat(o -> o.getTotal() > 100));
verify(repository).findById(eq(5L));
verify(repository).save(any(Order.class));
```

> [!danger] Matchers must be used consistently If **any** argument in a call uses a matcher (`any()`, `eq()`, `argThat()`), **all** arguments in that same call must use matchers too — mixing a raw literal with a matcher in the same call throws `InvalidUseOfMatchersException`.
> 
> ```java
> verify(repository).update(eq(5L), any(Order.class));   // OK — both are matchers
> verify(repository).update(5L, any(Order.class));         // WRONG — mixes raw value with a matcher
> ```

### Capturing Arguments for Deeper Inspection

```java
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(repository).save(captor.capture());

Order savedOrder = captor.getValue();
assertEquals("PENDING", savedOrder.getStatus());
```

Useful when you need to inspect the **actual object** a collaborator was called with, beyond what a matcher can express in-line.

### Ordering

```java
InOrder inOrder = inOrder(repository, emailService);
inOrder.verify(repository).save(any());
inOrder.verify(emailService).send(any());        // must have been called AFTER repository.save
```

---

## 12. AssertJ — Fluent Assertions

`spring-boot-starter-test` also bundles **AssertJ**, offering a more readable, chainable alternative to JUnit's `assertX` static methods.

```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import static org.assertj.core.api.Assertions.assertThat;

assertThat(account.getBalance()).isEqualTo(100);
assertThat(name).isNotNull().startsWith("A").hasSize(5);

assertThat(orders)
    .hasSize(3)
    .extracting(Order::getStatus)
    .containsExactly("PENDING", "SHIPPED", "DELIVERED");

assertThat(orders).filteredOn(o -> o.getTotal() > 100).isNotEmpty();

assertThatThrownBy(() -> account.withdraw(200))
    .isInstanceOf(InsufficientFundsException.class)
    .hasMessage("Insufficient funds");
```

> [!tip] Why teams prefer AssertJ The **fluent, chainable** style reads closer to natural language and gives noticeably better failure messages out of the box (e.g. showing exactly which elements of a collection didn't match), compared to a plain `assertEquals(expected, actual)` which just prints two raw values.

---

## 13. Spring Testing — Overview of Test Slices

Loading the **full** Spring context for every test is slow — Spring Boot provides **test slices** that load only the beans relevant to one architectural layer.

|Annotation|Loads|Use For|
|---|---|---|
|`@SpringBootTest`|Full application context|True integration tests|
|`@WebMvcTest`|Web layer only (controllers, `@ControllerAdvice`, converters)|Controller tests|
|`@DataJpaTest`|JPA/repository layer + in-memory DB|Repository tests|
|`@JsonTest`|Just Jackson-related beans|Serialization tests|
|`@RestClientTest`|REST client infrastructure (`RestTemplate`/`RestClient`)|Testing outbound HTTP calls|
|`@WebFluxTest`|WebFlux equivalent of `@WebMvcTest`|Reactive controller tests|

This directly mirrors the [[#1. Testing Fundamentals & the Test Pyramid|test pyramid]] — slices sit between pure unit tests (no Spring at all) and a full `@SpringBootTest` integration test.

---

## 14. Spring Testing — @SpringBootTest

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository repository;

    @Test
    void placeOrder_persistsAndReturnsCreated() {
        var response = restTemplate.postForEntity("/orders", new OrderRequest(...), OrderDto.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(repository.count()).isEqualTo(1);
    }
}
```

### `webEnvironment` Options

|Value|Behavior|
|---|---|
|`MOCK` (default)|No real server — mocked servlet environment, use `MockMvc`|
|`RANDOM_PORT`|Real embedded server on a random free port — use `TestRestTemplate`/`WebTestClient`|
|`DEFINED_PORT`|Real embedded server on the configured port|
|`NONE`|No web environment at all|

### Test-Specific Configuration

```java
@SpringBootTest
@ActiveProfiles("test")                             // loads application-test.yml
@TestPropertySource(properties = "feature.x.enabled=true")
class MyIntegrationTest { }
```

```java
@TestConfiguration
static class TestConfig {
    @Bean
    public Clock fixedClock() {
        return Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneOffset.UTC);
    }
}
```

---

## 15. Spring Testing — Web Layer (@WebMvcTest, MockMvc)


```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    void getOrder_returnsOrder() throws Exception {
        when(orderService.findById(1L)).thenReturn(Optional.of(new OrderDto(1L, "Widget")));

        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("Widget"));
    }

    @Test
    void createOrder_withInvalidBody_returns400() throws Exception {
        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{}"))
               .andExpect(status().isBadRequest());
    }
}
```

### Common `MockMvc` Expressions

|Expression|Checks|
|---|---|
|`status().isOk()` / `.isCreated()` / `.isNotFound()` / `.isBadRequest()`|HTTP status code|
|`jsonPath("$.field").value(x)`|A specific field in the JSON response body|
|`content().contentType(MediaType.APPLICATION_JSON)`|Response `Content-Type`|
|`header().string("Location", value)`|A specific response header|
|`content().json(expectedJson)`|Whole-body JSON comparison|

---

## 16. Spring Testing — Data Layer (@DataJpaTest)

```java
@DataJpaTest
class OrderRepositoryTest {

    @Autowired
    private OrderRepository repository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void findByStatus_returnsMatchingOrders() {
        entityManager.persist(new Order("PENDING"));
        entityManager.persist(new Order("SHIPPED"));

        List<Order> pending = repository.findByStatus("PENDING");

        assertThat(pending).hasSize(1);
    }
}
```

- Configures an **in-memory database** (H2, by default) unless you override it — and rolls back each test's transaction automatically (`@Transactional` is applied implicitly), so tests don't leak data into each other.
- `TestEntityManager` is a testing-focused wrapper giving direct access to persist/flush entities without going through the repository under test — useful for setting up fixture data independently of the code being tested.

> [!warning] In-memory DB ≠ your production DB `@DataJpaTest`'s default H2 database can behave subtly differently from PostgreSQL/MySQL (SQL dialect quirks, case sensitivity, date handling). For genuinely trustworthy repository tests, prefer [[#18. Spring Testing — Testcontainers|Testcontainers]] running your **actual** database engine.

---

## 17. Spring Testing — @MockBean vs @Mock, @SpyBean

||`@Mock` (Mockito)|`@MockBean` (Spring Boot)|
|---|---|---|
|Registers in Spring context?|No — plain Mockito object|**Yes** — replaces the real bean of that type in the `ApplicationContext`|
|Use with|Plain unit tests, no Spring context loaded|`@SpringBootTest`, `@WebMvcTest`, or any Spring-context test|
|Injected via|`@InjectMocks`, or manual constructor wiring|Spring's normal `@Autowired`|

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @MockBean
    private OrderService orderService;    // REPLACES the real OrderService bean in this test's context
}
```

```java
@SpringBootTest
class OrderServiceIntegrationTest {
    @SpyBean
    private OrderRepository repository;    // real bean, but wrapped so calls can be verified/partially stubbed
}
```

> [!tip] Rule of thumb **No Spring context involved at all** → plain Mockito `@Mock`/`@ExtendWith(MockitoExtension.class)`. **Testing inside a Spring test slice/context** → `@MockBean`/`@SpyBean`, so the fake actually gets wired into the real dependency graph Spring manages.

---

## 18. Spring Testing — Testcontainers

**Testcontainers** spins up **real** Dockerized dependencies (PostgreSQL, Kafka, Redis, ...) for the duration of a test run — closing the gap left by in-memory substitutes like H2.

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository repository;

    @Test
    void savesAndRetrievesOrder() {
        repository.save(new Order("PENDING"));
        assertThat(repository.findAll()).hasSize(1);
    }
}
```

- The container starts once (per class, with `static @Container`), runs against your **actual** database engine and version, and is torn down automatically after the test class finishes.
- Directly relevant to contract testing and integration testing in a microservices architecture — Testcontainers is the standard way to get genuinely trustworthy integration tests without a shared, flaky external test environment.

---

## 19. Code Coverage & Test Quality

- **Code coverage** (line/branch coverage, typically measured via **JaCoCo**) tells you what percentage of code was **executed** during tests — it says nothing about whether the _right assertions_ were made.
- **100% coverage is not the goal.** A test that calls a method but asserts nothing meaningful about the result contributes to coverage while providing near-zero actual protection against bugs.
- **Mutation testing** (e.g. **PIT/Pitest**) is a stronger signal: it deliberately introduces small bugs ("mutants") into your code and checks whether your test suite actually **fails** as a result — a test suite that still passes against a mutated, broken version of the code isn't really testing anything.

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
</plugin>
```

> [!tip] What actually matters Favor tests that cover **meaningful behavior and edge cases** (boundary values, error paths, empty/null inputs) over chasing a coverage percentage. Coverage is a useful _diagnostic_ for finding untested code paths — not a target to optimize directly.

---

## 20. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|Test pyramid|Many unit tests, fewer integration, fewest E2E|
|JUnit 5 modules|Platform (runs), Jupiter (write tests here), Vintage (JUnit 3/4 compat)|
|`@BeforeEach`/`@AfterEach`|Per-test setup/teardown — the common case|
|`@BeforeAll`/`@AfterAll`|Once per class, `static` by default|
|`assertAll(...)`|Runs every assertion, reports all failures together|
|`assertThrows`|Verify an exception is thrown, and inspect it|
|`Assumptions`|Skip (not fail) when a precondition isn't met|
|`@ParameterizedTest`|One test method, many inputs — `@ValueSource`/`@CsvSource`/`@MethodSource`|
|Stub vs Mock|Stub = control return values; Mock = verify interactions|
|`@Mock` + `@InjectMocks`|Plain Mockito, no Spring context|
|`verify(mock, times(n))`|Confirms exact call count|
|Matchers|All-or-nothing per call — never mix a matcher with a raw literal|
|`ArgumentCaptor`|Inspect the actual object passed to a mocked call|
|AssertJ|Fluent, chainable, better failure messages than plain JUnit asserts|
|`@SpringBootTest`|Full context — true integration test|
|`@WebMvcTest`|Web layer only, use with `MockMvc`|
|`@DataJpaTest`|Repository layer, in-memory DB, auto-rollback per test|
|`@MockBean`|Replaces a bean **inside the Spring context** — use when Spring is involved|
|`@Mock`|Plain Mockito double — use when Spring is **not** involved|
|Testcontainers|Real Dockerized dependencies — trustworthy over in-memory substitutes|
|Coverage|A diagnostic, not a target — mutation testing is a stronger signal of real test quality|

---
