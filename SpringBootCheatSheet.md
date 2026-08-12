# Spring Boot interview quick reference — senior / lead / architect level

## 1. Core Spring & IoC fundamentals

**IoC container & DI**
- Inversion of Control — framework controls object creation/lifecycle instead of the developer
- `ApplicationContext` (vs older `BeanFactory`) — eager singleton init, supports events, AOP, i18n
- DI types: constructor injection (preferred — immutability, testability, avoids circular dependency masking), setter injection, field injection (avoid — hard to test, hides dependencies)

**Bean scopes**
| Scope | Lifecycle |
|---|---|
| `singleton` (default) | One instance per container |
| `prototype` | New instance every request |
| `request` | One per HTTP request (web-aware) |
| `session` | One per HTTP session |
| `application` | One per ServletContext |

**Bean lifecycle**
`Instantiate → Populate properties → Aware interfaces (BeanNameAware, etc.) → BeanPostProcessor.before → @PostConstruct/InitializingBean → BeanPostProcessor.after → Ready → @PreDestroy/DisposableBean on shutdown`

**Circular dependency**
- Spring resolves via early bean reference / 3-level cache — but only for singleton + setter/field injection
- Constructor injection circular dependency → fails at startup, forces you to refactor (generally a design smell — should be resolved with `@Lazy`, restructuring, or an event/facade)

## 2. Spring Boot auto-configuration & internals

**How auto-configuration works**
- `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
- `@EnableAutoConfiguration` triggers scanning of `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+; older versions used `spring.factories`)
- Conditional annotations gate each auto-config class: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, `@ConditionalOnBean`
- Auto-config classes are ordered last (lowest precedence) so user-defined beans always win via `@ConditionalOnMissingBean`

**Starter dependencies**
- `spring-boot-starter-*` are curated dependency bundles with tested version compatibility (managed via `spring-boot-dependencies` BOM)

**Externalized configuration precedence** (high to low, common ones)
1. Command-line args
2. `SPRING_APPLICATION_JSON`
3. Environment variables
4. `application-{profile}.yml/properties`
5. `application.yml/properties`
6. `@PropertySource` on `@Configuration`
7. Default properties

**Profiles**
- `@Profile("dev")`, `spring.profiles.active`, profile-specific YAML — used for environment-specific bean wiring

## 3. REST API design & web layer

- `@RestController` = `@Controller` + `@ResponseBody`
- `@RequestMapping`/`@GetMapping` etc., `@PathVariable`, `@RequestParam`, `@RequestBody`
- Global exception handling — `@ControllerAdvice` + `@ExceptionHandler`, returning `ProblemDetail` (RFC 7807, Spring 6+) or custom error DTOs
- Validation — `@Valid`/`@Validated` with Bean Validation (`@NotNull`, `@Size`, custom `ConstraintValidator`)
- Content negotiation — `Accept` header, `produces`/`consumes`
- HATEOAS — Spring HATEOAS for hypermedia-driven APIs (less common in modern microservices, but asked about at architect level)
- API versioning strategies — URI versioning, header versioning, content negotiation versioning — trade-offs each interviewer expects you to discuss

## 4. Data access — Spring Data JPA / transactions

**JPA/Hibernate essentials**
- Entity lifecycle states: Transient → Persistent (managed) → Detached → Removed
- `@Entity`, `@Id`, relationship mappings (`@OneToMany`, `@ManyToOne`, `@ManyToMany`) — know `mappedBy`, cascade types, `fetch = LAZY` vs `EAGER` (default LAZY for collections, EAGER for `@ManyToOne`/`@OneToOne` — usually override to LAZY)
- **N+1 query problem** — classic senior-level question; solved via `JOIN FETCH`, `@EntityGraph`, batch fetching (`hibernate.default_batch_fetch_size`)
- First-level cache (session/persistence context, always on) vs second-level cache (`@Cacheable`, opt-in, e.g. Ehcache/Redis)

