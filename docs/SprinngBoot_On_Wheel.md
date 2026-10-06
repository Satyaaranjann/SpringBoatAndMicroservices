# Spring Boot: Complete Guide (Basic → Advanced)

> Covers theory, industry-oriented examples, interview Q&A and scenario-based Q&A for every topic.
> Targets Spring Boot 3.x (Java 17+, Jakarta EE namespace). Concepts carry over to newer versions.

---

## Table of Contents

**Part 1 – Foundations**
1. Spring vs Spring Boot, IoC and DI
2. Project Setup, Starters, Auto-Configuration
3. Beans, Scopes, Lifecycle
4. Configuration: Properties, YAML, Profiles, `@ConfigurationProperties`

**Part 2 – Web Layer**
5. REST Controllers and Layered Architecture
6. Validation and Global Exception Handling
7. REST Best Practices (DTO, Pagination, Versioning, Idempotency)

**Part 3 – Data Layer**
8. Spring Data JPA and Hibernate
9. Transactions
10. Database Migration (Flyway/Liquibase) and Connection Pooling

**Part 4 – Security**
11. Spring Security, JWT, OAuth2

**Part 5 – Cross-Cutting**
12. AOP
13. Caching
14. Async, Scheduling, Events
15. Logging, Actuator, Observability

**Part 6 – Testing**
16. Testing Spring Boot Applications

**Part 7 – Distributed Systems**
17. Messaging: Kafka / RabbitMQ
18. Microservices Patterns (Gateway, Discovery, Feign, Resilience4j, Saga, Outbox)

**Part 8 – Advanced and Production**
19. Reactive (WebFlux) and Virtual Threads
20. Docker, Kubernetes, GraalVM Native
21. Performance Tuning and Production Checklist
22. Spring Boot 3 Specifics
23. Rapid-Fire Interview Questions
24. Cheat Sheet and Learning Roadmap

---

# PART 1: FOUNDATIONS

## 1. Spring vs Spring Boot, IoC and DI

### Definition
- **Spring Framework**: a Java framework providing a lightweight container for managing objects (beans) and their dependencies, plus modules for web, data, security, AOP, etc.
- **Spring Boot**: an opinionated layer on top of Spring that removes boilerplate through **auto-configuration**, **starter dependencies**, **embedded servers** and **production-ready features** (Actuator).
- **IoC (Inversion of Control)**: the framework, not your code, creates and controls the lifecycle of objects.
- **DI (Dependency Injection)**: the mechanism of IoC; dependencies are supplied to a class from outside.

### Theory
| Aspect | Spring | Spring Boot |
|---|---|---|
| Configuration | Manual XML/Java config | Auto-configuration |
| Server | External WAR on Tomcat | Embedded Tomcat/Jetty/Undertow |
| Dependencies | Pick each version yourself | Starters + managed BOM |
| Production features | Add yourself | Actuator built in |
| Setup time | High | Minutes |

**Container types**: `BeanFactory` (lazy, basic) and `ApplicationContext` (eager, events, i18n, AOP integration). Spring Boot uses `ApplicationContext`.

**DI types**
1. **Constructor injection** (recommended): immutable, testable, fails fast, supports `final`.
2. **Setter injection**: for optional dependencies.
3. **Field injection** (`@Autowired` on field): avoid; hard to test, hides dependencies.

### Industry Example
```java
@Service
public class OrderService {
    private final PaymentGateway paymentGateway;   // interface
    private final OrderRepository orderRepository;

    // @Autowired optional with a single constructor
    public OrderService(PaymentGateway paymentGateway, OrderRepository orderRepository) {
        this.paymentGateway = paymentGateway;
        this.orderRepository = orderRepository;
    }
}
```
In production you swap `PaymentGateway` between `StripeGateway` and `RazorpayGateway` using profiles or `@ConditionalOnProperty` without touching `OrderService`.

### Interview Q&A
**Q1. What is the difference between Spring and Spring Boot?**
Spring is the core framework (IoC, AOP, MVC, Data). Spring Boot simplifies using it via auto-configuration, starters, embedded servers and Actuator. Boot does not replace Spring; it configures it.

**Q2. Why is constructor injection preferred?**
Immutability (`final` fields), mandatory dependencies guaranteed, easy unit tests without Spring (`new OrderService(mock1, mock2)`), and circular dependencies fail at startup instead of hiding.

**Q3. What if two beans of the same type exist?**
Spring throws `NoUniqueBeanDefinitionException`. Resolve with `@Primary`, `@Qualifier("name")`, or inject `List<Type>` / `Map<String, Type>`.

**Q4. BeanFactory vs ApplicationContext?**
`BeanFactory` is the basic lazy container. `ApplicationContext` extends it with eager singleton creation, event publishing, message sources, environment abstraction and AOP support.

**Q5. What is a circular dependency and how is it handled?**
A needs B and B needs A. With constructor injection it fails. Since Boot 2.6 circular references are prohibited by default. Fix by redesign (extract a third class, use events), or as a last resort `@Lazy`.

### Scenario-Based Q&A
**S1. Your app fails at startup with `NoUniqueBeanDefinitionException` for `NotificationService` (Email and SMS implementations). What do you do?**
Decide the intent. If one is default: mark with `@Primary`. If callers pick one: `@Qualifier("smsNotification")`. If you need all: inject `List<NotificationService>` and iterate (Strategy pattern), or `Map<String, NotificationService>` keyed by bean name.

**S2. Two services depend on each other and startup fails. Quick fix vs right fix?**
Quick: `@Lazy` on one constructor parameter. Right: extract the shared logic into a third service, or decouple through domain events (`ApplicationEventPublisher`).

---

## 2. Project Setup, Starters, Auto-Configuration

### Definition
- **Starter**: a curated dependency descriptor (e.g. `spring-boot-starter-web`) that pulls a compatible set of libraries.
- **Auto-configuration**: Boot inspects the classpath, properties and existing beans, then registers beans conditionally.
- **`@SpringBootApplication`** = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`.

### Theory
**Typical structure**
```
com.company.orderservice
├── OrderServiceApplication.java      (root, scans sub-packages)
├── controller/
├── service/
├── repository/
├── domain/ (entities)
├── dto/
├── config/
├── exception/
└── mapper/
```
**How auto-configuration works**
1. `@EnableAutoConfiguration` loads class names from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 2.7+/3.x; earlier `spring.factories`).
2. Each auto-config class has conditions: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, `@ConditionalOnWebApplication`.
3. Your own bean always wins due to `@ConditionalOnMissingBean`.

Debug with `--debug` or `/actuator/conditions` to see the conditions report (positive/negative matches).

**Common starters**: `web`, `data-jpa`, `security`, `validation`, `actuator`, `test`, `data-redis`, `amqp`, `cache`, `webflux`, `oauth2-resource-server`.

**Embedded server swap**
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

### Industry Example: custom starter for company-wide logging/audit
Large organisations create an internal starter (`company-audit-spring-boot-starter`) with an `@AutoConfiguration` class, `@ConditionalOnProperty("company.audit.enabled")` and an `AuditAspect`. Every microservice adds one dependency and gets uniform audit behaviour.

```java
@AutoConfiguration
@ConditionalOnProperty(prefix = "company.audit", name = "enabled", havingValue = "true")
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public AuditAspect auditAspect(AuditProperties props) {
        return new AuditAspect(props);
    }
}
```
Register it in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

### Interview Q&A
**Q1. What does `@SpringBootApplication` do?**
Combines `@Configuration` (bean definitions), `@EnableAutoConfiguration` (conditional bean setup) and `@ComponentScan` (scans the package and sub-packages of the annotated class).

**Q2. How do you exclude an auto-configuration?**
`@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` or `spring.autoconfigure.exclude=...` in properties.

**Q3. How does Boot decide to configure a `DataSource`?**
`DataSourceAutoConfiguration` is `@ConditionalOnClass(DataSource.class)` plus a driver on the classpath. If you define your own `DataSource` bean, auto-config backs off.

**Q4. JAR vs WAR?**
Boot defaults to an executable "fat" JAR with embedded server. WAR is only for deploying to an external container. Prefer JAR (+ Docker).

**Q5. How do you run code at startup?**
`CommandLineRunner`, `ApplicationRunner`, `@PostConstruct`, or listen to `ApplicationReadyEvent`. Use `ApplicationReadyEvent` when the app must be fully up.

### Scenario-Based Q&A
**S1. App fails: "Failed to configure a DataSource: no embedded datasource could be configured". Why?**
`spring-boot-starter-data-jpa` is present but no `spring.datasource.url` or driver. Add the properties and driver, or exclude `DataSourceAutoConfiguration` if no DB is needed.

**S2. Your component is not being picked up. Why?**
Likely it is outside the root package's scope. Move it beneath the `@SpringBootApplication` class package or add `@ComponentScan(basePackages=...)`/`scanBasePackages`.

**S3. You want the same logging and metrics setup across 40 microservices.**
Build a custom starter/library (auto-configuration + properties), publish to the internal Maven repo, version it, and let teams adopt it by dependency.

---

## 3. Beans, Scopes, Lifecycle

### Definition
A **bean** is an object instantiated, assembled and managed by the Spring container.

### Theory
**Stereotype annotations**
| Annotation | Layer | Notes |
|---|---|---|
| `@Component` | Generic | Base stereotype |
| `@Service` | Business | Semantic only |
| `@Repository` | Persistence | Adds exception translation to `DataAccessException` |
| `@Controller` / `@RestController` | Web | `@RestController` = `@Controller` + `@ResponseBody` |
| `@Configuration` + `@Bean` | Config | For third-party classes, explicit creation |

**Scopes**
- `singleton` (default): one instance per container
- `prototype`: new instance on each request for the bean
- `request`, `session`, `application`, `websocket`: web scopes

**Singleton beans must be stateless/thread-safe.** Never keep per-request data in fields.

**Lifecycle**
1. Instantiate → 2. Populate dependencies → 3. `BeanNameAware`/`BeanFactoryAware` → 4. `BeanPostProcessor.postProcessBeforeInitialization` → 5. `@PostConstruct` → `InitializingBean.afterPropertiesSet` → custom `initMethod` → 6. `postProcessAfterInitialization` (AOP proxies created here) → 7. Ready → 8. `@PreDestroy` → `DisposableBean.destroy` → custom `destroyMethod`.

**`@Configuration` full vs lite**: `@Configuration` classes are CGLIB-proxied so inter-`@Bean` method calls return the same singleton. `@Component` with `@Bean` (lite mode) does not guarantee this. `proxyBeanMethods=false` speeds startup.

### Industry Example
```java
@Configuration
public class HttpClientConfig {
    @Bean
    public RestClient paymentRestClient(RestClient.Builder builder,
                                        @Value("${payment.base-url}") String baseUrl) {
        return builder.baseUrl(baseUrl)
                      .defaultHeader("X-Source", "order-service")
                      .build();
    }
}
```

```java
@Component
public class CacheWarmer {
    private final ProductRepository repo;
    public CacheWarmer(ProductRepository repo) { this.repo = repo; }

