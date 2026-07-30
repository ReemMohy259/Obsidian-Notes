## General Testing Concepts

**1. Unit vs Integration vs End-to-End testing**

- **Unit test**: tests one class/method in isolation, dependencies mocked. Fast, no Spring context, no DB.
- **Integration test**: tests how multiple components work together (e.g., Service + real Repository + real DB). Slower, may load Spring context.
- **End-to-end (E2E) test**: tests the whole system as a user would, e.g., hitting a real HTTP endpoint that touches the full stack (controller → service → DB). Slowest, most realistic.

**2. Testing Pyramid** A pyramid shape describing test distribution: many fast **unit tests** at the bottom, fewer **integration tests** in the middle, and very few slow **E2E tests** at the top. It matters because unit tests are cheap and fast to run constantly, while E2E tests are expensive and brittle — you want confidence without paying too much in speed/maintenance.

**3. What makes a good unit test — FIRST principles**

- **F**ast — runs in milliseconds
- **I**solated — doesn't depend on other tests or external systems
- **R**epeatable — same result every time, any environment
- **S**elf-checking — pass/fail automatically, no manual inspection
- **T**imely — written close to when the code is written (ideally before/alongside it)

**4. Mock vs Stub vs Spy**

- **Stub**: a fake object that returns predefined answers to calls, doesn't care how it's used.
- **Mock**: like a stub, but you also verify _how_ it was used (was it called, how many times, with what args).
- **Spy**: wraps a real object — real methods run by default, but you can selectively override behavior or just observe calls made to it.

**5. TDD (Test-Driven Development)** A workflow: **Red → Green → Refactor**.

1. Write a failing test first (Red)
2. Write the minimum code to make it pass (Green)
3. Refactor the code while keeping tests green

Benefits: forces you to think about the API/requirements before implementation, and guarantees test coverage. If you haven't used it professionally, it's fine to say you understand the concept and have practiced it in personal projects.

**6. AAA pattern** Structure for writing a test body:

- **Arrange**: set up objects, inputs, mocks
- **Act**: call the method under test
- **Assert**: check the result matches expectations

**7. Black-box vs White-box testing**

