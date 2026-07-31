---
share_link: https://share.note.sx/4tkallsr#TGK5WVC3/uMdxYEu6Yrgng
share_updated: 2026-07-31T04:07:11+03:00
---

| **Term**                         | **Definition**                                                           | **Example**                           |
| -------------------------------- | ------------------------------------------------------------------------ | ------------------------------------- |
| **Test**                         | The process of verifying software behaves as expected.                   | Testing the login feature             |
| **Test Case**                    | A single test that verifies one specific behavior.                       | `shouldCreateUser()`                  |
| **Test Suite**                   | A collection of test cases or test classes executed together.            | `UserServiceTest`, `OrderServiceTest` |
| **Test Scenario**                | A high-level business requirement to test.                               | User logs in successfully             |
| **Test Plan**                    | A document describing testing strategy, scope, schedule, and objectives. | Sprint testing plan                   |
| **Test Script**                  | Automated or manual steps used to execute a test.                        | Selenium script                       |
| **Test Fixture**                 | The environment and objects prepared before a test runs.                 | Creating mocks and test data          |
| **Setup**                        | Code executed before each test.                                          | `@BeforeEach`                         |
| **Teardown**                     | Cleanup executed after each test.                                        | `@AfterEach`                          |
| **Assertion**                    | Verifies the expected outcome.                                           | `assertEquals(5, result)`             |
| **Expected Result**              | What the test should produce.                                            | Status = 200                          |
| **Actual Result**                | What the application actually produced.                                  | Status = 500                          |
| **Test Oracle**                  | Mechanism that determines whether a test passed or failed.               | Assertions                            |
| **SUT (System Under Test)**      | The software/component currently being tested.                           | `OrderService`                        |
| **CUT (Class Under Test)**       | The specific class being tested.                                         | `UserService`                         |
| **AUT (Application Under Test)** | The complete application under test.                                     | Entire Spring Boot application        |

---

## Test Doubles

|**Term**|**Purpose**|**Example**|
|---|---|---|
|**Test Double**|Generic term for replacing a real dependency during testing.|Repository replacement|
|**Dummy**|Placeholder object that is never used.|Unused constructor parameter|
|**Stub**|Returns predefined values.|Repository always returns the same user|
|**Fake**|Simplified working implementation.|In-memory repository|
|**Mock**|Verifies interactions and expectations.|`verify(emailService).sendEmail()`|
|**Spy**|Wraps a real object and records interactions.|`spy(new ArrayList<>())`|

---

## Testing Levels

|**Term**|**Purpose**|**Example**|
|---|---|---|
|**Unit Test**|Tests one class in isolation.|Testing `OrderService` with mocked dependencies|
|**Integration Test**|Tests interaction between multiple components.|Service + Repository + PostgreSQL|
|**System Test**|Tests the complete integrated application.|Entire backend|
|**End-to-End (E2E) Test**|Tests the entire user workflow.|Browser → Backend → Database|
|**Acceptance Test (UAT)**|Confirms the system meets business requirements.|Customer acceptance testing|

---

## Testing Types

|**Term**|**Purpose**|**Example**|
|---|---|---|
|**Functional Testing**|Verifies business functionality.|User registration|
|**Non-Functional Testing**|Verifies quality attributes.|Performance testing|
|**Regression Testing**|Ensures existing features still work after changes.|Running all tests after a refactor|
|**Smoke Testing**|Verifies the application's critical functionality works.|Application starts and login works|
|**Sanity Testing**|Verifies a specific bug fix or feature.|Testing only the fixed payment bug|
|**Performance Testing**|Measures speed and responsiveness.|Response time under load|
|**Load Testing**|Measures behavior under expected user load.|1,000 concurrent users|
|**Stress Testing**|Pushes the system beyond its limits.|100,000 concurrent users|
|**Security Testing**|Checks for vulnerabilities.|SQL Injection, XSS|
|**Usability Testing**|Evaluates user experience.|User interface testing|

---

## Test Design Techniques

|**Term**|**Purpose**|**Example**|
|---|---|---|
|**Happy Path**|Tests normal successful execution.|Valid login|
|**Negative Testing**|Tests invalid input or failures.|Wrong password|
|**Edge Case Testing**|Tests uncommon or extreme values.|Empty string|
|**Boundary Value Analysis**|Tests values at the limits.|Age = 0, 1, 100, 101|
|**Equivalence Partitioning**|Groups similar inputs into partitions.|Ages 18–60|
|**Decision Table Testing**|Tests combinations of conditions.|Discount rules|
|**State Transition Testing**|Tests changes between states.|Order: Pending → Paid → Shipped|

---

## Code Quality & Coverage

|**Term**|**Purpose**|**Example**|
|---|---|---|
|**Code Coverage**|Percentage of code executed by tests.|85% coverage|
|**Statement Coverage**|Every statement executes at least once.|All lines executed|
|**Branch Coverage**|Every decision branch is tested.|`if` true and false|
|**Path Coverage**|Every execution path is tested.|All possible branches|
|**Mutation Testing**|Introduces bugs to evaluate test quality.|PIT Mutation Testing|
|**Cyclomatic Complexity**|Measures the number of execution paths.|Complexity = 10|

---

## Test Execution Concepts

|**Term**|**Purpose**|**Example**|
|---|---|---|
|**AAA Pattern**|Arrange → Act → Assert|Standard unit test structure|
|**BDD (Behavior-Driven Development)**|Given → When → Then|Cucumber tests|
|**TDD (Test-Driven Development)**|Red → Green → Refactor|Write test before code|
|**Parameterized Test**|Same test with multiple inputs.|`@ParameterizedTest`|
|**Test Isolation**|Tests should not depend on each other.|Independent unit tests|
|**Flaky Test**|Sometimes passes and sometimes fails.|Timing-dependent tests|
|**Deterministic Test**|Always produces the same result.|Pure function tests|

---

## JUnit Terminology

|**Term**|**Description**|
|---|---|
|`@Test`|Marks a test method|
|`@BeforeEach`|Runs before every test|
|`@AfterEach`|Runs after every test|
|`@BeforeAll`|Runs once before all tests|
|`@AfterAll`|Runs once after all tests|
|`@Disabled`|Skips a test|
|`@DisplayName`|Custom test name|
|`@Nested`|Groups related tests|
|`@RepeatedTest`|Runs a test multiple times|
|`@ParameterizedTest`|Executes one test with multiple inputs|

---

## Mockito Terminology

|**Term**|**Purpose**|
|---|---|
|`mock()`|Creates a mock object|
|`spy()`|Creates a spy around a real object|
|`when()`|Defines stubbed behavior|
|`thenReturn()`|Specifies return value|
|`doReturn()`|Stubs spies without calling the real method|
|`verify()`|Verifies interactions|
|`times()`|Verifies invocation count|
|`never()`|Verifies a method was never called|
|`any()`|Matches any argument|
|`ArgumentCaptor`|Captures arguments passed to a method|
|`InOrder`|Verifies the order of method calls|