    @PostConstruct
    void warm() { repo.findTop100ByOrderByPopularityDesc(); }

    @PreDestroy
    void shutdown() { /* flush metrics, close resources */ }
}
```

### Interview Q&A
**Q1. Is a singleton bean thread-safe?**
No. Singleton scope means one instance, not thread safety. Keep it stateless or use thread-safe structures.

**Q2. What happens if a singleton injects a prototype bean?**
The prototype is created once at injection time, so it effectively behaves like a singleton. Use `ObjectProvider<T>`, `@Lookup`, or a scoped proxy (`proxyMode = TARGET_CLASS`).

**Q3. `@Component` vs `@Bean`?**
`@Component` is class-level, auto-detected by scanning. `@Bean` is method-level inside `@Configuration`, used for classes you do not own or need custom construction.

**Q4. What is a `BeanPostProcessor`?**
A hook that runs for every bean before/after initialization. Spring AOP, `@Autowired` processing and `@Async` use it.

**Q5. `@Repository` special behaviour?**
Translates persistence exceptions (e.g. `SQLException`) into Spring's unchecked `DataAccessException` hierarchy.

**Q6. How to create beans conditionally?**
`@Profile`, `@ConditionalOnProperty`, `@ConditionalOnMissingBean`, `@ConditionalOnClass`.

**Q7. Lazy initialization?**
`@Lazy` or `spring.main.lazy-initialization=true`. Faster startup but errors surface at first use.

### Scenario-Based Q&A
**S1. A singleton `ReportService` stores the "current user" in a field and users see each other's data. Fix?**
That is shared mutable state in a singleton. Pass the user as a method parameter, or read it from `SecurityContextHolder`/request scope. Never store request data in singleton fields.

**S2. Startup is slow (60 s) in a large app. What do you do?**
Enable lazy init selectively, reduce component scan scope, set `proxyBeanMethods=false`, remove unused starters, analyze with `--debug` and startup actuator endpoint (`ApplicationStartup` with `BufferingApplicationStartup`), consider AOT/native or CDS/CRaC.

---

## 4. Configuration: Properties, YAML, Profiles, `@ConfigurationProperties`

### Definition
Externalized configuration lets the same artifact run in dev/test/prod by changing environment-specific values outside the code.

### Theory
**Property source precedence (high → low, simplified)**
1. Command-line args
2. `SPRING_APPLICATION_JSON`
3. OS environment variables
4. Profile-specific `application-{profile}.yml`
5. `application.yml` / `.properties`
6. `@PropertySource`
7. Defaults (`SpringApplication.setDefaultProperties`)

**Profiles**: `spring.profiles.active=prod`. Use `@Profile("prod")` on beans. Profile groups: `spring.profiles.group.prod=prod-db,prod-logging`.

**`@Value` vs `@ConfigurationProperties`**
| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Binding | Single value, SpEL | Structured, typed object |
| Relaxed binding | Limited | Yes (`my-prop`, `myProp`, `MY_PROP`) |
| Validation | No | `@Validated` + JSR-380 |
| Metadata/IDE | No | Yes |

**Secrets**: never commit to Git. Use environment variables, Kubernetes Secrets, Vault, AWS Secrets Manager, or Spring Cloud Config with encryption.

### Industry Example
```yaml
# application.yml
app:
  payment:
    base-url: https://api.pay.example.com
    timeout: 3s
    retries: 3
---
spring:
  config:
    activate:
      on-profile: prod
app:
  payment:
    base-url: https://api.pay.live.com
```

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
        @NotBlank String baseUrl,
        @NotNull Duration timeout,
        @Min(0) @Max(10) int retries) {}
```
```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class App {}
```

### Interview Q&A
**Q1. `.properties` vs `.yml`?**
YAML is hierarchical and readable, supports multi-document files. Properties is flat. Both are supported; YAML cannot be loaded with `@PropertySource`.

**Q2. How do you override a property in Docker/Kubernetes?**
Environment variable: `APP_PAYMENT_BASE_URL=...` (upper-case, underscores). Relaxed binding maps it.

**Q3. How to fail fast on missing config?**
`@ConfigurationProperties` + `@Validated` with constraints; the app will not start with invalid values.

**Q4. What is `spring.config.import`?**
Imports additional config (files, Config Server, Vault, `configtree:` for mounted secrets). Example: `spring.config.import=optional:configserver:http://config:8888`.

**Q5. How to refresh config at runtime?**
Spring Cloud Config + `@RefreshScope` and `/actuator/refresh`, or Spring Cloud Bus for broadcast.

### Scenario-Based Q&A
**S1. DB password accidentally committed to Git. What now?**
Rotate the credential immediately (assume compromised), purge from history only as extra hygiene, move to secret manager/env var, add secret scanning to CI.

**S2. Same JAR must run in dev, QA, prod with different DBs.**
Build once, deploy many. Pass `SPRING_PROFILES_ACTIVE` and env vars; keep only non-secret defaults in the JAR.

---

# PART 2: WEB LAYER

## 5. REST Controllers and Layered Architecture

### Definition
A **REST controller** maps HTTP requests to Java methods and serializes responses (JSON by default via Jackson).

### Theory
**Layers**: Controller (HTTP) → Service (business, transactions) → Repository (data) → Database. Keep controllers thin, services cohesive, entities out of the API.

**Key annotations**: `@RestController`, `@RequestMapping`, `@GetMapping/@PostMapping/@PutMapping/@PatchMapping/@DeleteMapping`, `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader`, `@ResponseStatus`.

**Request flow**: `DispatcherServlet` → `HandlerMapping` → interceptors → `HandlerAdapter` → controller → `HttpMessageConverter` → response. Filters run before the DispatcherServlet; interceptors run inside it.

**HTTP semantics**
| Method | Safe | Idempotent | Use |
|---|---|---|---|
| GET | Yes | Yes | Read |
| POST | No | No | Create/action |
| PUT | No | Yes | Full replace |
| PATCH | No | Not guaranteed | Partial update |
| DELETE | No | Yes | Remove |

**Status codes**: 200, 201 (+`Location`), 204, 400, 401, 403, 404, 409, 422, 429, 500, 503.

### Industry Example
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService service;
    public OrderController(OrderService service) { this.service = service; }

    @PostMapping
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
        OrderResponse created = service.create(req);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    @GetMapping("/{id}")
    public OrderResponse get(@PathVariable Long id) { return service.get(id); }

    @GetMapping
    public Page<OrderResponse> list(@RequestParam(defaultValue = "PENDING") OrderStatus status,
                                    Pageable pageable) {
        return service.list(status, pageable);
    }
}
```

### Interview Q&A
**Q1. `@Controller` vs `@RestController`?**
`@RestController` adds `@ResponseBody` to all methods, so return values are written to the body (JSON), not resolved as views.

**Q2. `@RequestParam` vs `@PathVariable`?**
`@PathVariable` extracts from the URI path (`/orders/{id}`); `@RequestParam` from query string/form (`?status=NEW`).

**Q3. Filter vs Interceptor vs AOP?**
Filter: Servlet level, before Spring MVC, good for security/CORS/logging. Interceptor: Spring MVC level, access to handler, pre/post/afterCompletion. AOP: method-level, any bean, business cross-cutting.

**Q4. PUT vs PATCH?**
PUT replaces the entire resource; PATCH changes only given fields.

**Q5. How does Jackson serialization get customized?**
`@JsonProperty`, `@JsonIgnore`, `@JsonFormat`, `@JsonInclude(NON_NULL)`, a `Jackson2ObjectMapperBuilderCustomizer`, or `spring.jackson.*` properties.

**Q6. How do you return a file/stream?**
`ResponseEntity<Resource>` or `StreamingResponseBody` for large data; set `Content-Disposition`.

### Scenario-Based Q&A
**S1. Double-click on "Pay" created two orders.**
Make the endpoint idempotent: client sends an `Idempotency-Key` header; server stores key + response (Redis/DB unique constraint) and returns the stored result on repeats.

**S2. A 50 MB upload times out.**
Raise `spring.servlet.multipart.max-file-size`, stream to object storage (S3) using pre-signed URLs so the file bypasses the app server, process asynchronously.

---

## 6. Validation and Global Exception Handling

### Definition
- **Bean Validation (Jakarta Validation)**: annotation-based constraints on inputs.
- **Global exception handling**: central translation of exceptions into consistent HTTP error responses via `@RestControllerAdvice`.

### Theory
Constraints: `@NotNull`, `@NotBlank`, `@Size`, `@Email`, `@Pattern`, `@Min/@Max`, `@Positive`, `@Past`, `@Valid` (cascade). Use `@Validated` on class for method-parameter validation and validation groups.

Exceptions: `MethodArgumentNotValidException` (body), `ConstraintViolationException` (params), `HttpMessageNotReadableException` (bad JSON), `HttpRequestMethodNotSupportedException`.

**Spring Boot 3 / Spring 6**: `ProblemDetail` (RFC 7807/9457) is supported natively. Enable `spring.mvc.problemdetails.enabled=true` or extend `ResponseEntityExceptionHandler`.

### Industry Example
```java
public record CreateOrderRequest(
    @NotNull Long customerId,
    @NotEmpty List<@Valid OrderItemRequest> items,
    @Size(max = 200) String note) {}

