# Spring Boot — Complete Guide (Basic to Advanced) for Interviews

---

## Table of Contents
1. [Spring Framework Fundamentals](#1-spring-framework-fundamentals)
2. [What is Spring Boot & Why](#2-what-is-spring-boot--why)
3. [Spring Boot Starters & Auto-Configuration](#3-spring-boot-starters--auto-configuration)
4. [Application Structure & Annotations](#4-application-structure--annotations)
5. [Configuration (properties/yml, Profiles, @ConfigurationProperties)](#5-configuration)
6. [Dependency Injection & Bean Lifecycle Deep Dive](#6-dependency-injection--bean-lifecycle-deep-dive)
7. [Building REST APIs](#7-building-rest-apis)
8. [Exception Handling](#8-exception-handling)
9. [Validation](#9-validation)
10. [Spring Data JPA](#10-spring-data-jpa)
11. [Transactions](#11-transactions)
12. [Spring Security](#12-spring-security)
13. [Actuator & Monitoring](#13-actuator--monitoring)
14. [Caching](#14-caching)
15. [AOP (Aspect Oriented Programming)](#15-aop)
16. [Testing in Spring Boot](#16-testing-in-spring-boot)
17. [Microservices with Spring Boot](#17-microservices-with-spring-boot)
18. [Spring Boot Internals (Auto-Configuration Mechanism)](#18-spring-boot-internals)
19. [Performance, Deployment & Best Practices](#19-performance-deployment--best-practices)
20. [Design Patterns Used in Spring](#20-design-patterns-used-in-spring)
21. [Rapid-Fire Interview Q&A](#21-rapid-fire-interview-qa)

---

## 1. Spring Framework Fundamentals

### 1.1 What is Spring Framework?
Spring is a lightweight, open-source Java framework that provides comprehensive infrastructure support for developing Java applications. Core value: **Inversion of Control (IoC)** and **Dependency Injection (DI)**.

### 1.2 IoC (Inversion of Control)
Instead of objects creating their own dependencies, control is inverted — the **Spring IoC Container** creates and manages objects (beans) and injects dependencies.

**Interview Q:** *Why is IoC useful?*
A: Loose coupling, easier testing (mock injection), centralized object management, and better maintainability.

### 1.3 Dependency Injection Types
| Type | Example | Notes |
|---|---|---|
| Constructor Injection | `@Autowired` on constructor | **Recommended** — enables immutability, easy testing |
| Setter Injection | `@Autowired` on setter | Optional dependencies |
| Field Injection | `@Autowired` on field | Discouraged — hard to test, hides dependencies |

```java
@Service
public class OrderService {
    private final PaymentService paymentService;

    // Constructor injection (no @Autowired needed if single constructor, Spring 4.3+)
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### 1.4 ApplicationContext vs BeanFactory
- **BeanFactory**: Basic container, lazy loading, minimal features.
- **ApplicationContext**: Superset of BeanFactory — supports AOP, event propagation, internationalization, eager loading of singleton beans. **This is what Spring Boot uses.**

### 1.5 Spring Bean Scopes
| Scope | Description |
|---|---|
| `singleton` (default) | One instance per Spring container |
| `prototype` | New instance every time requested |
| `request` | One instance per HTTP request (web-aware) |
| `session` | One instance per HTTP session |
| `application` | One instance per ServletContext |

---

## 2. What is Spring Boot & Why

Spring Boot is built **on top of** Spring to eliminate boilerplate configuration. It provides:
- **Auto-configuration** — sensible defaults based on classpath contents.
- **Embedded servers** — Tomcat/Jetty/Undertow, no need for external WAR deployment.
- **Starter dependencies** — curated dependency bundles.
- **Production-ready features** — Actuator (health checks, metrics).
- **No XML configuration** — Java-based/annotation-based config.
- **Opinionated "convention over configuration"** defaults, all overridable.

**Interview Q:** *Spring vs Spring Boot?*
A: Spring requires manual configuration (XML/Java config) for DataSource, ViewResolver, DispatcherServlet etc. Spring Boot auto-configures these via starters and `@EnableAutoConfiguration`, and ships an embedded server so apps run as a standalone JAR (`java -jar app.jar`).

---

## 3. Spring Boot Starters & Auto-Configuration

### 3.1 Starters
Starters are dependency descriptors that pull in a curated set of libraries.
- `spring-boot-starter-web` → Spring MVC + embedded Tomcat + Jackson
- `spring-boot-starter-data-jpa` → Hibernate + Spring Data JPA + JDBC
- `spring-boot-starter-security` → Spring Security
- `spring-boot-starter-test` → JUnit, Mockito, AssertJ, Spring Test
- `spring-boot-starter-actuator` → production monitoring endpoints

### 3.2 Auto-Configuration
Triggered by `@EnableAutoConfiguration` (bundled inside `@SpringBootApplication`). Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3.x; older used `spring.factories`) and conditionally applies configuration classes using `@Conditional` annotations (e.g., `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`).

**Example:** If `spring-boot-starter-data-jpa` and an H2/MySQL driver are on the classpath, Spring Boot auto-configures a `DataSource`, `EntityManagerFactory`, and `TransactionManager` — unless you've defined your own beans (auto-config always backs off if a user-defined bean exists).

**Interview Q:** *How does Spring Boot know which beans to auto-configure?*
A: Via conditional annotations evaluated at startup against the classpath, existing beans, and property values — implemented as `@Configuration` classes annotated with `@Conditional...` guards.

---

## 4. Application Structure & Annotations

### 4.1 Entry Point
```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

`@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`

### 4.2 Stereotype Annotations
| Annotation | Purpose |
|---|---|
| `@Component` | Generic Spring-managed bean |
| `@Service` | Business/service layer (semantic marker) |
| `@Repository` | DAO layer; also enables exception translation (JDBC/Hibernate exceptions → `DataAccessException`) |
| `@Controller` | MVC controller returning views |
| `@RestController` | `@Controller` + `@ResponseBody` — returns data (JSON/XML) directly |
| `@Configuration` | Marks a class providing `@Bean` definitions |
| `@Bean` | Declares a bean inside a `@Configuration` class |

### 4.3 Component Scanning
`@ComponentScan` tells Spring where to look for annotated classes. By default, Spring Boot scans the package of the main class and its sub-packages — so **place `@SpringBootApplication` in the root package**.

---

## 5. Configuration

### 5.1 application.properties vs application.yml
```yaml
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
  jpa:
    hibernate:
      ddl-auto: update
```

### 5.2 Profiles
```yaml
spring:
  profiles:
    active: dev
```
Separate files: `application-dev.yml`, `application-prod.yml`. Activate via `-Dspring.profiles.active=prod` or env var `SPRING_PROFILES_ACTIVE`.

`@Profile("dev")` on a `@Bean`/`@Component` restricts it to that profile.

### 5.3 Externalized Config & @ConfigurationProperties
```java
@ConfigurationProperties(prefix = "app")
@Component
public class AppProps {
    private String name;
    private int timeout;
    // getters/setters
}
```
Property precedence (high → low, key ones): command-line args > `SPRING_APPLICATION_JSON` > OS env vars > `application-{profile}.yml` > `application.yml` > `@PropertySource` > defaults.

**Interview Q:** *`@Value` vs `@ConfigurationProperties`?*
A: `@Value` injects a single property with SpEL support; `@ConfigurationProperties` binds a whole prefixed group to a POJO, supports relaxed binding, validation (`@Validated`), and is better for structured config.

---

## 6. Dependency Injection & Bean Lifecycle Deep Dive

### 6.1 Bean Lifecycle
1. Instantiate → 2. Populate properties (DI) → 3. `Aware` interfaces (`BeanNameAware`, etc.) → 4. `BeanPostProcessor.postProcessBeforeInitialization` → 5. `@PostConstruct` / `InitializingBean.afterPropertiesSet()` → 6. Custom `init-method` → 7. `postProcessAfterInitialization` → 8. **Bean ready for use** → 9. `@PreDestroy` / `DisposableBean.destroy()` on container shutdown.

### 6.2 Circular Dependency
Spring throws `BeanCurrentlyInCreationException` for constructor-injection cycles. Fix: use setter/field injection for one side, refactor design, or use `@Lazy`.

### 6.3 `@Primary` vs `@Qualifier`
Used when multiple beans of the same type exist. `@Primary` sets a default; `@Qualifier("beanName")` picks explicitly at injection point.

---

## 7. Building REST APIs

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;
    public UserController(UserService userService) { this.userService = userService; }

    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserDto> create(@Valid @RequestBody UserDto dto) {
        UserDto saved = userService.save(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }

    @GetMapping
    public List<UserDto> list(@RequestParam(defaultValue = "0") int page) {
        return userService.findAll(page);
    }
}
```

### Key Annotations
- `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`
- `@PathVariable` — URI template variable
- `@RequestParam` — query parameter
- `@RequestBody` — deserialize JSON body to object
- `@ResponseEntity<T>` — full control over status code/headers/body

**Interview Q:** *Difference between `@RequestParam` and `@PathVariable`?*
A: `@RequestParam` extracts query-string params (`?id=5`); `@PathVariable` extracts values from the URI path (`/users/5`).

**Interview Q:** *PUT vs PATCH?*
A: PUT replaces the entire resource (idempotent, full payload); PATCH applies partial updates.

---

## 8. Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }
}
```
`@ControllerAdvice`/`@RestControllerAdvice` centralizes exception handling across all controllers (cross-cutting concern via AOP-like interception).

---

## 9. Validation

Uses **Bean Validation (JSR 380 / Jakarta Validation)** with Hibernate Validator.

```java
public class UserDto {
    @NotBlank(message = "Name is required")
    private String name;

    @Email
    private String email;

    @Min(18)
    private int age;
}
```
Trigger with `@Valid` or `@Validated` on controller method parameters. `@Validated` also supports validation groups and enables method-level validation on service classes.

---

## 10. Spring Data JPA

### 10.1 Entity & Repository
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders;
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByNameContainingIgnoreCase(String name);

    @Query("SELECT u FROM User u WHERE u.age > :age")
    List<User> findOlderThan(@Param("age") int age);
}
```

### 10.2 Key Concepts
- **JpaRepository** extends `PagingAndSortingRepository` extends `CrudRepository` — gives `save`, `findById`, `findAll`, `delete`, pagination/sorting out of the box.
- **Derived query methods** — Spring parses method names (`findBy...`, `countBy...`, `existsBy...`) into JPQL.
- **`@Query`** — custom JPQL or native SQL (`nativeQuery = true`).
- **Fetch types**: `LAZY` (default for collections, loads on access) vs `EAGER` (default for `@ManyToOne`/`@OneToOne`, loads immediately). Prefer LAZY to avoid N+1 issues; fetch explicitly with `JOIN FETCH` when needed.
- **Cascade types**: `ALL, PERSIST, MERGE, REMOVE, REFRESH, DETACH` — controls how operations propagate to related entities.
- **N+1 Problem**: occurs when a query fetches N parent rows then issues 1 additional query per row for lazy associations. Fix with `JOIN FETCH`, `@EntityGraph`, or batch fetching.

### 10.3 Pagination
```java
Page<User> page = userRepository.findAll(PageRequest.of(0, 10, Sort.by("name")));
```

**Interview Q:** *`CrudRepository` vs `JpaRepository`?*
A: `JpaRepository` adds JPA-specific methods (`flush()`, batch delete) and pagination/sorting support beyond the generic `CrudRepository`.

---

## 11. Transactions

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        // multiple DB operations — all commit or all rollback
    }
}
```

- Default rollback: only on **unchecked** exceptions (`RuntimeException`). Use `rollbackFor = Exception.class` to include checked exceptions.
- **Propagation** (common): `REQUIRED` (default — join existing or create new), `REQUIRES_NEW` (suspend current, start new), `NESTED`, `SUPPORTS`, `MANDATORY`.
- **Isolation levels**: `READ_UNCOMMITTED`, `READ_COMMITTED` (common default), `REPEATABLE_READ`, `SERIALIZABLE`.
- `@Transactional` works via **AOP proxies** — self-invocation (calling an `@Transactional` method from within the same class) bypasses the proxy and the annotation has **no effect**. This is a classic interview trap.

---

## 12. Spring Security

### 12.1 Basic Setup (Spring Security 6 / Boot 3 style — lambda DSL)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 12.2 Core Concepts
- **Authentication** — "who are you" (verifying identity).
- **Authorization** — "what can you do" (permissions/roles).
- **Filter Chain** — Security is implemented as a chain of Servlet Filters (`UsernamePasswordAuthenticationFilter`, `BasicAuthenticationFilter`, etc.).
- **JWT-based auth** — stateless, common in REST APIs: client sends `Authorization: Bearer <token>`, a custom filter validates the token and populates `SecurityContext`.
- **`@PreAuthorize("hasRole('ADMIN')")`** — method-level security (requires `@EnableMethodSecurity`).
- **CORS** vs **CSRF** — CORS controls cross-origin browser requests; CSRF protects against forged state-changing requests from authenticated sessions (usually disabled for stateless JWT APIs).

---

## 13. Actuator & Monitoring

Add `spring-boot-starter-actuator`. Exposes production endpoints:
- `/actuator/health` — app health (DB, disk space, custom `HealthIndicator`)
- `/actuator/metrics` — JVM/HTTP/custom metrics (integrates with Micrometer → Prometheus/Grafana)
- `/actuator/info` — build/app info
- `/actuator/env`, `/actuator/beans`, `/actuator/mappings` — introspection (should be secured/disabled in prod)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

---

## 14. Caching

```java
@EnableCaching
@Configuration
public class CacheConfig {}

@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) { ... }

    @CachePut(value = "products", key = "#product.id")
    public Product update(Product product) { ... }

    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) { ... }
}
```
Default cache is a simple in-memory `ConcurrentHashMap`; pluggable providers: **Redis, EhCache, Caffeine, Hazelcast**.

---

## 15. AOP

Spring AOP handles cross-cutting concerns (logging, security, transactions) declaratively.

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        System.out.println(pjp.getSignature() + " took " + (System.currentTimeMillis() - start) + "ms");
        return result;
    }
}
```

Key terms: **Aspect** (module of cross-cutting logic), **Advice** (`@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`), **Pointcut** (expression selecting join points), **Join Point** (a point during execution, e.g., method call), **Weaving** (linking aspects — Spring AOP does this at **runtime via proxies**, unlike AspectJ which weaves at compile/load time).

**Interview Q:** *How does Spring AOP work internally?*
A: Via **JDK dynamic proxies** (if target implements an interface) or **CGLIB proxies** (subclassing, if no interface) — both created by the container to wrap the target bean and intercept method calls.

---

## 16. Testing in Spring Boot

```java
@SpringBootTest
class OrderServiceTest {
    @Autowired
    private OrderService orderService;

    @MockBean
    private PaymentGateway paymentGateway; // replaces real bean in context
}

@WebMvcTest(UserController.class)   // loads only web layer
class UserControllerTest {
    @Autowired MockMvc mockMvc;

    @Test
    void shouldReturnUser() throws Exception {
        mockMvc.perform(get("/api/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("John"));
    }
}

@DataJpaTest  // loads only JPA layer, in-memory DB by default
class UserRepositoryTest { ... }
```

| Annotation | Scope |
|---|---|
| `@SpringBootTest` | Full application context — integration tests |
| `@WebMvcTest` | Only MVC layer (controllers, filters) |
| `@DataJpaTest` | Only JPA repositories (rolls back transactions automatically) |
| `@MockBean` | Adds a Mockito mock into the Spring context |
| `@Mock` / `@InjectMocks` (pure Mockito) | Unit tests without Spring context — faster |

---

## 17. Microservices with Spring Boot

- **Spring Cloud Config** — centralized external configuration server.
- **Eureka (Netflix)** — service discovery/registry.
- **Spring Cloud Gateway** — API gateway, routing, filters.
- **OpenFeign** — declarative REST client (`@FeignClient`).
- **Resilience4j** — circuit breaker, retry, rate limiter (replaced Netflix Hystrix).
- **Distributed tracing** — Micrometer Tracing + Zipkin/Sleuth.
- **Message brokers** — Kafka/RabbitMQ via `spring-kafka` / `spring-amqp` for async communication.

```java
@FeignClient(name = "inventory-service", url = "${inventory.url}")
public interface InventoryClient {
    @GetMapping("/stock/{sku}")
    StockDto getStock(@PathVariable String sku);
}
```

**Interview Q:** *How do microservices talk to each other?*
A: Synchronously via REST/Feign/gRPC, or asynchronously via message brokers (Kafka/RabbitMQ) for decoupled, event-driven communication.

---

## 18. Spring Boot Internals

### 18.1 `SpringApplication.run()` — what happens
1. Determine application type (Servlet/Reactive/None).
2. Load `ApplicationContextInitializer`s and `ApplicationListener`s.
3. Prepare `Environment` (load property sources, active profiles).
4. Print the banner.
5. Create the `ApplicationContext` (e.g., `AnnotationConfigServletWebServerApplicationContext`).
6. Run all `ApplicationContextInitializer`s.
7. Load bean definitions (component scan + auto-configuration classes).
8. Refresh the context → instantiate singleton beans, start embedded server.
9. Run `CommandLineRunner` / `ApplicationRunner` beans.
10. Publish `ApplicationReadyEvent`.

### 18.2 Auto-Configuration Mechanism
`@EnableAutoConfiguration` triggers `AutoConfigurationImportSelector`, which reads configuration class names from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Each class is guarded by conditions like:
- `@ConditionalOnClass` — class present on classpath
- `@ConditionalOnMissingBean` — no user-defined bean of that type exists yet
- `@ConditionalOnProperty` — a property has a specific value
- `@ConditionalOnWebApplication` — app is a web app

### 18.3 Embedded Server
`spring-boot-starter-web` pulls in Tomcat by default. Boot auto-configures a `TomcatServletWebServerFactory` (or Jetty/Undertow if that starter is used instead), which creates and starts the server as part of context refresh — no external servlet container/WAR needed.

---

## 19. Performance, Deployment & Best Practices

- **Build an executable JAR**: `mvn clean package` → `java -jar app.jar` (Spring Boot's Maven/Gradle plugin repackages it with an embedded server and nested dependency JARs).
- **Dockerize**: multi-stage builds, or use Spring Boot's built-in `spring-boot:build-image` (Buildpacks) for OCI images without a Dockerfile.
- **Layered JARs** — separate dependencies from app code into Docker layers for better caching.
- **Externalize secrets** — never hardcode; use env vars, Vault, or Config Server.
- **Connection pooling** — HikariCP is the default and fastest connection pool in Spring Boot 2+.
- **Graceful shutdown** — `server.shutdown=graceful` to drain in-flight requests.
- **Reduce startup time** — lazy initialization (`spring.main.lazy-initialization=true`), consider Spring Native/GraalVM for serverless.
- **Logging** — SLF4J + Logback by default; configure via `logback-spring.xml`.

---

## 20. Design Patterns Used in Spring

| Pattern | Where Used |
|---|---|
| Singleton | Default bean scope |
| Factory | `BeanFactory`, `ApplicationContext` |
| Proxy | AOP, `@Transactional`, `@Cacheable` |
| Template Method | `JdbcTemplate`, `RestTemplate` |
| Observer | `ApplicationEvent` / `ApplicationListener` |
| Builder | `UriComponentsBuilder`, `ResponseEntity.BodyBuilder` |
| Front Controller | `DispatcherServlet` |
| Strategy | `PasswordEncoder`, `HandlerAdapter` implementations |

---

## 21. Rapid-Fire Interview Q&A

**Q: What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?**
A: All are `@Component` specializations for classification/readability; `@Repository` additionally enables persistence exception translation; `@Controller`/`@RestController` are for the web layer.

**Q: What is `@RestControllerAdvice` used for?**
A: Global, centralized exception handling and response body advice across all `@RestController`s.

**Q: How do you handle versioning of REST APIs?**
A: URI versioning (`/api/v1/...`), request header versioning, or media-type (content negotiation) versioning.

**Q: What's the difference between `@Autowired` and `@Inject`?**
A: `@Autowired` is Spring-specific with `required` attribute; `@Inject` is standard JSR-330, no `required` attribute (use `Optional` or `@Nullable` instead), and prefers by-type then by-qualifier like `@Autowired`.

**Q: What happens if two beans of the same type exist and neither has `@Primary`/`@Qualifier`?**
A: `NoUniqueBeanDefinitionException` at startup.

**Q: Explain `application.yml` profile-specific override.**
A: `application-{profile}.yml` properties override the base `application.yml` for that active profile; still overridden by env vars/command-line args.

**Q: What is the difference between `@Async` and a normal method call?**
A: `@Async` (with `@EnableAsync`) executes the method in a separate thread from a `TaskExecutor`, returning immediately (optionally wrapped in `CompletableFuture`) — used for non-blocking, fire-and-forget or parallel work.

**Q: What is `CommandLineRunner`?**
A: A functional interface whose `run()` method executes once, right after the Spring context is fully loaded and just before the app is "ready" — useful for startup tasks (seeding data, etc.).

**Q: How does Spring Boot handle a 404 vs 500 error by default?**
A: `BasicErrorController` (`/error`) auto-configures a default JSON error response including status, error, message, timestamp — customizable via `ErrorAttributes` or `@ControllerAdvice`.

**Q: What is idempotency and which HTTP methods are idempotent?**
A: An idempotent operation produces the same result no matter how many times it's called. GET, PUT, DELETE are idempotent; POST is not; PATCH is generally not guaranteed to be.

**Q: What is `OncePerRequestFilter`?**
A: A Spring base filter class guaranteeing single execution per request (important in async/forward scenarios), commonly extended for custom JWT auth filters.

**Q: DispatcherServlet — what does it do?**
A: The Front Controller of Spring MVC — receives all incoming HTTP requests, delegates to the right `HandlerMapping`/`Controller`, then to a `ViewResolver` (or directly serializes via `HttpMessageConverter` for REST).

**Q: What's the difference between `spring-boot-starter-web` and `spring-boot-starter-webflux`?**
A: `-web` uses the traditional Servlet stack (Tomcat, blocking, thread-per-request); `-webflux` uses the Reactive stack (Netty, non-blocking, event-loop, backed by Project Reactor `Mono`/`Flux`).

**Q: How do you secure sensitive Actuator endpoints?**
A: Restrict exposure via `management.endpoints.web.exposure.include`, and secure with Spring Security rules on the `/actuator/**` path, or move them to a separate management port.

---

## Quick Revision Checklist Before an Interview
- [ ] Explain IoC/DI with an example
- [ ] Explain `@SpringBootApplication` composition
- [ ] Explain auto-configuration mechanism + `@Conditional` annotations
- [ ] Write a REST controller with proper status codes
- [ ] Explain `@Transactional` + self-invocation pitfall
- [ ] Explain N+1 problem and how to fix it
- [ ] Explain Spring Security filter chain basics + JWT flow
- [ ] Explain bean scopes and lifecycle
- [ ] Explain testing annotations (`@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`)
- [ ] Explain how microservices communicate (REST/Feign vs Kafka)
- [ ] Know 2–3 design patterns used internally by Spring