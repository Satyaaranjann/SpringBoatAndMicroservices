# Testing — Q&A

---

**Q1. What is Unit Testing with JUnit 5, and what are its key features?**

Unit testing verifies a **single unit of code** (typically one method or class) in isolation, without involving external dependencies like databases, network calls, or the Spring context. **JUnit 5** is the standard testing framework in modern Spring Boot, built from three modules:
- **JUnit Platform** — foundation for launching testing frameworks on the JVM
- **JUnit Jupiter** — the new programming/extension model (annotations like `@Test`)
- **JUnit Vintage** — backward compatibility for JUnit 3/4 tests

```java
class CalculatorTest {

    private final Calculator calculator = new Calculator();

    @Test
    void shouldAddTwoNumbers() {
        int result = calculator.add(2, 3);
        assertEquals(5, result);
    }

    @Test
    @DisplayName("Division by zero should throw")
    void shouldThrowOnDivideByZero() {
        assertThrows(ArithmeticException.class, () -> calculator.divide(10, 0));
    }

    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3, 4})
    void shouldBePositive(int number) {
        assertTrue(number > 0);
    }

    @BeforeEach
    void setUp() { /* runs before each test */ }

    @AfterEach
    void tearDown() { /* runs after each test */ }
}
```

**Key annotations:** `@Test`, `@BeforeEach`/`@AfterEach` (per-test setup/teardown), `@BeforeAll`/`@AfterAll` (once per class, must be `static`), `@DisplayName`, `@Disabled`, `@ParameterizedTest` (run the same test with multiple inputs), `@Nested` (grouping related tests).

**Assertions** typically come from `org.junit.jupiter.api.Assertions` (`assertEquals`, `assertTrue`, `assertThrows`, `assertAll`) or a fluent library like **AssertJ** (`assertThat(result).isEqualTo(5)`), which is bundled in `spring-boot-starter-test` and generally preferred for readability.

---

**Q2. What are Mockito Basics?**

**Mockito** is a mocking framework used to create fake ("mock") implementations of dependencies, so you can test a class **in isolation** without invoking the real behavior of its collaborators (database calls, external APIs, other services).

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserWhenFound() {
        User user = new User(1L, "John");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        User result = userService.getUser(1L);

        assertEquals("John", result.getName());
        verify(userRepository, times(1)).findById(1L);
    }

    @Test
    void shouldThrowWhenUserNotFound() {
        when(userRepository.findById(99L)).thenReturn(Optional.empty());

        assertThrows(ResourceNotFoundException.class, () -> userService.getUser(99L));
    }
}
```

**Core annotations/methods:**
- **`@Mock`** — creates a fake instance of a dependency
- **`@InjectMocks`** — creates the class under test and injects the `@Mock`s into it
- **`when(...).thenReturn(...)`** — stubs behavior for a mock method call
- **`verify(mock, times(n))`** — asserts a mock method was called a specific number of times
- **`ArgumentCaptor`** — captures the actual arguments passed to a mock, for detailed assertions
- **`when(...).thenThrow(...)`** — stubs a mock to throw an exception

**Why mock at all:** unit tests should be fast, deterministic, and isolated — hitting a real database or external API in every unit test makes tests slow, flaky, and dependent on external state.

---

**Q3. What is `@SpringBootTest`?**

Loads the **full Spring application context** — all beans, all auto-configuration — making it a true **integration test** rather than a unit test. Use it when you need to verify how multiple components work together end-to-end.

```java
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Test
    void shouldCreateAndRetrieveUser() {
        User created = userService.create(new UserDto("John", "john@example.com"));
        User found = userService.getUser(created.getId());
        assertEquals("John", found.getName());
    }
}
```

**Web environment options:**
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)  // starts a real embedded server on a random port
@SpringBootTest(webEnvironment = WebEnvironment.MOCK)         // mocks the servlet environment, no real server (default)
@SpringBootTest(webEnvironment = WebEnvironment.NONE)         // no web environment at all
```

**Trade-off:** `@SpringBootTest` is comprehensive but **slow** — it boots the entire context, which adds up significantly across a large test suite. It should be used selectively for true integration scenarios, not as the default for every test (a very common anti-pattern interviewers ask about).

---

**Q4. What is `@WebMvcTest`?**

A **slice test** that loads only the **web layer** — controllers, `@ControllerAdvice`, filters, and MVC infrastructure — while **excluding** service/repository beans and the full application context. Dependencies like services must be mocked (typically with `@MockBean`).

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

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

**Why use it over `@SpringBootTest`:** it's much **faster** (only the relevant slice of beans loads) and more focused — it tests exactly what a controller test should test: request mapping, serialization, validation, exception handling — without dragging in the database or business logic layers.