public record OrderItemRequest(@NotNull Long productId, @Positive int quantity) {}
```
```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String m) { super(m); }
}

@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail notFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource not found");
        return pd;
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    ProblemDetail conflict(OptimisticLockingFailureException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT,
                "Resource was modified by another user. Reload and retry.");
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail fallback(Exception ex) {
        log.error("Unhandled", ex);            // log full stack
        return ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR,
                "Unexpected error");           // never leak internals
    }
}
```

### Interview Q&A
**Q1. `@Valid` vs `@Validated`?**
`@Valid` is the standard (Jakarta) and supports nested cascade. `@Validated` is Spring's, adds validation groups and enables method-level validation on class-annotated beans.

**Q2. How do you create a custom validator?**
Define an annotation with `@Constraint(validatedBy = X.class)` and implement `ConstraintValidator<A, T>`.

**Q3. `@ControllerAdvice` vs `@RestControllerAdvice`?**
The latter includes `@ResponseBody`.

**Q4. Should you expose stack traces?**
Never. Log them server-side with a correlation id, return a generic message plus the correlation id.

**Q5. Checked vs unchecked exceptions in Spring?**
Spring transactions roll back on unchecked by default. Checked exceptions do not roll back unless `rollbackFor` is set.

### Scenario-Based Q&A
**S1. Frontend wants field-level errors for a registration form.**
Handle `MethodArgumentNotValidException`, collect `FieldError`s into `Map<field, message>` inside `ProblemDetail` properties and return 400 (or 422).

**S2. Support gets "something went wrong" tickets and cannot trace them.**
Add a correlation/trace ID via MDC (filter or Micrometer Tracing), include it in logs and the error response body, so support can search logs by ID.

---

## 7. REST Best Practices (DTO, Pagination, Versioning, Idempotency)

### Definition
Design guidelines that keep APIs stable, secure and efficient.

### Theory
- **DTOs**: never expose entities (lazy-loading issues, over-posting/mass assignment, coupling). Map with MapStruct.
- **Pagination**: `Pageable` (offset) for small/medium data; **keyset/cursor pagination** for large tables (offset gets slow, `OFFSET 1,000,000`).
- **Versioning**: URI (`/v1`), header, or media type. Prefer additive, backward compatible changes.
- **HATEOAS**: optional hypermedia links.
- **OpenAPI**: `springdoc-openapi-starter-webmvc-ui`.
- **Rate limiting**: gateway level (Bucket4j, Redis, API Gateway).
- **CORS**: configure explicitly, never `*` with credentials.
- **Compression and caching**: `server.compression.enabled`, ETag with `ShallowEtagHeaderFilter`.

### Industry Example
```java
@Mapper(componentModel = "spring")
public interface OrderMapper {
    OrderResponse toResponse(Order order);
    Order toEntity(CreateOrderRequest req);
}
```
Keyset query:
```java
@Query("select o from Order o where o.id < :lastId order by o.id desc")
List<Order> next(@Param("lastId") Long lastId, Pageable limit);
```

### Interview Q&A
**Q1. Why not return entities directly?**
Leaks internal schema, triggers `LazyInitializationException`, causes infinite recursion in bidirectional relations, exposes sensitive fields, and couples DB schema to API contract.

**Q2. Offset vs keyset pagination?**
Offset is simple but O(n) skipping and unstable under inserts. Keyset uses an indexed cursor, consistent speed.

**Q3. How do you protect against mass assignment?**
Use request DTOs with only allowed fields, not entities.

**Q4. How do you version an API without breaking clients?**
Add fields (non-breaking), deprecate with headers, introduce `/v2` for breaking changes, support both for a sunset period.

### Scenario-Based Q&A
**S1. `/orders` endpoint with 10M rows is slow on page 50,000.**
Switch to keyset pagination on indexed column, add covering index, limit max page size, consider search engine (Elasticsearch) for arbitrary filtering.

**S2. Mobile clients still use v1 of the API, and you must change a response field.**
Keep v1 unchanged, introduce v2, track v1 usage via metrics, announce deprecation (Sunset/Deprecation headers), retire after traffic drops.

---

# PART 3: DATA LAYER

## 8. Spring Data JPA and Hibernate

### Definition
- **JPA**: Jakarta Persistence specification for ORM.
- **Hibernate**: the most common JPA implementation.
- **Spring Data JPA**: reduces repository boilerplate; generates implementations from interfaces.

### Theory
**Repository hierarchy**: `Repository` → `CrudRepository` → `PagingAndSortingRepository` → `JpaRepository`.

**Query options**: derived queries (`findByEmailAndStatus`), `@Query` (JPQL/native), `Specification` (dynamic), Querydsl, projections (interface/DTO), `@EntityGraph`.

**Entity states**: Transient → Managed (persistent) → Detached → Removed. Persistence context = first-level cache; dirty checking flushes changes at commit.

**Relationships**: `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`. Prefer `LAZY` fetching; `@ManyToOne` defaults to EAGER, so set `fetch = LAZY` explicitly.

**N+1 problem**: 1 query for parents + N queries for children. Fix with `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, or DTO projections.

**Locking**
- Optimistic: `@Version` (conflict detected at commit → `OptimisticLockException`).
- Pessimistic: `@Lock(PESSIMISTIC_WRITE)` (`SELECT ... FOR UPDATE`).

**Open Session in View (OSIV)**: `spring.jpa.open-in-view` defaults to true; keeps the session open during view rendering. Disable it in APIs (`false`) to avoid hidden queries and connection holding.

**ddl-auto**: `none`/`validate` in production; never `update`/`create` there. Use migrations.

**equals/hashCode on entities**: base on business key or id carefully; avoid Lombok `@Data` on entities (breaks lazy loading and `toString` recursion).

### Industry Example
```java
@Entity
@Table(name = "orders", indexes = @Index(name = "idx_orders_customer", columnList = "customer_id"))
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version
    private Long version;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @CreatedDate private Instant createdAt;

    public void addItem(OrderItem item) { items.add(item); item.setOrder(this); }
}

public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"items", "customer"})
    Optional<Order> findWithItemsById(Long id);

    @Query("select new com.company.dto.OrderSummary(o.id, o.status, sum(i.price * i.qty)) " +
           "from Order o join o.items i where o.customer.id = :cid group by o.id, o.status")
    List<OrderSummary> summaries(@Param("cid") Long customerId);
}
```

### Interview Q&A
**Q1. `save()` vs `saveAndFlush()`?**
`save` schedules persistence (flush at commit or before queries); `saveAndFlush` forces an immediate flush to DB.

**Q2. `findById` vs `getReferenceById`?**
`findById` hits the DB and returns `Optional`. `getReferenceById` returns a lazy proxy without querying, useful for setting FK references.

**Q3. LAZY vs EAGER?**
LAZY loads on access (needs open session/transaction). EAGER loads immediately and often causes unwanted joins. Prefer LAZY and fetch what you need explicitly.

**Q4. What causes `LazyInitializationException`?**
Accessing a lazy association after the session closed. Fix with fetch join/EntityGraph/DTO inside the transaction, not by switching to EAGER.

**Q5. `CascadeType` vs `orphanRemoval`?**
Cascade propagates operations to children. `orphanRemoval` deletes children removed from the parent's collection.

**Q6. What is the first-level vs second-level cache?**
L1: per `EntityManager`/transaction, always on. L2: shared across sessions (Ehcache/Redis via Hibernate), opt-in per entity.

**Q7. How to do batch inserts efficiently?**
`spring.jpa.properties.hibernate.jdbc.batch_size=50`, `order_inserts=true`; avoid `IDENTITY` (disables batching), use `SEQUENCE`; flush/clear periodically; or `JdbcTemplate.batchUpdate`.

**Q8. JPA vs JdbcTemplate vs jOOQ/MyBatis?**
JPA for CRUD domain models; JdbcTemplate/jOOQ for heavy reporting, complex SQL, bulk operations. Many systems mix them (CQRS-style).

### Scenario-Based Q&A
**S1. Listing 100 orders triggers 101 queries.**
Classic N+1. Use `@EntityGraph`/`JOIN FETCH` for the needed association, or a DTO projection. For collections with pagination, use `@BatchSize` or two-step queries (ids first, then fetch), since fetch-join with pagination applies limit in memory.

