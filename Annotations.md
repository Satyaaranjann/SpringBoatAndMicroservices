# Annotations — Complete Guide & Spring Boot Usage

---

# Part 1: Core Java Annotations

## 1. What is an Annotation?

An **annotation** is metadata added to code that provides additional information without directly affecting program logic. Read by the compiler, IDEs, frameworks (via reflection), and documentation tools.

## 2. Built-in Java Annotations

| Annotation | Purpose |
|---|---|
| `@Override` | Method overrides a superclass/interface method |
| `@Deprecated` | Marks code as outdated |
| `@SuppressWarnings` | Suppresses specific compiler warnings |
| `@FunctionalInterface` | Marks interface intended for lambdas |
| `@SafeVarargs` | Suppresses warnings for varargs + generics |
| `@Retention` | Meta-annotation — how long annotation is retained |
| `@Target` | Meta-annotation — where annotation can be applied |
| `@Inherited` | Meta-annotation — subclasses inherit the annotation |
| `@Documented` | Meta-annotation — included in generated docs |
| `@Repeatable` | Allows same annotation multiple times on one element |

**Example:**
```java
public class Animal {
    @Override
    public String toString() {
        return "Animal";
    }

    @Deprecated
    public void oldMethod() { }

    @SuppressWarnings("unchecked")
    public void legacyCast(Object obj) {
        List list = (List) obj;
    }
}
```

## 3. Custom Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface LogExecutionTime {
    String value() default "";
}
```

`RetentionPolicy` options:
- `SOURCE` — discarded by compiler (e.g., `@Override`)
- `CLASS` — kept in .class file, not available at runtime
- `RUNTIME` — available via reflection at runtime (needed for frameworks like Spring)

`ElementType` options: `TYPE`, `METHOD`, `FIELD`, `PARAMETER`, `CONSTRUCTOR`, `ANNOTATION_TYPE`, etc.

## 4. Annotations in Other Languages

**Python (type hints):**
```python
def greet(name: str, age: int = 25) -> str:
    return f"Hello {name}, age {age}"
```

**C# (attributes):**
```csharp
[Obsolete("Use NewMethod instead")]
public void OldMethod() { }
```

---

# Part 2: Spring Boot Annotations (Detailed)

## 1. Core Application Annotations

| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | Combines `@Configuration`, `@EnableAutoConfiguration`, `@ComponentScan` — entry point |
| `@Configuration` | Marks a class as a source of bean definitions |
| `@EnableAutoConfiguration` | Enables Spring Boot's auto-configuration mechanism |
| `@ComponentScan` | Tells Spring where to scan for components/beans |

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

## 2. Stereotype / Component Annotations

| Annotation | Purpose |
|---|---|
| `@Component` | Generic Spring-managed bean |
| `@Service` | Marks a service-layer class (business logic) |
| `@Repository` | Marks a DAO/persistence-layer class; enables exception translation |
| `@Controller` | Marks a web MVC controller (returns views) |
| `@RestController` | `@Controller` + `@ResponseBody` combined — returns JSON/XML directly |

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> { }

@RestController
@RequestMapping("/api/users")
public class UserController { }
```

## 3. Dependency Injection Annotations

| Annotation | Purpose |
|---|---|
| `@Autowired` | Injects a bean automatically (by type) |
| `@Qualifier` | Specifies which bean to inject when multiple candidates exist |
| `@Primary` | Marks a bean as the default choice among candidates |
| `@Value` | Injects a value from `application.properties`/`.yml` |
| `@Bean` | Declares a bean manually inside a `@Configuration` class |
| `@Lazy` | Delays bean initialization until first use |
| `@Scope` | Defines bean scope (`singleton`, `prototype`, `request`, `session`) |

```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class NotificationService {

    @Value("${notification.enabled}")
    private boolean enabled;

    @Autowired
    @Qualifier("emailSender")
    private Sender sender;
}
```

## 4. Web / REST Annotations

| Annotation | Purpose |
|---|---|
| `@RequestMapping` | Maps HTTP requests to handler methods/classes |
| `@GetMapping` | Shortcut for `@RequestMapping(method = GET)` |
| `@PostMapping` | Shortcut for POST |
| `@PutMapping` | Shortcut for PUT |
| `@DeleteMapping` | Shortcut for DELETE |
| `@PatchMapping` | Shortcut for PATCH |
| `@PathVariable` | Binds a URI template variable to a method parameter |
| `@RequestParam` | Binds a query parameter to a method parameter |
| `@RequestBody` | Binds the HTTP request body to an object (JSON → Java) |
| `@ResponseBody` | Serializes return value directly into the response body |
| `@ResponseStatus` | Sets the HTTP status code for a response |
| `@RequestHeader` | Binds a request header to a method parameter |
| `@CrossOrigin` | Enables CORS for a controller/method |

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @GetMapping
    public List<User> searchUsers(@RequestParam String name) {
        return userService.search(name);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

## 5. Data / JPA Annotations

| Annotation | Purpose |
|---|---|
| `@Entity` | Marks a class as a JPA entity (maps to a DB table) |
| `@Table` | Specifies table name/schema for an entity |
| `@Id` | Marks the primary key field |
| `@GeneratedValue` | Specifies primary key generation strategy |
| `@Column` | Customizes column mapping (name, nullable, length, unique) |
| `@Transient` | Field excluded from persistence |
| `@OneToOne` / `@OneToMany` / `@ManyToOne` / `@ManyToMany` | Entity relationship mappings |
| `@JoinColumn` | Specifies the foreign key column |
| `@JoinTable` | Specifies join table for many-to-many |
| `@Embeddable` / `@Embedded` | Embeds a value object into an entity |
| `@Version` | Enables optimistic locking |
| `@Lob` | Marks a large object field (BLOB/CLOB) |

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;

    @Version
    private Integer version;
}
```

## 6. Transaction Annotations

| Annotation | Purpose |
|---|---|
| `@Transactional` | Wraps method in a DB transaction (commit/rollback) |
| `@EnableTransactionManagement` | Enables annotation-driven transaction management |

```java
@Service
public class UserService {