**Transactions**
- `@Transactional` — proxy-based (JDK dynamic proxy or CGLIB); **self-invocation doesn't trigger it** (calling a `@Transactional` method from within the same class bypasses the proxy) — a very common gotcha question
- Propagation types: `REQUIRED` (default), `REQUIRES_NEW`, `NESTED`, `SUPPORTS`, `MANDATORY`, `NEVER`, `NOT_SUPPORTED`
- Isolation levels: `READ_UNCOMMITTED`, `READ_COMMITTED` (most DB defaults), `REPEATABLE_READ`, `SERIALIZABLE` — trade-off between consistency and concurrency
- Rollback rules — by default rolls back on unchecked exceptions only; use `rollbackFor` for checked exceptions
- Optimistic locking (`@Version`) vs pessimistic locking (`@Lock(LockModeType.PESSIMISTIC_WRITE)`) — when to use each (low vs high contention)

**Query methods**
- Derived query methods, `@Query` (JPQL/native), `Specification` API for dynamic queries, `Pageable`/`Sort`

## 5. Microservices architecture (architect-level focus)

**Core patterns**
- Service decomposition — by business capability / DDD bounded context
- **Database per service** — no shared DB; data consistency across services via events, not distributed transactions
- **Saga pattern** — choreography (event-driven, decentralized) vs orchestration (central coordinator) — for distributed transactions without 2PC
- **CQRS** — separate read/write models, often paired with event sourcing
- **API Gateway** — single entry point, routing, auth, rate limiting (Spring Cloud Gateway)
- **Backend for Frontend (BFF)** — gateway tailored per client type

**Service discovery & config**
- Eureka / Consul — client-side discovery; `@EnableDiscoveryClient`
- Spring Cloud Config Server — centralized externalized config, refreshable via `/actuator/refresh` or Spring Cloud Bus

**Resilience patterns (Resilience4j — replaced Hystrix)**
- **Circuit breaker** — states: Closed → Open → Half-Open; prevents cascading failures
- **Retry** — with exponential backoff
- **Bulkhead** — isolate thread pools per dependency to prevent resource exhaustion
- **Rate limiter** — cap request throughput
- **Timeout** — fail fast on slow dependencies
- Combine annotations: `@CircuitBreaker`, `@Retry`, `@Bulkhead`, `@RateLimiter` — order of application matters

**Inter-service communication**
- Synchronous — REST (`RestTemplate` legacy, `WebClient` reactive/non-blocking, `RestClient` new in Spring 6.1), gRPC (performance-sensitive)
- Asynchronous — Kafka, RabbitMQ — event-driven decoupling, better resilience to downstream outages
- Idempotency — critical for retries in distributed systems (idempotency keys, dedup tables)

## 6. Spring Security

- Filter chain — `SecurityFilterChain` bean (replaces `WebSecurityConfigurerAdapter`, deprecated since Spring Security 5.7)
- Authentication vs authorization — `AuthenticationManager`/`AuthenticationProvider` vs `AccessDecisionManager`/method security (`@PreAuthorize`, `@PostAuthorize`)
- **JWT-based stateless auth** — typical for microservices; token validation filter, no server-side session
- OAuth2 / OpenID Connect — Authorization Code flow (standard for web apps), Client Credentials (service-to-service)
- CORS, CSRF — CSRF typically disabled for stateless REST APIs (no cookies/session), must be enabled for session-based apps
- Method-level security — `@PreAuthorize("hasRole('ADMIN')")`

## 7. Actuator, observability & monitoring

- `/actuator/health`, `/metrics`, `/info`, `/env` — production-readiness endpoints
- Custom health indicators — implement `HealthIndicator`
- **Micrometer** — vendor-neutral metrics facade, integrates with Prometheus/Grafana, Datadog, etc.
- Distributed tracing — Micrometer Tracing (replaced Spring Cloud Sleuth) + Zipkin/Jaeger — trace ID propagation across service boundaries
- Structured logging, correlation IDs (MDC — `MDC.put()`) for tracing requests across logs
- Centralized logging — ELK/EFK stack, log aggregation strategy in distributed systems

## 8. Caching

- `@EnableCaching`, `@Cacheable`, `@CacheEvict`, `@CachePut`
- Cache providers — Caffeine (in-memory, high performance), Redis (distributed cache, shared across instances)
- Cache invalidation strategies — TTL, write-through, write-behind, cache-aside (most common pattern)
- Distributed cache consistency challenges — stale reads across service instances

## 9. Messaging & event-driven design

- Spring Kafka / Spring AMQP (RabbitMQ)
- **At-least-once vs exactly-once vs at-most-once** delivery semantics
- Consumer group rebalancing, partition strategy (Kafka), dead-letter queues for poison messages
- Outbox pattern — ensures atomicity between DB write and event publish (avoids dual-write problem)
- Event sourcing — storing state as a sequence of events rather than current state snapshot