**S2. Two users edit the same product price simultaneously; last write wins silently.**
Add `@Version` for optimistic locking, catch `OptimisticLockingFailureException`, return 409, ask the user to refresh/retry. For hot counters (inventory) use atomic DB updates (`update stock set qty = qty - :n where id=:id and qty >= :n`) or pessimistic lock.

**S3. Report query on 50M rows times out.**
Check execution plan, add proper index, project only needed columns, use read replica, pre-aggregate (materialized view / summary table), paginate or stream with `@QueryHints(fetchSize)`, and never load entities for reports.

**S4. `ddl-auto=update` in production dropped nothing but added a wrong column. How to prevent?**
Use `validate`/`none` and manage schema via Flyway/Liquibase, reviewed in PRs.

---

## 9. Transactions

### Definition
A **transaction** groups operations into an atomic unit (ACID). Spring provides declarative transactions via `@Transactional` using AOP proxies.

### Theory
**Propagation**
| Type | Behaviour |
|---|---|
| `REQUIRED` (default) | Join existing or create new |
| `REQUIRES_NEW` | Suspend current, create new (independent commit/rollback) |
| `SUPPORTS` | Join if exists, else non-transactional |
| `MANDATORY` | Must exist, else exception |
| `NOT_SUPPORTED` | Run non-transactionally, suspend current |
| `NEVER` | Fail if one exists |
| `NESTED` | Savepoint inside the current transaction |

**Isolation**: `READ_UNCOMMITTED`, `READ_COMMITTED` (common default), `REPEATABLE_READ`, `SERIALIZABLE`. Problems: dirty read, non-repeatable read, phantom read.

**Pitfalls**
1. **Self-invocation**: calling a `@Transactional` method from another method in the same class bypasses the proxy, so no transaction.
2. **Private/final methods** are not proxied (CGLIB limitation).
3. **Checked exceptions** do not roll back by default; use `rollbackFor`.
4. **Swallowed exceptions** inside the method prevent rollback.
5. **`@Transactional` on non-Spring-managed objects** does nothing.
6. Long transactions hold DB connections; do not call remote HTTP inside them.
7. Use `readOnly = true` for queries (Hibernate skips dirty-checking, may route to replica).
8. **`@TransactionalEventListener(phase = AFTER_COMMIT)`** to publish events/messages only after commit.

### Industry Example
```java
@Service
public class CheckoutService {

    @Transactional
    public Order checkout(CheckoutRequest req) {
        Order order = orderRepository.save(Order.from(req));
        inventoryService.reserve(order);           // REQUIRED: same transaction
        auditService.record(order);                // REQUIRES_NEW: audit survives rollback
        eventPublisher.publishEvent(new OrderPlaced(order.getId()));
        return order;
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(Order order) { /* always committed */ }
}

@Component
class OrderEventHandler {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void on(OrderPlaced e) { kafkaTemplate.send("orders.placed", e.orderId().toString(), e); }
}
```

### Interview Q&A
**Q1. How does `@Transactional` work internally?**
Spring creates a proxy (JDK or CGLIB) around the bean. A `TransactionInterceptor` begins the transaction before the method, commits on success, rolls back on `RuntimeException`/`Error`.

**Q2. Why is my `@Transactional` not working?**
Self-invocation, non-public/final method, class not a Spring bean, exception caught and swallowed, checked exception without `rollbackFor`, or wrong `TransactionManager`.

**Q3. `REQUIRED` vs `REQUIRES_NEW`?**
`REQUIRED` joins the caller's transaction; failure rolls everything back. `REQUIRES_NEW` suspends the caller and uses its own physical transaction.

**Q4. What does `readOnly=true` do?**
Hints to the provider/driver: Hibernate sets flush mode to MANUAL (no dirty checking), some drivers/routers use replicas. It does not forbid writes at DB level in every case.

**Q5. Transactions across microservices?**
No distributed ACID (avoid 2PC). Use Saga (choreography/orchestration) with compensating actions and Outbox pattern.

**Q6. `@Transactional` on controller or service?**
Service layer, where business units of work live.

### Scenario-Based Q&A
**S1. Order saved but the "order placed" Kafka message was sent and then DB rolled back. Customers got emails for non-existent orders.**
Publishing inside the transaction is unsafe. Use `@TransactionalEventListener(AFTER_COMMIT)` or better the **Transactional Outbox**: write the event to an `outbox` table in the same transaction, a poller/CDC (Debezium) publishes it, consumers are idempotent.

**S2. Payment succeeded, inventory failed, you need to undo payment.**
Within one service: rollback. Across services: Saga with compensation (refund). Orchestrator tracks state; each step has a compensating action; steps are idempotent and retriable.

**S3. `processBatch()` calls `this.processOne()` (annotated `REQUIRES_NEW`) and failures roll back all.**
Self-invocation bypasses the proxy. Move `processOne` to another bean (or inject self proxy / use `TransactionTemplate`).

---

## 10. Database Migration and Connection Pooling

### Definition
- **Flyway / Liquibase**: version-controlled database schema migrations.
- **HikariCP**: default high-performance JDBC connection pool in Spring Boot.

### Theory
**Flyway**: files `V1__init.sql`, `V2__add_index.sql` in `db/migration`; checksums prevent tampering; `R__` repeatable migrations. Never edit an applied migration; add a new one. Use **expand/contract** for zero-downtime changes (add nullable column → deploy → backfill → enforce → drop old).

**Hikari tuning**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000
      max-lifetime: 1800000
      leak-detection-threshold: 20000
```
Pool size guideline: `connections = (core_count * 2) + effective_spindle_count`; more is not faster. Total pool across all instances must not exceed DB `max_connections`.

### Industry Example
`V5__add_orders_status_index.sql`:
```sql
CREATE INDEX CONCURRENTLY idx_orders_status_created ON orders(status, created_at);
```
(Flyway needs `executeInTransaction=false` for `CONCURRENTLY` in PostgreSQL.)

### Interview Q&A
**Q1. Why use Flyway over `ddl-auto`?**
Reproducible, reviewable, auditable schema evolution across environments; `ddl-auto` is unpredictable and unsafe in production.

**Q2. What happens when a connection pool is exhausted?**
Threads wait up to `connection-timeout` then throw `SQLTransientConnectionException`. Causes: long transactions, leaks, too-small pool, slow queries.

**Q3. Why is HikariCP fast?**
Lean bytecode, `ConcurrentBag`, minimal locking, efficient proxy generation.

### Scenario-Based Q&A
**S1. Under load: "Connection is not available, request timed out after 30000ms".**
Check slow queries, long transactions (remote calls inside `@Transactional`), `open-in-view`, connection leaks (`leak-detection-threshold`), then tune pool size relative to DB capacity. Adding more app instances may worsen it if DB max connections is the limit.

**S2. You must rename a column with zero downtime.**
Expand/contract: add new column, dual-write, backfill, switch reads, remove old column in a later release.

---

# PART 4: SECURITY

## 11. Spring Security, JWT, OAuth2

### Definition
**Spring Security** provides authentication (who are you), authorization (what can you do) and protection against common attacks (CSRF, session fixation, clickjacking).

### Theory
**Core components**
- `SecurityFilterChain`: ordered filters applied to requests (Spring Security 6 uses a `@Bean` of this type; `WebSecurityConfigurerAdapter` was removed).
- `AuthenticationManager` → `AuthenticationProvider` → `UserDetailsService` + `PasswordEncoder`.
- `SecurityContextHolder`: holds the current `Authentication`.
- Authorization: URL rules (`authorizeHttpRequests`) and method security (`@EnableMethodSecurity`, `@PreAuthorize("hasRole('ADMIN')")`).

**Passwords**: `BCryptPasswordEncoder` or `DelegatingPasswordEncoder` (Argon2, scrypt). Never store plaintext or plain SHA.

**Session vs Token**
| | Session (stateful) | JWT (stateless) |
|---|---|---|
| Scaling | Needs sticky/shared store | Easy horizontally |
| Revocation | Easy | Hard (use short expiry + refresh + denylist) |
| Fit | Server-rendered apps | APIs, microservices |

**JWT**: `header.payload.signature`. Validate signature, `exp`, `iss`, `aud`. Do not put sensitive data in the payload (it is only encoded, not encrypted). Prefer asymmetric signing (RS256/ES256) in multi-service setups.

**OAuth2 / OIDC roles**: Resource Owner, Client, Authorization Server (Keycloak, Okta, Auth0, Azure AD), Resource Server. Flows: Authorization Code + PKCE (web/mobile), Client Credentials (service-to-service). Avoid Implicit and Password flows.

**CSRF**: needed for cookie-based sessions; typically disabled for stateless token APIs (Authorization header). **CORS**: browser policy configured in the security chain.

### Industry Example: Resource server validating JWT from Keycloak
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://sso.company.com/realms/prod
```
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http
          .csrf(csrf -> csrf.disable())
          .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
          .authorizeHttpRequests(a -> a
              .requestMatchers("/actuator/health/**", "/v3/api-docs/**").permitAll()
              .requestMatchers(HttpMethod.GET, "/api/v1/products/**").hasAuthority("SCOPE_catalog.read")
              .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
              .anyRequest().authenticated())
          .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()));
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(12); }
}
```
Method-level ownership check:
```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.claims['sub']")
public Profile getProfile(String userId) { ... }
```

### Interview Q&A
**Q1. Authentication vs authorization?**
Authentication verifies identity; authorization decides permissions for that identity.

**Q2. Explain the Spring Security filter chain.**
A `DelegatingFilterProxy` → `FilterChainProxy` → a matching `SecurityFilterChain` with filters like `SecurityContextHolderFilter`, `CorsFilter`, `CsrfFilter`, `BearerTokenAuthenticationFilter`/`UsernamePasswordAuthenticationFilter`, `ExceptionTranslationFilter`, `AuthorizationFilter`.

**Q3. `hasRole('ADMIN')` vs `hasAuthority('ROLE_ADMIN')`?**
Equivalent; `hasRole` auto-prefixes `ROLE_`.

**Q4. How to secure microservices?**
Gateway validates/forwards tokens, each service is a resource server validating JWT (zero trust), use mTLS/service mesh internally, Client Credentials for service-to-service, least-privilege scopes.

**Q5. How do you handle JWT logout/revocation?**
Short-lived access tokens (5–15 min), refresh tokens with rotation, server-side denylist or token introspection for critical flows, key rotation via JWKS.

**Q6. Where to store JWT in a browser?**
`HttpOnly`, `Secure`, `SameSite` cookies mitigate XSS theft (then CSRF protection is needed); localStorage is XSS-vulnerable.

**Q7. Common vulnerabilities (OWASP) and prevention?**
SQL injection (parameterized queries), broken access control (method security, ownership checks), XSS (output encoding, CSP), SSRF (allow-lists), sensitive data exposure (TLS, no secrets in logs), insecure deserialization, vulnerable dependencies (SCA scans like Dependabot/OWASP Dependency-Check).

### Scenario-Based Q&A
**S1. A user can fetch another user's order by changing `/orders/123` ID (IDOR).**
Authorization on the resource, not just the endpoint: query by `id AND ownerId = currentUser`, or `@PostAuthorize("returnObject.ownerId == authentication.name")`. Add tests for cross-tenant access.

**S2. A leaked JWT signing key is discovered.**
Rotate keys immediately (JWKS with new `kid`), invalidate old tokens, force re-login, audit access logs, move key to a secret manager/HSM.

**S3. Multi-tenant SaaS: tenant data must be isolated.**
Put `tenant_id` in the token claims, enforce it in every query (Hibernate `@Filter`/`@TenantId`, row-level security in PostgreSQL), or schema/database per tenant for stronger isolation.

**S4. Brute-force attacks on login.**
Rate limiting per IP/account, progressive delays/lockout, CAPTCHA, MFA, monitor with alerts.

---

# PART 5: CROSS-CUTTING

## 12. AOP

### Definition
**Aspect-Oriented Programming** modularizes cross-cutting concerns (logging, security, transactions, metrics) separate from business logic.

### Theory
Terms: **Aspect**, **Join point** (method execution in Spring AOP), **Pointcut**, **Advice** (`@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`), **Weaving** (runtime proxies in Spring AOP).

Proxy-based: JDK dynamic proxy (interfaces) or CGLIB (classes; Boot default). Same limits as transactions: self-invocation, private/final methods.

### Industry Example: execution-time and audit aspect
```java
@Aspect
@Component
public class PerformanceAspect {
    private static final Logger log = LoggerFactory.getLogger(PerformanceAspect.class);