    @Transactional
    public void transferBalance(Long fromId, Long toId, double amount) {
        // rollback automatically on unchecked exception
    }
}
```

## 7. Validation Annotations (Bean Validation / JSR-380)

| Annotation | Purpose |
|---|---|
| `@Valid` | Triggers validation on a method parameter |
| `@NotNull` | Field must not be null |
| `@NotEmpty` | Field must not be null/empty (strings, collections) |
| `@NotBlank` | String must not be null and must contain non-whitespace |
| `@Size` | Restricts string/collection length |
| `@Min` / `@Max` | Numeric bounds |
| `@Email` | Must be valid email format |
| `@Pattern` | Must match regex |
| `@Positive` / `@Negative` | Numeric sign constraints |

```java
public class UserDto {
    @NotBlank
    private String name;

    @Email
    private String email;

    @Min(18)
    private int age;
}

@PostMapping
public User create(@Valid @RequestBody UserDto dto) { ... }
```

## 8. Testing Annotations

| Annotation | Purpose |
|---|---|
| `@SpringBootTest` | Loads full application context for integration tests |
| `@Test` | Marks a JUnit test method |
| `@MockBean` | Adds a Mockito mock to the Spring context |
| `@WebMvcTest` | Loads only the web layer for controller testing |
| `@DataJpaTest` | Loads only JPA-related components for repository testing |
| `@BeforeEach` / `@AfterEach` | Setup/teardown before/after each test |
| `@ActiveProfiles` | Specifies active Spring profile for tests |

```java
@SpringBootTest
class UserServiceTest {

    @MockBean
    private UserRepository userRepository;

    @Test
    void testFindUser() {
        Mockito.when(userRepository.findById(1L)).thenReturn(Optional.of(new User()));
    }
}
```

## 9. Scheduling & Async Annotations

| Annotation | Purpose |
|---|---|
| `@EnableScheduling` | Enables scheduled task execution |
| `@Scheduled` | Marks a method to run on a schedule (cron/fixedRate/fixedDelay) |
| `@EnableAsync` | Enables asynchronous method execution |
| `@Async` | Runs method in a separate thread |

```java
@Component
public class ReportJob {

    @Scheduled(cron = "0 0 1 * * ?")
    public void generateDailyReport() { }

    @Async
    public void sendEmailAsync() { }
}
```

## 10. Exception Handling Annotations

| Annotation | Purpose |
|---|---|
| `@ExceptionHandler` | Handles specific exceptions in a controller |
| `@ControllerAdvice` | Global exception handling across all controllers |
| `@RestControllerAdvice` | `@ControllerAdvice` + `@ResponseBody` |

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

## 11. Security Annotations (Spring Security)

| Annotation | Purpose |
|---|---|
| `@EnableWebSecurity` | Enables Spring Security web configuration |
| `@PreAuthorize` | Method-level security check before execution |
| `@PostAuthorize` | Security check after method execution |
| `@Secured` | Restricts method access by role |
| `@RolesAllowed` | JSR-250 equivalent of `@Secured` |

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public void deleteUser(@PathVariable Long id) { }
```

## 12. Caching Annotations

| Annotation | Purpose |
|---|---|
| `@EnableCaching` | Enables Spring's caching abstraction |
| `@Cacheable` | Caches method result |
| `@CachePut` | Updates cache without skipping method execution |
| `@CacheEvict` | Removes entries from cache |

```java
@Cacheable("users")
public User getUser(Long id) { ... }

@CacheEvict(value = "users", key = "#id")
public void deleteUser(Long id) { ... }
```

## 13. Configuration Properties Annotations

| Annotation | Purpose |
|---|---|
| `@ConfigurationProperties` | Binds a group of properties to a POJO |
| `@EnableConfigurationProperties` | Enables `@ConfigurationProperties` beans |
| `@PropertySource` | Loads a specific properties file |
| `@Profile` | Activates a bean only for a specific environment/profile |

```java
@Configuration
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    private int port;
    // getters/setters
}

@Service
@Profile("dev")
public class DevEmailService implements EmailService { }
```

---

## 14. Quick Reference — Most Frequently Asked in Interviews

| Annotation | One-liner Answer |
|---|---|
| `@SpringBootApplication` | Bootstraps the app — combines config, auto-config, component scan |
| `@Autowired` | Dependency injection by type |
| `@Component` vs `@Service` vs `@Repository` | Functionally similar; semantic distinction + `@Repository` adds exception translation |
| `@RestController` | `@Controller` + `@ResponseBody`, returns data not views |
| `@Transactional` | Manages commit/rollback automatically |
| `@Qualifier` | Resolves ambiguity when multiple beans of same type exist |
| `@RequestBody` vs `@RequestParam` | Body = JSON payload; Param = query string |
| `@Entity` | Marks class as JPA-mapped table |
| `@Valid` | Triggers bean validation constraints |
| `@ControllerAdvice` | Centralized exception handling |