- **Black-box**: testing based only on inputs/outputs, without knowing internal implementation (e.g., testing a REST API's contract).
- **White-box**: testing with knowledge of internal logic/structure (e.g., unit tests that know exactly which branches/methods to hit).

---

## JUnit Basics

**8. JUnit 4 vs JUnit 5**

- JUnit 5 (Jupiter) is modular: split into JUnit Platform, JUnit Jupiter (new API), JUnit Vintage (runs old JUnit 3/4 tests).
- Annotations changed: `@Before`→`@BeforeEach`, `@After`→`@AfterEach`, `@BeforeClass`→`@BeforeAll`, `@AfterClass`→`@AfterAll`.
- JUnit 5 supports `@ParameterizedTest`, better extension model (`@ExtendWith` replaces `@RunWith`), nested tests (`@Nested`), and doesn't require public no-arg constructors/methods.

**9. Lifecycle annotations**

- `@BeforeAll` — runs once before all tests in the class (must be static, unless using `PER_CLASS` lifecycle)
- `@BeforeEach` — runs before every test method (setup)
- `@Test` — marks a test method
- `@AfterEach` — runs after every test method (cleanup)
- `@AfterAll` — runs once after all tests finish

**10. Testing for thrown exceptions**

```java
@Test
void shouldThrowWhenUserNotFound() {
    assertThrows(UserNotFoundException.class, () -> userService.getUser(999L));
}
```

You can also capture the exception to assert on its message:

```java
Exception ex = assertThrows(UserNotFoundException.class, () -> userService.getUser(999L));
assertEquals("User not found", ex.getMessage());
```

**11. `@ParameterizedTest`** Lets you run the same test logic with different inputs, avoiding duplicate test methods.

```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "  "})
void shouldRejectBlankNames(String name) {
    assertFalse(validator.isValid(name));
}
```

Data sources: `@ValueSource`, `@CsvSource`, `@MethodSource`, `@EnumSource`.

**12. Disabling a test**

```java
@Disabled("Flaky - investigating race condition")
@Test
void someTest() { ... }
```

Useful temporarily, but should always have a reason/ticket attached — shouldn't be left forever.

**13. Assertion libraries**

- **JUnit built-in**: `assertEquals`, `assertTrue`, `assertThrows`, etc.
- **AssertJ**: fluent, readable assertions — `assertThat(result).isEqualTo(5).isPositive();`
- **Hamcrest**: matcher-based — `assertThat(result, is(equalTo(5)));`

AssertJ is very popular in Spring Boot projects because of its readability and rich matcher set for collections, exceptions, etc.

**14. `assertEquals` vs `assertSame`**

- `assertEquals(a, b)` — checks **value equality** (uses `.equals()`)
- `assertSame(a, b)` — checks **reference equality** (are they the exact same object in memory, like `==`)

---

## Mockito

**15. What is Mockito?** A mocking framework that lets you create fake ("mock") versions of dependencies so you can test a class in isolation without needing real databases, APIs, or other collaborators.

**16. `@Mock` vs `@InjectMocks`**

- `@Mock` — creates a fake/mock instance of a dependency.
- `@InjectMocks` — creates a real instance of the class under test and automatically injects the `@Mock`-annotated fields into its constructor/setters/fields.

```java
@Mock
private UserRepository userRepository;

@InjectMocks
private UserService userService; // real UserService, with the mock repository injected
```

**17. Stubbing return values**

```java
when(userRepository.findById(1L)).thenReturn(Optional.of(user));
```

For void methods that should throw:

```java
when(userRepository.findById(1L)).thenThrow(new RuntimeException("DB error"));
```

**18. Verifying calls**

```java
verify(userRepository).save(user);              // called exactly once
verify(userRepository, times(2)).save(user);    // called exactly twice
verify(userRepository, never()).delete(user);   // never called
```

**19. Mock vs Spy**

- `@Mock` — completely fake, all methods do nothing/return defaults unless stubbed.
- `@Spy` — wraps a **real object**; real methods actually execute unless you explicitly stub them.

```java
@Spy
private List<String> spyList = new ArrayList<>();
```

**20. Mocking a void method** Since void methods return nothing, you use `doX().when()` syntax instead of `when().thenX()`:

```java
doNothing().when(emailService).sendEmail(any());
doThrow(new RuntimeException()).when(emailService).sendEmail(any());
```

**21. `ArgumentCaptor`** Captures the actual argument passed into a mocked method call, so you can assert on it — useful when the object is created/modified inside the method under test.

```java
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(userRepository).save(captor.capture());
assertEquals("John", captor.getValue().getName());
```

**22. Verifying a method was never called**

```java
verify(userRepository, never()).delete(any());
```

Or, to confirm literally _no_ interactions happened with a mock at all:

```java
verifyNoInteractions(userRepository);
```

---

## Spring Boot Testing

**23. `@SpringBootTest`** Loads the **full Spring application context** (all beans, configuration, etc.) — closest to a real integration test. Use it when you genuinely need the whole context wired together. Avoid it for simple unit tests since it's slow — prefer plain Mockito unit tests or sliced tests (`@WebMvcTest`, `@DataJpaTest`) when you only need part of the context.

**24. `@SpringBootTest` vs `@WebMvcTest` vs `@DataJpaTest`**

- `@SpringBootTest` — loads the entire application context (full integration test).
- `@WebMvcTest` — loads only the web layer (controllers, `MockMvc`, JSON converters) — service/repository beans are **not** loaded, so you mock them with `@MockBean`.
- `@DataJpaTest` — loads only JPA-related components (repositories, entity manager), typically with an in-memory DB, and rolls back transactions after each test.

**25. `@MockBean` vs Mockito's `@Mock`**

- `@Mock` — plain Mockito annotation, creates a mock object with no relation to Spring.
- `@MockBean` — Spring Boot annotation that creates a mock **and registers it in the Spring application context**, replacing the real bean. Used when the class under test is wired via Spring (e.g., inside a `@WebMvcTest` controller test).

**26. Testing a REST controller with `MockMvc`** `MockMvc` simulates HTTP requests without starting a real server — faster than full integration tests.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.getUser(1L)).thenReturn(new User(1L, "John"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

**27. Testing the Repository layer** Use `@DataJpaTest`, which spins up an in-memory database (like H2) and configures Spring Data JPA automatically.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindByEmail() {
        userRepository.save(new User("john@test.com"));
        Optional<User> found = userRepository.findByEmail("john@test.com");
        assertTrue(found.isPresent());
    }
}
```

**28. `TestRestTemplate` / `WebTestClient`** Used in full `@SpringBootTest(webEnvironment = RANDOM_PORT)` tests to make **real HTTP calls** to a running (test) server instance — this is a true integration/E2E-style test.

- `TestRestTemplate` — for traditional Spring MVC (blocking).
- `WebTestClient` — for WebFlux (reactive), but also usable for regular MVC tests.

**29. Loading different test configuration**

- `application-test.yml` / `application-test.properties` — activated with a profile
- `@ActiveProfiles("test")` — tells Spring to use that profile's config during the test
- `@TestPropertySource(properties = {"some.key=value"})` — override specific properties inline in the test class

**30. `@ActiveProfiles("test")`** Activates the `test` Spring profile during the test run, so Spring loads `application-test.yml`/properties instead of the default — commonly used to point tests at an in-memory DB or disable certain beans (e.g., real email sending).

**31. Testing exception handling in a controller** Trigger the condition that causes the exception and verify the `@ExceptionHandler` response:

```java
@Test
void shouldReturn404WhenUserNotFound() throws Exception {
    when(userService.getUser(99L)).thenThrow(new UserNotFoundException("Not found"));

    mockMvc.perform(get("/users/99"))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.message").value("Not found"));
}
```

**32. Testcontainers** A library that spins up **real Docker containers** (e.g., a real PostgreSQL, Kafka, Redis instance) for integration tests, instead of using an in-memory substitute like H2. It solves the "works in H2 but breaks in real Postgres" problem — SQL dialects, constraints, and features can differ subtly from a real DB, so Testcontainers gives you higher-fidelity integration tests while still being automated and disposable.

---

## Practical / Scenario Questions

**33. Testing a Service that depends on a Repository** Mock the repository, inject it into the service, and verify behavior:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserWhenExists() {
        User user = new User(1L, "John");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        User result = userService.getUser(1L);

        assertEquals("John", result.getName());
        verify(userRepository).findById(1L);
    }

    @Test
    void shouldThrowWhenUserMissing() {
        when(userRepository.findById(1L)).thenReturn(Optional.empty());

        assertThrows(UserNotFoundException.class, () -> userService.getUser(1L));
    }
}
```

**34. Avoiding real DB/external API calls in tests**

- Unit tests: mock dependencies with Mockito (`@Mock`/`@MockBean`)
- Repository-layer tests: use an in-memory DB (H2) via `@DataJpaTest`, or Testcontainers for higher fidelity
- External APIs: mock the client/service layer, or use tools like **WireMock** to simulate HTTP responses

**35. Code coverage and 100%** Code coverage measures what percentage of code is executed by tests. It's a useful signal but **not a goal in itself** — 100% coverage doesn't mean the tests actually assert meaningful things (you can "cover" a line without checking its behavior is correct). It's better to aim for meaningful coverage of business logic and edge cases rather than chasing a percentage, especially skipping trivial code like simple getters/setters.

**36. Structuring tests for a CRUD REST API** A typical layered approach:

- **Unit tests** for the Service layer (mock the Repository) — covers business logic, validation, exceptions
- **Repository tests** with `@DataJpaTest` — covers custom queries, constraints
- **Controller tests** with `@WebMvcTest` + `MockMvc` (mock the Service) — covers request/response mapping, status codes, validation errors
- **A few integration/E2E tests** with `@SpringBootTest` — covers the full flow works end-to-end, catches wiring issues the other layers miss

**37. Investigating a flaky test** Common causes to check:

- **Shared/mutable state** between tests (e.g., a static field, or a database row not cleaned up) — tests should be independent
- **Timing issues** — async code, threads, or `Thread.sleep()` instead of proper waiting/synchronization
- **Order dependency** — a test relying on another test having run first
- **External dependencies** — hitting a real network resource, time-sensitive logic (e.g., `LocalDate.now()`)
- Fix strategies: isolate test data, use `@DirtiesContext` or transactional rollback, replace sleeps with `Awaitility`, mock time-dependent code

---

## Quick tips for the interview

- If asked something you genuinely don't know, it's fine to say "I haven't used that yet, but based on what I know it's probably used for X" — shows honest reasoning over guessing confidently.
- Be ready to **walk through code out loud** — many interviewers care more about your reasoning process than a memorized definition.
- If you have a personal project, mention it — "In my project, I tested the Service layer with Mockito and the Controller with MockMvc" is a strong, concrete answer.