    @Around("@annotation(com.company.aop.Timed)")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            log.info("{} took {} ms", pjp.getSignature().toShortString(), ms);
        }
    }
}

@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.METHOD)
public @interface Timed {}
```

### Interview Q&A
**Q1. Spring AOP vs AspectJ?**
Spring AOP: proxy-based, method execution only, simpler. AspectJ: bytecode weaving, field/constructor join points, more powerful, more complex.

**Q2. Real uses of AOP in Spring?**
`@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, `@Retryable`.

**Q3. Why does my aspect not trigger?**
Bean not Spring-managed, self-invocation, final/private method, wrong pointcut, missing `spring-boot-starter-aop` (Boot 3 requires `spring-boot-starter-aop` for `@Aspect`).

### Scenario-Based Q&A
**S1. Audit every change on service methods marked `@Auditable` without touching business code.**
Custom annotation + `@Around`/`@AfterReturning` aspect that reads the principal from `SecurityContextHolder`, records action/args/result to an audit table (async, `REQUIRES_NEW`).

---

## 13. Caching

### Definition
Storing results of expensive operations to serve repeated requests faster.

### Theory
Enable with `@EnableCaching`. Annotations: `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`. Providers: Caffeine (local, in-process), Redis (distributed), Ehcache, Hazelcast.

Issues: stale data, **cache stampede** (many misses at once; use `sync=true`, locks, jittered TTL), **cache penetration** (nonexistent keys; cache nulls/Bloom filter), **cache avalanche** (mass expiry; randomize TTL), serialization versioning, key design.

Strategies: cache-aside (Spring default), write-through, write-behind.

### Industry Example
```java
@Service
public class ProductService {

    @Cacheable(cacheNames = "product", key = "#id", sync = true)
    public ProductDto get(Long id) { return mapper.toDto(repo.findById(id).orElseThrow()); }

    @CacheEvict(cacheNames = "product", key = "#dto.id")
    public void update(ProductDto dto) { ... }
}
```
```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 10m
```

### Interview Q&A
**Q1. `@Cacheable` vs `@CachePut`?**
`@Cacheable` skips the method if present. `@CachePut` always runs the method and updates the cache.

**Q2. Local vs distributed cache?**
Local (Caffeine): fastest but inconsistent between instances. Distributed (Redis): shared and consistent but network hop.

**Q3. Why isn't `@Cacheable` working?**
Self-invocation, non-public method, no `@EnableCaching`, key mismatch, or object not serializable (Redis).

### Scenario-Based Q&A
**S1. Product price changed but users still see old price for 10 minutes.**
Evict/update on write (`@CacheEvict`/event-driven invalidation across instances via Redis pub/sub or Kafka), reduce TTL for volatile data, or don't cache prices.

**S2. At midnight a flash sale cache expires and DB spikes.**
Stampede protection: `sync=true`, single-flight loading, TTL jitter, pre-warm, request coalescing.

---

## 14. Async, Scheduling, Events

### Definition
- **`@Async`**: run a method on a separate thread.
- **`@Scheduled`**: run tasks at fixed rate/delay/cron.
- **Application events**: in-process publish/subscribe for decoupling.

### Theory
Enable with `@EnableAsync` / `@EnableScheduling`. Always define a bounded `ThreadPoolTaskExecutor` (default `SimpleAsyncTaskExecutor` creates unbounded threads in older setups; Boot auto-configures an executor but tune it). Return `CompletableFuture<T>`. Exceptions in void async methods need `AsyncUncaughtExceptionHandler`. `@Async` has the same proxy limitations. Security/MDC context is not propagated by default; use `TaskDecorator`/`DelegatingSecurityContextExecutor`.

**Scheduling in a cluster**: each instance runs the job. Use ShedLock (DB/Redis lock), Quartz clustered, or a Kubernetes CronJob.

`fixedRate` (start-to-start) vs `fixedDelay` (end-to-start) vs `cron`.

### Industry Example
```java
@Configuration
@EnableAsync
class AsyncConfig {
    @Bean("notifyExecutor")
    Executor notifyExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(10); ex.setMaxPoolSize(50); ex.setQueueCapacity(500);
        ex.setThreadNamePrefix("notify-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.initialize();
        return ex;
    }
}

@Service
class NotificationService {
    @Async("notifyExecutor")
    public CompletableFuture<Void> sendEmail(String to, String body) { ... return completedFuture(null); }
}

@Component
class SettlementJob {
    @Scheduled(cron = "0 0 2 * * *", zone = "Asia/Kolkata")
    @SchedulerLock(name = "dailySettlement", lockAtMostFor = "PT30M")
    void run() { ... }
}
```

### Interview Q&A
**Q1. Why is `@Async` running synchronously?**
Self-invocation, missing `@EnableAsync`, or called within the same bean.

**Q2. `fixedRate` vs `fixedDelay`?**
`fixedRate` triggers every N ms from start times (can overlap if single-threaded it queues); `fixedDelay` waits N ms after the previous execution finishes.

**Q3. `ApplicationEvent` vs Kafka?**
Application events are in-JVM, synchronous by default, not durable. Kafka is distributed, durable, asynchronous across services.

### Scenario-Based Q&A
**S1. Your scheduled report email goes out 3 times (3 pods).**
Add a distributed lock (ShedLock) or move to a single Kubernetes CronJob/leader election.

**S2. Sending email inside the request slows API responses to 3 s.**
Publish an event / use `@Async` with a bounded executor, or better a queue (outbox + Kafka/RabbitMQ) with retries and DLQ for reliability.

---

## 15. Logging, Actuator, Observability

### Definition
- **Logging**: SLF4J API with Logback (default).
- **Actuator**: production-ready endpoints (health, metrics, info, env, loggers, threaddump, heapdump…).
- **Observability**: logs + metrics + traces (Micrometer, OpenTelemetry).

### Theory
**Logging best practices**: structured JSON logs (Logstash encoder; Boot 3.4+ supports structured logging natively), levels (ERROR/WARN/INFO/DEBUG), MDC for traceId/userId, never log secrets/PII, parameterized messages (`log.info("order {}", id)`).

**Actuator**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true          # /health/liveness, /health/readiness for Kubernetes
      show-details: when-authorized
  metrics:
    tags:
      application: ${spring.application.name}
```
Secure Actuator (separate management port, authentication). Never expose `env`, `heapdump` publicly.

**Micrometer**: vendor-neutral metrics → Prometheus → Grafana. Custom metrics: `Counter`, `Timer`, `Gauge`. **Tracing**: Micrometer Tracing + OpenTelemetry/Zipkin; propagates `traceparent` across services. Boot 3 supports the Observation API (`@Observed`).

**Golden signals**: latency, traffic, errors, saturation. Define SLIs/SLOs and alerts.

### Industry Example
```java
@Component
class OrderMetrics {
    private final Counter placed;
    OrderMetrics(MeterRegistry registry) {
        this.placed = Counter.builder("orders.placed").tag("channel", "web").register(registry);
    }
    void orderPlaced() { placed.increment(); }
}