## 10. Testing strategy

- `@SpringBootTest` — full context load, integration tests
- `@WebMvcTest` — slice test, web layer only (mocks service layer)
- `@DataJpaTest` — slice test, JPA repository layer with embedded DB
- `Mockito` — `@Mock`, `@InjectMocks` for unit tests
- **Testcontainers** — spin up real DB/Kafka/Redis in Docker for integration tests — expected knowledge at senior level, replaces brittle in-memory DB tests
- Contract testing (Spring Cloud Contract / Pact) — verify producer-consumer API compatibility in microservices without full integration environment
- Test pyramid — unit (many, fast) → integration (fewer) → E2E (fewest, slowest) — architects are expected to defend this ratio

## 11. Performance & scalability

- Connection pooling — HikariCP (default since Boot 2), tuning pool size relative to DB max connections and thread count
- **Virtual threads (Java 21 + Spring Boot 3.2+)** — `spring.threads.virtual.enabled=true`; huge for I/O-bound blocking workloads, reduces need for reactive complexity in many cases
- Reactive stack (WebFlux) vs Servlet stack (MVC) — when to choose reactive: high-concurrency I/O-bound, streaming; trade-off: steeper learning curve, harder debugging, mixing blocking calls breaks the event loop
- Horizontal scaling — statelessness is critical, externalize session state (Redis) if session affinity can't be guaranteed
- JVM tuning — heap sizing, GC selection (G1/ZGC), thread pool sizing formulas
- Database read replicas, connection pool exhaustion diagnosis

## 12. Deployment & DevOps

- Building containers — Buildpacks (`spring-boot:build-image`) vs Dockerfile
- Layered JARs — `spring-boot:repackage` layered mode for better Docker layer caching
- Graceful shutdown — `server.shutdown=graceful`, drains in-flight requests before termination
- Kubernetes readiness/liveness probes — mapped to actuator health groups (`management.endpoint.health.probes.enabled=true`)
- Blue-green / canary deployments, rolling updates
- Config/secrets management — Kubernetes ConfigMaps/Secrets, Vault integration

## 13. Common architect-level discussion questions

- "How would you design a system to handle X requests/sec with Y availability?" — expect to discuss load balancing, caching layers, DB sharding/read replicas, async processing, circuit breakers
- "How do you handle distributed transactions across microservices?" — Saga pattern, eventual consistency, outbox pattern — **not** 2PC (rarely practical at scale)
- "How do you version APIs without breaking consumers?" — backward-compatible changes, deprecation policy, consumer-driven contracts
- "How do you ensure zero-downtime deployments?" — rolling updates, feature flags, DB migration strategy (expand/contract pattern for schema changes)
- "Monolith to microservices — how would you approach it?" — Strangler Fig pattern, identify bounded contexts, extract incrementally, avoid distributed monolith anti-pattern
- "How do you debug a production issue across 10 microservices?" — distributed tracing, correlation IDs, centralized logs, metrics dashboards

## 14. Common gotchas & "gotcha" interview questions

- `@Transactional` self-invocation bypass (proxy limitation)
- `@Autowired` field injection vs constructor injection — testability argument
- Circular bean dependency — usually a design smell, not just a technical fix
- `@Async` methods also proxy-based — same self-invocation caveat, and exceptions in `void` async methods are swallowed unless `AsyncUncaughtExceptionHandler` configured
- Bean initialization order issues — `@DependsOn`, `@Lazy`
- `equals()`/`hashCode()` on JPA entities — must be careful with lazy-loaded proxies and mutable IDs (common bug source)
- Component scanning package placement — `@SpringBootApplication` only scans sub-packages by default

## 15. Design patterns commonly discussed in Spring context

- Singleton — default bean scope
- Factory — `BeanFactory`, `FactoryBean` interface
- Proxy — AOP, `@Transactional`, `@Async`, lazy loading
- Template method — `JdbcTemplate`, `RestTemplate`
- Observer — `ApplicationEvent`/`ApplicationListener`
- Strategy — pluggable `Sort`/`Comparator` beans, multiple implementations wired via `@Qualifier`
- Decorator — `HandlerInterceptor`, filters