---

**Q5. What is `@DataJpaTest`?**

A **slice test** for the JPA/persistence layer — loads only JPA-related configuration (`@Entity` classes, Spring Data repositories) plus an embedded, **in-memory database** (H2 by default) instead of your real production database.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindByEmail() {
        User user = new User(null, "John", "john@example.com");
        entityManager.persistAndFlush(user);

        Optional<User> found = userRepository.findByEmail("john@example.com");

        assertTrue(found.isPresent());
        assertEquals("John", found.get().getName());
    }
}
```

**Key characteristics:**
- **Transactional by default** — each test method runs in a transaction that's **rolled back** at the end, so tests don't leave residual data affecting each other
- Uses **`TestEntityManager`** — a test-friendly wrapper around `EntityManager` for setting up test data directly
- Replaces your configured `DataSource` with an embedded one **unless** you explicitly override this (see Testcontainers, Q6) — a detail that matters a lot when your queries rely on database-specific SQL features not supported by H2

---

**Q6. What is MockMvc, and how is it used?**

`MockMvc` simulates HTTP requests **against your Spring MVC controllers without starting a real HTTP server** — it dispatches requests through the actual `DispatcherServlet` machinery in-memory, making controller tests fast while still exercising real request-handling logic (routing, validation, serialization, exception handling).

```java
mockMvc.perform(post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{\"name\":\"John\",\"email\":\"john@example.com\"}"))
    .andExpect(status().isCreated())
    .andExpect(header().exists("Location"))
    .andExpect(jsonPath("$.name").value("John"));

mockMvc.perform(get("/api/users/999"))
    .andExpect(status().isNotFound())
    .andExpect(jsonPath("$.message").value("User not found with id: 999"));
```

**Common assertions:** `status().isOk()`/`isCreated()`/`isNotFound()`, `jsonPath("$.field").value(...)` (verify specific JSON fields in the response), `content().contentType(...)`, `header().string(...)`.

`MockMvc` is typically auto-configured and injected when using `@WebMvcTest`, or can be added to a full `@SpringBootTest` via `@AutoConfigureMockMvc`.

---

**Q7. What are Testcontainers, and why use them?**

**Testcontainers** is a library that spins up real, disposable **Docker containers** during test execution — e.g., an actual PostgreSQL, MySQL, Kafka, or Redis instance — instead of relying on an in-memory substitute (like H2) or mocks. Containers start before tests run and are automatically destroyed afterward.

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldPersistAndRetrieveUser() {
        User saved = userRepository.save(new User(null, "John", "john@example.com"));
        assertTrue(userRepository.findById(saved.getId()).isPresent());
    }
}
```

**Why prefer this over H2 for integration tests:** H2 behaves differently from production databases in subtle ways (different SQL dialect quirks, different handling of certain constraints/types) — tests passing against H2 can still fail against real PostgreSQL/MySQL in production. Testcontainers eliminates this gap entirely by testing against the **exact same database engine and version** used in production, at the cost of requiring Docker to be available in the test/CI environment.

---

**Q8. What are the main Integration Testing Strategies in a Spring Boot application?**

Different layers/scopes of integration testing, from narrowest to broadest:

1. **Slice tests** (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`) — test one architectural layer in isolation with the rest mocked/excluded; fast, focused
2. **Full context tests** (`@SpringBootTest`) — boot the entire application context, testing how components integrate together; slower, more comprehensive
3. **Testcontainers-backed integration tests** — full context tests against real external dependencies (DB, message broker) via Docker; slowest, most realistic
4. **Contract testing** (e.g., Spring Cloud Contract) — verifies that a service's API matches the contract expected by its consumers, useful in microservices to catch breaking changes without spinning up the actual consumer service
5. **End-to-end (E2E) tests** — test the fully deployed system (or a close approximation) including all real integrations; broadest scope, typically fewer in number, run in CI/staging rather than locally

**The testing pyramid principle:** many fast unit tests at the base, a moderate number of slice/integration tests in the middle, and few, slower full end-to-end tests at the top — because a suite dominated by slow, broad tests becomes painful to run frequently and slows down development feedback loops.

---

**Q9. What are common Test Coverage Tools, and what do they measure?**

**JaCoCo (Java Code Coverage)** is the standard tool in the Spring Boot/Maven-Gradle ecosystem — it instruments bytecode during test execution to measure how much of the codebase is actually exercised by tests.

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

**Types of coverage it reports:**
- **Line coverage** — percentage of executable lines run by at least one test
- **Branch coverage** — percentage of decision branches (`if`/`else`, `switch` cases) exercised in both directions
- **Instruction coverage** — most granular, at the bytecode level

**Important nuance for interviews:** high coverage percentage does **not** guarantee good tests — a test can execute a line without meaningfully **asserting** its correctness (e.g., calling a method but never checking its output). Coverage tells you what code ran, not whether it was verified correctly — treating a coverage target (e.g., "80%") as the actual quality goal, rather than a rough signal, is a common anti-pattern worth calling out explicitly.

---

## Extra / Important Interview Questions

**Q10. What's the difference between `@Mock` and `@MockBean`?**

- **`@Mock`** (pure Mockito) — creates a mock **without any Spring context involvement**; used in plain unit tests with `MockitoExtension`, fastest option
- **`@MockBean`** (Spring Boot Test) — creates a mock **and registers it as a bean in the Spring application context**, replacing any real bean of that type; used within `@SpringBootTest`/`@WebMvcTest`/etc. when the class under test is wired via Spring dependency injection rather than manually constructed

Use `@Mock` for pure unit tests (fastest, no Spring context); use `@MockBean` only when you actually need the Spring context loaded (e.g., testing a controller via `@WebMvcTest`, where the service dependency must be mocked but still injected by Spring).

---

**Q11. Why can overusing `@SpringBootTest` across an entire test suite become a serious problem at scale?**

Each `@SpringBootTest` (with a distinct configuration) forces Spring to **boot a new application context**, which is expensive (seconds per boot, multiplied across potentially hundreds of test classes). Spring **does cache contexts** across test classes when the configuration is identical, but even minor differences (different `@MockBean`s, different `@ActiveProfiles`) create a **new** cached context, multiplying startup cost dramatically in a large test suite — turning a test run that should take seconds into one taking many minutes. This is why slice tests (`@WebMvcTest`, `@DataJpaTest`) and plain unit tests (no Spring context at all) should make up the vast majority of a healthy test suite, with full `@SpringBootTest` reserved for genuinely necessary integration scenarios.

---

**Q12. If a test passes locally against H2 but fails in production against PostgreSQL, what's the likely root cause, and how would you have caught it earlier?**

Likely cause: a SQL dialect or behavior difference between H2's compatibility mode and real PostgreSQL — e.g., a native query using PostgreSQL-specific syntax, a case-sensitivity difference, or a constraint/type that H2 handles more leniently than Postgres does. This is a textbook argument for using **Testcontainers** instead of H2 for any test that exercises real queries — testing against the actual database engine used in production would have caught the discrepancy locally, before it ever reached production.

---

**Q13. How do you test that a `@Transactional` service method actually rolls back correctly on failure?**

Write a test that deliberately triggers the failure condition and then asserts the **data state afterward**, not just that an exception was thrown:
```java
@SpringBootTest
class OrderServiceTransactionTest {

    @Autowired private OrderService orderService;
    @Autowired private OrderRepository orderRepository;

    @Test
    void shouldRollBackEntireOrderOnPaymentFailure() {
        Order order = buildOrderWithInvalidPayment();

        assertThrows(PaymentFailedException.class, () -> orderService.placeOrder(order));

        // verify NOTHING was persisted, confirming the rollback actually happened
        assertEquals(0, orderRepository.count());
    }
}
```
This is a stronger, more meaningful test than simply asserting the exception is thrown — it verifies the **actual guarantee** `@Transactional` is supposed to provide (atomicity), not just the surface-level exception behavior.

---

**Q14. What's a "flaky test," and what are common causes in a Spring Boot test suite?**

A flaky test passes and fails **inconsistently** without any code change — undermining trust in the test suite (developers start ignoring failures, assuming "it's just flaky"). Common causes:
- **Shared mutable state** between tests (e.g., a database not properly cleaned up/rolled back between tests)
- **Timing/async issues** — asserting on an `@Async` or `@Scheduled` side effect before it's actually completed
- **Test execution order dependency** — a test that only passes if another test ran first and left behind specific state
- **External dependencies** — hitting a real network resource that's occasionally slow/unavailable, instead of mocking it
- **Non-deterministic data** — relying on `LocalDateTime.now()`, random IDs, or unordered collections without proper handling

Fixing flakiness usually means enforcing test isolation (fresh state per test, via `@Transactional` rollback or Testcontainers), and avoiding unmocked external calls in the unit/integration test layers.

---

*Interview tip: Testing questions increasingly focus on test suite **health at scale** — slow suites from context-caching misses, flaky async tests, coverage-as-vanity-metric — rather than just "what annotation loads what." Being able to diagnose a slow or flaky suite is a stronger senior-level signal than reciting the slice-test list.*