@Component
class DbHealth implements HealthIndicator {
    public Health health() {
        return checkPrimaryDb() ? Health.up().build()
                                : Health.down().withDetail("db", "unreachable").build();
    }
}
```

### Interview Q&A
**Q1. Liveness vs readiness?**
Liveness: is the process healthy (restart if not). Readiness: can it receive traffic (remove from load balancer when dependencies are down/warming).

**Q2. How do you trace a request across microservices?**
Propagate trace context (W3C `traceparent`) via Micrometer Tracing/OpenTelemetry; include `traceId` in logs; view in Zipkin/Jaeger/Tempo.

**Q3. How do you change log level at runtime?**
`POST /actuator/loggers/com.company` with `{"configuredLevel":"DEBUG"}`.

### Scenario-Based Q&A
**S1. Intermittent 2-second latency spikes in production.**
Check metrics (p99), traces to find slow span, GC logs, thread dump (`/actuator/threaddump`), DB slow query log, connection pool wait time, downstream timeouts. Reproduce with load testing.

**S2. Pods keep restarting in Kubernetes right after deployment.**
Check liveness probe too aggressive vs startup time; use a `startupProbe` or readiness-only; check memory limits (OOMKilled), config/secret errors in logs.

---

# PART 6: TESTING

## 16. Testing Spring Boot Applications

### Definition
Verifying behaviour at different levels: unit, slice, integration, contract, end-to-end.

### Theory
**Test pyramid**: many unit tests, fewer slice/integration tests, few E2E.

| Annotation | Purpose |
|---|---|
| Plain JUnit 5 + Mockito | Unit tests, no Spring (fastest) |
| `@WebMvcTest` | Controller slice (MockMvc), mocks via `@MockBean`/`@MockitoBean` |
| `@DataJpaTest` | Repository slice, in-memory/Testcontainers DB, rolls back |
| `@JsonTest` | JSON serialization |
| `@SpringBootTest` | Full context; `webEnvironment=RANDOM_PORT` with `TestRestTemplate`/`WebTestClient` |
| `@ServiceConnection` + Testcontainers | Real Postgres/Kafka/Redis in tests (Boot 3.1+) |

Prefer Testcontainers over H2 to avoid dialect differences. **Contract testing**: Spring Cloud Contract or Pact. **WireMock** for external HTTP APIs.

### Industry Example
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;

    @Test
    void createsOrder() throws Exception {
        when(service.create(any())).thenReturn(new OrderResponse(1L, "PENDING"));

        mvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                  {"customerId":1,"items":[{"productId":10,"quantity":2}]}"""))
           .andExpect(status().isCreated())
           .andExpect(jsonPath("$.status").value("PENDING"));
    }
}
```
```java
@Testcontainers
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryIT {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @Autowired OrderRepository repo;

    @Test void findsByStatus() { /* ... */ }
}
```
Unit test with Mockito:
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @InjectMocks OrderService service;
    @Test void throwsWhenMissing() {
        when(repo.findById(1L)).thenReturn(Optional.empty());
        assertThrows(ResourceNotFoundException.class, () -> service.get(1L));
    }
}
```

### Interview Q&A
**Q1. `@Mock` vs `@MockBean`?**
`@Mock` (Mockito) creates a plain mock outside Spring. `@MockBean` registers/replaces a bean in the Spring context (slower, context cache is invalidated per unique config).

**Q2. `@SpringBootTest` vs `@WebMvcTest`?**
Full application context vs only web layer beans; slices are faster.

**Q3. Why Testcontainers rather than H2?**
Production-like behaviour (SQL dialect, JSON types, locking, migrations).

**Q4. How do you test security?**
`spring-security-test`: `@WithMockUser`, `jwt()` request post-processor, test 401/403 cases.

**Q5. How do you keep integration tests fast?**
Reuse containers (singleton pattern), reuse Spring context (avoid unneeded `@MockBean` variance), `@Transactional` rollback or clean via Flyway, parallelize carefully.

### Scenario-Based Q&A
**S1. Tests pass locally, fail on CI with date-related errors.**
Time/timezone dependency. Inject `java.time.Clock` and fix it in tests; set `-Duser.timezone=UTC`.

**S2. Third-party payment API is flaky in tests.**
Stub with WireMock or mock server, add contract tests, test timeouts/retries/circuit breaker behaviours explicitly.

---

# PART 7: DISTRIBUTED SYSTEMS

## 17. Messaging: Kafka / RabbitMQ

### Definition
**Message brokers** enable asynchronous, decoupled communication. Kafka is a distributed log (high throughput, replay). RabbitMQ is a broker with flexible routing (queues, exchanges).

### Theory
**Kafka concepts**: topic, partition, offset, producer, consumer group, broker, replication factor, ISR, retention, compacted topics. Ordering is guaranteed **per partition** (key-based partitioning). One partition is consumed by one consumer in a group, so parallelism ≤ partition count.

**Delivery semantics**: at-most-once, **at-least-once** (default practical), exactly-once (idempotent producer + transactions; end-to-end needs idempotent consumers/outbox).

**Reliability**: producer `acks=all`, `enable.idempotence=true`, retries; consumer manual/after-processing commits; **retry topics + DLT (dead letter topic)**; idempotent consumers (dedupe by event id); schema management (Avro/Protobuf + Schema Registry); backward compatible evolution.

**RabbitMQ**: exchange types (direct, topic, fanout, headers), acks, prefetch, DLX, TTL, quorum queues.

### Industry Example
```java
@Service
class OrderEventPublisher {
    private final KafkaTemplate<String, OrderPlaced> kafka;
    OrderEventPublisher(KafkaTemplate<String, OrderPlaced> kafka) { this.kafka = kafka; }

    void publish(OrderPlaced e) {
        kafka.send("orders.placed", e.orderId().toString(), e);   // key = orderId keeps order per order
    }
}

@Component
class InventoryConsumer {
    @RetryableTopic(attempts = "4", backoff = @Backoff(delay = 1000, multiplier = 2))
    @KafkaListener(topics = "orders.placed", groupId = "inventory-service")
    void on(OrderPlaced e) {
        if (processedRepo.existsById(e.eventId())) return;        // idempotency
        inventoryService.reserve(e);
        processedRepo.save(new ProcessedEvent(e.eventId()));
    }

    @DltHandler
    void dlt(OrderPlaced e) { alertService.notifyOps(e); }
}
```

### Interview Q&A
**Q1. How does Kafka guarantee ordering?**
Only within a partition. Use the same key for events that must remain ordered.

**Q2. How do you avoid duplicate processing?**
Idempotent consumers: unique event id with a processed-events table or natural idempotent operations (upserts), plus transactional processing.

**Q3. Kafka vs RabbitMQ?**
Kafka: event streaming, replay, very high throughput, long retention, partitioned scaling. RabbitMQ: task queues, complex routing, per-message acks, lower throughput.

**Q4. What is consumer lag and how to handle it?**
Difference between latest offset and committed offset. Fix by scaling consumers (up to partitions), optimizing processing, increasing partitions, batching.

**Q5. What if a poison message keeps failing?**
Bounded retries with backoff then send to DLT; alert and replay after fix.

### Scenario-Based Q&A
**S1. Consumers are lagging by 2 million messages after a downstream outage.**
Fix the downstream, scale consumers (≤ partitions), use batch listeners, increase `max.poll.records` carefully, ensure idempotency, temporarily pause non-critical processing, monitor lag.

**S2. Same order events processed out of order (cancel before create).**
Ensure `orderId` is the partition key; handle out-of-order tolerant state machines (version numbers/ignore stale events).

**S3. Need "exactly once" money transfers.**
Use idempotency keys, transactional outbox, idempotent consumers, and the Saga pattern; treat exactly-once as effectively-once through idempotency.

---

## 18. Microservices Patterns

### Definition
**Microservices**: small, independently deployable services organized around business capabilities, each owning its data.

### Theory
**When to use**: large teams, independent scaling/deploys. **When not**: small team/early product (a modular monolith is often better).

**Key patterns**
| Pattern | Purpose | Tooling |
|---|---|---|
| API Gateway | Single entry, routing, auth, rate-limit | Spring Cloud Gateway |
| Service Discovery | Find instances dynamically | Eureka, Consul, Kubernetes DNS |
| Centralized Config | External configs | Spring Cloud Config, Vault, K8s ConfigMaps |
| Declarative HTTP client | Service calls | OpenFeign, `RestClient`, HTTP interfaces |
| Circuit Breaker, Retry, Bulkhead, Rate Limiter, Time Limiter | Resilience | Resilience4j |
| Saga | Distributed transactions | Orchestration/choreography |
| Outbox + CDC | Reliable events | Debezium |
| CQRS / Event Sourcing | Separate read/write models | Kafka, Axon |
| Database per service | Autonomy | |
| Strangler Fig | Migrate monolith gradually | |
| BFF | Per-client backend | |

On Kubernetes, many teams skip Eureka/Config Server and use native discovery and ConfigMaps/Secrets.

**Circuit breaker states**: CLOSED → (failure rate over threshold) → OPEN (fail fast) → after wait → HALF_OPEN (trial calls) → CLOSED/OPEN.

**Resilience guidelines**: always set timeouts, retry only idempotent calls with exponential backoff + jitter, provide fallbacks, bulkhead to isolate resources.

### Industry Example
```java
@Service
class PricingClient {
    private final RestClient client;
    PricingClient(RestClient.Builder b) { this.client = b.baseUrl("http://pricing-service").build(); }

    @CircuitBreaker(name = "pricing", fallbackMethod = "cachedPrice")
    @Retry(name = "pricing")
    @TimeLimiter(name = "pricing")   // for async/CompletableFuture
    public Price price(Long productId) {
        return client.get().uri("/prices/{id}", productId).retrieve().body(Price.class);
    }

    Price cachedPrice(Long productId, Throwable t) { return priceCache.getOrDefault(productId); }
}
```
```yaml
resilience4j:
  circuitbreaker:
    instances:
      pricing:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 20s
        permitted-number-of-calls-in-half-open-state: 5
  retry:
    instances:
      pricing:
        max-attempts: 3
        wait-duration: 200ms
        enable-exponential-backoff: true
```
Gateway route:
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://order-service
          predicates: [ Path=/api/orders/** ]
          filters:
            - name: RequestRateLimiter
              args: { redis-rate-limiter.replenishRate: 20, redis-rate-limiter.burstCapacity: 40 }
```

**Saga (orchestration) example, e-commerce**
1. Order Service: create order (PENDING) → emit `OrderCreated`.
2. Payment: charge → `PaymentCompleted` / `PaymentFailed`.
3. Inventory: reserve → `StockReserved` / `StockFailed` (compensate: refund payment).
4. Order: mark CONFIRMED or CANCELLED.

### Interview Q&A
**Q1. How do microservices communicate?**
Synchronous: REST/gRPC (simple but couples availability). Asynchronous: events/messages (decoupled, resilient, eventual consistency).

**Q2. Choreography vs orchestration saga?**
Choreography: services react to events (loose coupling, harder to follow). Orchestration: a coordinator drives steps (clear flow, central logic).

**Q3. How do you handle data consistency across services?**
Eventual consistency via Saga, Outbox, idempotent consumers; avoid shared DBs and distributed transactions.

**Q4. How do you prevent cascading failures?**
Timeouts, circuit breakers, bulkheads, retries with backoff, rate limiting, fallbacks, load shedding, async messaging.

**Q5. How do you version and deploy safely?**
Backward-compatible contracts, consumer-driven contract tests, blue-green/canary deployments, feature flags.

**Q6. How do you query data that lives in multiple services?**
API composition (gateway/BFF), CQRS read models populated by events, or a search/reporting store; avoid cross-service joins.

**Q7. Monolith vs microservices?**
Monolith: simple deployment/debugging; scaling and team independence limited. Microservices: independent scale/deploys; operational complexity (observability, networking, consistency). Start with a modular monolith if unsure.

### Scenario-Based Q&A
**S1. Payment service is slow; order service threads are exhausted; whole site slows.**
Timeouts + circuit breaker + bulkhead on payment calls, async handling (queue), return "payment pending" and complete later, fallback paths, scale payment service, alerts on latency.

**S2. A retry caused a customer to be charged twice.**
Retries on non-idempotent POST are dangerous. Add idempotency keys at the payment service, retry only safe operations, make consumers idempotent, reconcile with payment provider records.

**S3. You need to split a 10-year-old monolith.**
Strangler Fig: put a gateway in front, extract bounded contexts one by one (start with low-risk, high-change areas), anti-corruption layer, data migration with CDC, keep the monolith running until traffic is fully moved.

**S4. Which service owns "customer name" used in 5 services?**
One service is the source of truth (Customer Service); others keep read-only copies updated through events (`CustomerUpdated`), accepting eventual consistency.

---

# PART 8: ADVANCED AND PRODUCTION

## 19. Reactive (WebFlux) and Virtual Threads

### Definition
- **WebFlux**: non-blocking reactive web stack (Reactor `Mono`/`Flux`, Netty).
- **Virtual Threads (Java 21, Project Loom)**: lightweight threads that make blocking code scalable; enable with `spring.threads.virtual.enabled=true` (Boot 3.2+).

### Theory
| | Spring MVC (thread-per-request) | WebFlux | MVC + Virtual Threads |
|---|---|---|---|
| Model | Blocking | Non-blocking | Blocking style, cheap threads |
| Code style | Imperative | Functional/reactive | Imperative |
| Best for | Typical CRUD | High concurrency streaming, many slow I/O calls | I/O-bound services without rewriting |
| Learning curve | Low | High | Low |

Never block inside reactive pipelines (JDBC, `Thread.sleep`). Use R2DBC for reactive DB. Beware of **pinning** with virtual threads (`synchronized` blocks around blocking I/O; use `ReentrantLock`), and ThreadLocal-heavy code.

### Industry Example
```java
@RestController
class StockStreamController {
    @GetMapping(value = "/stocks/{symbol}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    Flux<StockPrice> stream(@PathVariable String symbol) {
        return priceService.prices(symbol).delayElements(Duration.ofSeconds(1));
    }
}

// WebClient with timeout and retry
webClient.get().uri("/inventory/{id}", id)
    .retrieve().bodyToMono(Inventory.class)
    .timeout(Duration.ofSeconds(2))
    .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));
```

### Interview Q&A
**Q1. When should you choose WebFlux over MVC?**
Streaming/SSE/WebSockets, very high concurrent connections with mostly I/O waits, entire stack non-blocking. For typical CRUD with JPA, MVC (optionally with virtual threads) is simpler.

**Q2. `Mono` vs `Flux`?**
`Mono` emits 0..1 item; `Flux` emits 0..N.

**Q3. What happens if you block in a WebFlux handler?**
You block a small event-loop thread, hurting all requests. Offload with `publishOn(Schedulers.boundedElastic())` or avoid blocking.

**Q4. Do virtual threads replace reactive?**
For many apps, yes for scalability goals, since you keep simple blocking code. Reactive still wins for backpressure-aware streaming pipelines.

### Scenario-Based Q&A
**S1. The team wants WebFlux for a CRUD app using JPA.**
Mismatch: JPA is blocking. Prefer MVC with virtual threads (Java 21) unless streaming/backpressure is truly needed; if reactive, use R2DBC and reactive drivers consistently.

---

## 20. Docker, Kubernetes, GraalVM Native

### Definition
- **Docker**: packages the app and runtime into an image.
- **Kubernetes**: orchestrates containers (scaling, healing, rollout).
- **GraalVM Native Image**: ahead-of-time compiled executable (fast startup, low memory).

### Theory
**Docker best practices**: multi-stage builds, slim JRE base, non-root user, layered JARs, `JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75`, one process per container, immutable images. Buildpacks: `./mvnw spring-boot:build-image`.

**Kubernetes**: Deployment, Service, Ingress, ConfigMap/Secret, HPA (autoscale on CPU/custom metrics), liveness/readiness/startup probes, resource requests/limits, PodDisruptionBudget, graceful shutdown (`server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`), rolling updates.

**Native**: startup in milliseconds, but longer build times, reflection/proxies need hints (Spring AOT generates most), fewer dynamic features. Alternatives: CDS, CRaC.

### Industry Example
```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:21-jre
RUN useradd -r spring
USER spring
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75"
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```
```yaml
# Kubernetes snippet
resources:
  requests: { cpu: "500m", memory: "768Mi" }
  limits:   { memory: "1Gi" }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
```

### Interview Q&A
**Q1. How do you ensure zero-downtime deployments?**
Rolling/blue-green/canary, readiness probes, graceful shutdown, backward-compatible DB migrations, connection draining.

**Q2. What is graceful shutdown?**
On SIGTERM the server stops accepting new requests but lets in-flight requests complete within a timeout.

**Q3. JVM memory in containers?**
Use container-aware JVM flags (`MaxRAMPercentage`), set memory limits, watch for OOMKilled (heap + metaspace + threads + native memory exceed limit).

**Q4. Why or why not GraalVM native?**
Pros: fast startup, small memory (serverless, scale-to-zero). Cons: slower builds, limited reflection, peak throughput may be lower than JIT.

### Scenario-Based Q&A
**S1. Pod is OOMKilled although heap is only 60% used.**
Non-heap memory (metaspace, thread stacks, direct buffers, native libs) plus heap exceeded limit. Lower `MaxRAMPercentage`, cap threads/direct memory, raise limits, check native memory tracking.

**S2. Traffic spikes 10x during sales.**
HPA on CPU/RPS, pre-scale before known events, cache aggressively, queue-based load leveling, connection pool and DB capacity planning, rate limiting at the gateway.

---

## 21. Performance Tuning and Production Checklist

### Theory
**Performance checklist**
- Measure first (profilers: JFR, async-profiler, VisualVM; APM; load tests with Gatling/k6/JMeter).
- DB: indexes, avoid N+1, projections, batch operations, read replicas, proper pool size.
- Caching (Caffeine/Redis) with correct invalidation.
- Pagination everywhere; limit payload size; gzip compression.
- HTTP clients: pooled connections, timeouts, keep-alive.
- JVM: right GC (G1 default; ZGC for low-latency), heap sizing, GC logs.
- Async/virtual threads for I/O-bound work.
- Avoid heavy logging in hot paths.

**Production checklist**
- [ ] Config externalized, secrets in a vault
- [ ] Health, metrics, tracing, structured logs, alerts
- [ ] Timeouts/retries/circuit breakers on all remote calls
- [ ] Graceful shutdown, probes, resource limits
- [ ] DB migrations via Flyway, backups, tested restore
- [ ] Security: HTTPS, authN/authZ, input validation, dependency scanning
- [ ] CI/CD with tests, static analysis (SonarQube), image scanning
- [ ] Runbooks, SLOs, on-call process

### Interview Q&A
**Q1. How do you troubleshoot high CPU in production?**
Identify the process/thread (`top -H`), take thread dumps (`jstack`/actuator), map hot thread id (hex) to stack, check GC activity, JFR profile, correlate with recent deploys/traffic.

**Q2. How do you investigate a memory leak?**
Monitor heap growth after GC, capture heap dump (`jmap`/actuator heapdump, `-XX:+HeapDumpOnOutOfMemoryError`), analyze with Eclipse MAT for dominator tree: usual suspects are unbounded caches, static collections, ThreadLocals, listeners.

**Q3. What causes slow startup and how do you fix it?**
Many beans/classpath scanning, heavy `@PostConstruct`, eager DB/remote calls. Use lazy init, AOT/CDS, trim dependencies, move warm-up async.

### Scenario-Based Q&A
**S1. API p99 latency doubled after a release.**
Compare traces before/after, check new queries (N+1), new remote calls without timeouts, lock contention, GC changes, config diffs. Roll back if customer impact is high, then fix forward with tests.

**S2. DB CPU at 95% during peak.**
Find top queries (slow query log/pg_stat_statements), add indexes, cache hot reads, move reads to replicas, batch writes, reduce chatty calls, tune pool.

---

## 22. Spring Boot 3 Specifics

- **Baseline**: Java 17+, Jakarta EE 9+ (`javax.*` → `jakarta.*`), Spring Framework 6.
- **Spring Security 6**: lambda DSL, `SecurityFilterChain` bean, `authorizeHttpRequests`, `requestMatchers`; `WebSecurityConfigurerAdapter` removed.
- **ProblemDetail** (RFC 7807) for errors.
- **HTTP interface clients** (`@HttpExchange`) and `RestClient` (6.1) as a modern alternative to `RestTemplate`/Feign.
- **Observability**: Micrometer Observation API + Tracing replaces Spring Cloud Sleuth.
- **AOT + GraalVM native** support.
- **Virtual threads** (3.2+ with Java 21), `JdbcClient` (6.1), Testcontainers `@ServiceConnection` (3.1+), Docker Compose support (`spring-boot-docker-compose`) for local dev, structured logging (3.4+), CDS support.
- **Auto-configuration registration** moved to `AutoConfiguration.imports`.
- `spring.factories` no longer used for auto-config registration.

### Interview Q&A
**Q1. Main migration steps from Boot 2 to 3?**
Upgrade to Java 17+, go to latest 2.7.x first, replace `javax` imports with `jakarta`, update Security config to the new DSL, replace Sleuth with Micrometer Tracing, update third-party libs (Hibernate 6 query changes, Flyway, springdoc), run OpenRewrite recipes, fix deprecated properties via `spring-boot-properties-migrator`.

**Q2. `RestTemplate` vs `WebClient` vs `RestClient`?**
`RestTemplate`: blocking, maintenance mode. `WebClient`: reactive/non-blocking (also usable synchronously). `RestClient`: modern fluent blocking client (Spring 6.1), recommended for new blocking code.

---

# 23. Rapid-Fire Interview Questions

1. **What is dependency injection?** Supplying a class's dependencies from outside, so it does not create them itself.
2. **`@Autowired` required?** Not for a single constructor.
3. **Default server port and how to change?** 8080; `server.port=9090`.
4. **Default embedded server?** Tomcat.
5. **What is `application.yml` vs `bootstrap.yml`?** Bootstrap was for early Spring Cloud config loading; now replaced by `spring.config.import`.
6. **What is `@SpringBootTest`?** Loads full application context for integration tests.
7. **`CommandLineRunner` vs `ApplicationRunner`?** Raw `String[]` args vs parsed `ApplicationArguments`.
8. **How to make a bean load only in dev?** `@Profile("dev")`.
9. **What is Spring Boot DevTools?** Auto-restart, live reload for development, disabled in production jars.
10. **What is `@Transactional(readOnly = true)` benefit?** Skips dirty checking, enables read-replica routing.
11. **Difference between `@Component` and `@Service`?** Functionally same; `@Service` documents intent.
12. **What is `@Primary`?** Marks the preferred bean when multiple candidates exist.
13. **What is `@Qualifier`?** Selects a specific bean by name/qualifier.
14. **What is `@Lazy`?** Defers bean creation until first use.
15. **What is `@ConditionalOnMissingBean`?** Registers a bean only if the user hasn't defined one.
16. **What is a Spring Boot fat JAR?** Executable JAR containing the app, dependencies, and launcher.
17. **How do you handle CORS?** `@CrossOrigin` or global `CorsConfigurationSource` in security config.
18. **How to secure Actuator?** Restrict exposure, put behind auth, use a separate management port, network policies.
19. **What are idempotent HTTP methods?** GET, PUT, DELETE, HEAD, OPTIONS (repeating gives same server state).
20. **How do you implement soft delete?** `deleted` flag/timestamp with `@SQLDelete` and `@SQLRestriction` (Hibernate 6.3+; `@Where` earlier).
21. **What is `Optional` best practice in repositories?** Return `Optional` from finders; do not use as entity fields/parameters.
22. **Equals/hashCode for entities?** Use stable business key or id; avoid mutable fields.
23. **How do you do audit fields (createdAt/By)?** `@EnableJpaAuditing`, `@CreatedDate`, `@LastModifiedBy`, `AuditorAware`.
24. **How do you do file storage?** Do not store in DB/local disk in containers; use S3/GCS/Blob storage.
25. **What is HikariCP default pool size?** 10.
26. **What is Spring Cloud?** Toolkit for distributed-system patterns (config, gateway, discovery, circuit breakers) built on Boot.
27. **SOLID in Spring?** DIP through DI; SRP via layers; OCP through strategy beans.
28. **How do you do feature flags?** Config properties, `@ConditionalOnProperty`, Unleash/LaunchDarkly/FF4J.
29. **What is `@ControllerAdvice` order?** Use `@Order` when multiple advices exist.
30. **Difference between `@RequestBody` and `@ModelAttribute`?** JSON body deserialization vs form/query parameter binding.

---

# 24. Cheat Sheet and Learning Roadmap

### Annotation Cheat Sheet
| Area | Annotations |
|---|---|
| Core | `@SpringBootApplication`, `@Component`, `@Service`, `@Repository`, `@Configuration`, `@Bean`, `@Autowired`, `@Qualifier`, `@Primary`, `@Lazy`, `@Scope`, `@Profile`, `@Value`, `@ConfigurationProperties` |
| Web | `@RestController`, `@RequestMapping`, `@Get/Post/Put/Patch/DeleteMapping`, `@PathVariable`, `@RequestParam`, `@RequestBody`, `@ResponseStatus`, `@RestControllerAdvice`, `@ExceptionHandler`, `@CrossOrigin` |
| Validation | `@Valid`, `@Validated`, `@NotNull`, `@NotBlank`, `@Size`, `@Email`, `@Pattern` |
| Data | `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`, `@OneToMany`, `@ManyToOne`, `@JoinColumn`, `@Version`, `@Query`, `@Modifying`, `@EntityGraph`, `@Transactional` |
| Security | `@EnableMethodSecurity`, `@PreAuthorize`, `@PostAuthorize`, `@Secured` |
| Cross-cutting | `@Aspect`, `@Around`, `@Cacheable`, `@CacheEvict`, `@Async`, `@Scheduled`, `@EventListener`, `@TransactionalEventListener` |
| Testing | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@MockBean`, `@Testcontainers`, `@ServiceConnection`, `@WithMockUser` |
| Resilience | `@CircuitBreaker`, `@Retry`, `@RateLimiter`, `@Bulkhead`, `@TimeLimiter` |
| Messaging | `@KafkaListener`, `@RetryableTopic`, `@RabbitListener` |

### Learning Roadmap
1. **Weeks 1–2: Foundations**: Java 17/21 features, Maven/Gradle, IoC/DI, beans, profiles, configuration.
2. **Weeks 3–4: Web + Data**: REST, validation, exception handling, JPA, transactions, Flyway, N+1, pagination.
3. **Week 5: Security**: Spring Security, JWT, OAuth2/Keycloak, method security.
4. **Week 6: Quality**: JUnit 5, Mockito, slice tests, Testcontainers, logging, Actuator.
5. **Weeks 7–8: Distributed**: Kafka/RabbitMQ, Redis caching, Resilience4j, Gateway, Saga/Outbox, tracing.
6. **Weeks 9–10: Production**: Docker, Kubernetes, CI/CD, performance tuning, observability, virtual threads/WebFlux.
7. **Ongoing**: build a capstone (e-commerce: user, catalog, order, payment, notification services) and practice system-design questions.

### Capstone Project Idea
**E-commerce platform**: API Gateway → User (JWT/Keycloak), Catalog (Postgres + Redis cache), Order (JPA, Saga orchestrator), Payment (idempotent, circuit breaker), Notification (Kafka consumer, email). Add Flyway, Testcontainers tests, Actuator + Prometheus + Grafana, Dockerfiles, Kubernetes manifests, GitHub Actions pipeline.

### Interview Strategy Tips (for senior/lead roles)
- Explain **trade-offs**, not just definitions ("I chose X because of Y; the cost is Z").
- Tie answers to production incidents you handled: symptoms → diagnosis → fix → prevention.
- Be ready for system-design: scale, consistency, failure handling, observability, security, cost.
- For leadership roles, add: code review standards, testing strategy, mentoring, release/rollback strategy, tech-debt management, estimation and stakeholder communication.

---

*End of guide.*