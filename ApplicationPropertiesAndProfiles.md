# Application Properties & Profiles — Q&A

---

**Q1. `application.properties` vs `application.yml` — what's the difference?**

Both serve the same purpose — externalizing configuration — but differ in syntax and capability:

| | `.properties` | `.yml` |
|---|---|---|
| Format | Flat key-value pairs | Hierarchical (indentation-based) |
| Readability for nested config | Repetitive, verbose | Cleaner, avoids key repetition |
| Lists/arrays | `list[0]=a`, `list[1]=b` | Native `- a` / `- b` syntax |
| Multiple profiles in one file | Not natively supported | Supported via `---` document separators |
| Comments | `#` | `#` |
| Parsing | Simpler, less error-prone | Sensitive to indentation (tabs not allowed) |

**Example — same config, both formats:**
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/db
spring.datasource.username=root
spring.jpa.hibernate.ddl-auto=update
```
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db
    username: root
  jpa:
    hibernate:
      ddl-auto: update
```
YAML is generally preferred in larger projects for readability; `.properties` is sometimes preferred for simplicity and because a stray space/indent can't silently break it.

---

**Q2. What is Externalized Configuration?**

It's Spring Boot's mechanism for keeping configuration values (URLs, credentials, feature flags, thresholds) **outside** the compiled code, so the same JAR/artifact can run in different environments (dev, staging, prod) without recompilation. Spring Boot supports pulling config from many sources — property files, environment variables, command-line args, config servers — and merges them according to a defined precedence order.

**Why it matters:** it enables the "build once, deploy everywhere" principle — you don't rebuild the app per environment, you just change external config.

---

**Q3. What is the `@Value` Annotation?**

`@Value` injects a single property value directly into a field, constructor parameter, or method parameter.

```java
@Component
public class MailService {
    @Value("${mail.host}")
    private String mailHost;

    @Value("${mail.port:25}")   // 25 = default if property is missing
    private int mailPort;
}
```

It also supports **SpEL (Spring Expression Language)**:
```java
@Value("#{systemProperties['user.timezone']}")
private String timezone;
```

**Limitation:** it's best for injecting one-off, simple values. For a group of related properties, `@ConfigurationProperties` is the better choice (type-safe, less repetitive, easier to validate).

---

**Q4. What is `@ConfigurationProperties`?**

It binds a whole block of related properties (a prefix) to a POJO, rather than injecting values one field at a time.

```yaml
app:
  mail:
    host: smtp.example.com
    port: 587
    username: admin
```
```java
@Component
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    private int port;
    private String username;
    // getters and setters
}
```

**Advantages over `@Value`:**
- Type-safe binding, including nested objects and lists
- Supports `@Validated` + JSR-303 annotations (`@NotNull`, `@Min`, etc.) for validating config at startup
- Cleaner — no repeated `@Value("${app.mail...}")` scattered across the class
- Relaxed binding — `app.mail.host`, `app.mail-host`, and `APP_MAIL_HOST` (env var) all bind to the same field

---

**Q5. What are Spring Profiles?**

Profiles let you define **environment-specific beans and configuration** that are only active under a given named profile (e.g., `dev`, `staging`, `prod`). This means you can have different `DataSource` configs, logging levels, or feature toggles per environment, all within the same codebase, and switch between them by activating a profile — no code change required.

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb

# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/app
```

Activate via:
```
spring.profiles.active=dev
```
or as a JVM arg: `-Dspring.profiles.active=prod`, or an environment variable `SPRING_PROFILES_ACTIVE=prod`.

---

**Q6. What is the `@Profile` Annotation?**

It restricts a `@Component` or `@Bean` to only be registered when a specific profile is active.

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        return DataSourceBuilder.create().url("jdbc:mysql://prod-db/app").build();
    }
}
```

You can also negate: `@Profile("!prod")` means "active in every profile except prod" — useful for mock/stub beans you want everywhere except production.

---

**Q7. What is Environment-Specific Configuration, and how is it structured in Spring Boot?**

It refers to the practice of maintaining separate configuration per deployment environment while keeping shared/common config in one place. Spring Boot's convention:

```
application.yml              → common config, shared across all profiles
application-dev.yml          → dev-only overrides
application-staging.yml      → staging-only overrides
application-prod.yml         → prod-only overrides
```

When a profile is active, Spring Boot loads `application.yml` first, then layers the profile-specific file on top — profile-specific values **override** the common ones for matching keys. This avoids duplicating shared config (like app name, common timeouts) across every environment file.

---

**Q8. What is Property Precedence Order in Spring Boot?**

When the same property is defined in multiple places, Spring Boot resolves conflicts using a defined precedence (highest to lowest, abbreviated to the most commonly tested ones):

1. **Command-line arguments** (`--server.port=9090`)
2. **`SPRING_APPLICATION_JSON`** (inline JSON env property)
3. **`ServletConfig` / `ServletContext` init parameters**
4. **JNDI attributes**
5. **Java System properties** (`-Dserver.port=9090`)
6. **OS environment variables** (`SERVER_PORT=9090`)
7. **Profile-specific properties outside the packaged jar** (`application-{profile}.yml` in an external location)
8. **Profile-specific properties inside the packaged jar**
9. **Application properties outside the packaged jar** (`application.yml` external)
10. **Application properties inside the packaged jar** (`application.yml` on classpath)
11. **`@PropertySource` annotations**
12. **Default properties** (set via `SpringApplication.setDefaultProperties`)

**The practical takeaway interviewers look for:** *external* configuration always wins over what's packaged inside the JAR, and command-line args/env vars are the highest-priority, most common way to override config in containerized/cloud deployments (e.g., Kubernetes ConfigMaps → env vars).

---

## Extra / Intelligent Interview Questions

**Q9. If the same property exists in `application.yml` and `application-prod.yml`, and the `prod` profile is active, which wins?**

`application-prod.yml` wins — profile-specific files override the base `application.yml` for any overlapping keys. Non-overlapping keys from `application.yml` still apply (they're merged, not replaced entirely).

---

**Q10. How would you inject a list or a nested object using `@ConfigurationProperties`?**

```yaml
app:
  servers:
    - name: server1
      url: http://s1.example.com
    - name: server2
      url: http://s2.example.com
```
```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private List<Server> servers;
    // getter/setter

    public static class Server {
        private String name;
        private String url;
        // getters/setters
    }
}
```
`@ConfigurationProperties` handles this natively via relaxed binding; `@Value` cannot bind lists of complex objects this way, which is a key reason it's preferred for structured config.

---

**Q11. Can you activate multiple profiles at once? What happens if they define conflicting beans?**

Yes: `spring.profiles.active=dev,feature-x`. Both profiles' beans/config are active simultaneously. If two active profiles define a bean of the **same type with the same name**, it results in ambiguity/conflict (similar to a duplicate bean definition) — Spring Boot doesn't automatically pick a "winner" between conflicting bean *definitions* from different profiles the way it merges *properties*. This is why profile-specific beans should generally target distinct concerns rather than overlapping ones.

---

**Q12. What's the difference between `@Value` and `Environment.getProperty()`?**

- `@Value` — declarative, injects the value at bean creation time into a field/parameter
- `Environment.getProperty("key")` — programmatic, lets you fetch a property value at runtime inside any method, useful when the property key itself might be dynamic (e.g., built from a variable) or when you need conditional logic based on presence/absence of a property

```java
@Autowired
private Environment env;

public void checkFeature() {
    String flag = env.getProperty("feature.enabled", "false");
}
```

---

**Q13. How does Spring Boot support "relaxed binding" for property names, and why does it matter?**

Relaxed binding means Spring Boot treats different naming conventions as equivalent when binding to `@ConfigurationProperties`:
- `app.mail-host`, `app.mailHost`, `app.mail_host`, and env var `APP_MAILHOST` all bind to a field named `mailHost`

This matters because environment variables (commonly used in Docker/Kubernetes) can't contain dots or mixed case reliably, so `APP_MAIL_HOST` still correctly maps to `app.mail-host` in YAML — letting you configure the same property consistently across property files, env vars, and command-line args without maintaining separate naming schemes.

---

**Q14. Why might you choose environment variables over a config file for secrets like database passwords?**

- Avoids committing secrets into version control (property files often live in the repo)
- Plays naturally with container orchestration secrets management (Kubernetes Secrets, Docker secrets injected as env vars)
- Higher in the precedence order, so it reliably overrides any default/packaged value without editing the JAR
- Keeps sensitive values out of build artifacts entirely

In production, a common pattern is: non-sensitive config lives in `application-prod.yml`, sensitive values (passwords, API keys) are injected purely via environment variables or a secrets manager, never hardcoded.

---

**Q15. What happens if a property referenced by `@Value` doesn't exist and no default is provided?**

Spring throws an `IllegalArgumentException` ("Could not resolve placeholder") at application startup, because the placeholder `${...}` cannot be resolved. This is actually useful — it fails fast rather than injecting `null` silently and causing a harder-to-diagnose `NullPointerException` later at runtime. Providing a default (`${key:defaultValue}`) is the way to make it optional.

---

**Q16. How can you validate configuration properties at startup and fail fast if they're invalid?**

Combine `@ConfigurationProperties` with `@Validated` and JSR-303 annotations:
```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {
    @NotBlank
    private String host;

    @Min(1) @Max(65535)
    private int port;
}
```
If `app.mail.host` is blank or `app.mail.port` is out of range, the application **fails to start** with a clear validation error — far better than discovering a misconfiguration deep into runtime.

---

*Interview tip: A strong signal to interviewers is connecting these concepts to real deployment practice — e.g., explaining precedence order in terms of "Kubernetes ConfigMap sets an env var, which overrides what's baked into the JAR" shows practical, not just textbook, understanding.*

# Spring Boot Microservice Integration Guide
### DB, Docker, Kafka, Redis, Eureka, Security, Tracing & More

This document covers a production-style `application.properties` setup for a Spring Boot microservice, along with a companion `docker-compose.yml` to run all dependencies locally.

---

## 1. Application Info

```properties
spring.application.name=order-service
server.port=8081
server.servlet.context-path=/api/v1
spring.profiles.active=dev
```

---

## 2. Database Integration (MySQL)

```properties
spring.datasource.url=jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME:orderdb}?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
spring.datasource.username=${DB_USERNAME:root}
spring.datasource.password=${DB_PASSWORD:root}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

**HikariCP Connection Pool**
```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.idle-timeout=30000
spring.datasource.hikari.connection-timeout=20000
spring.datasource.hikari.pool-name=OrderServiceHikariPool
```

**JPA / Hibernate**
```properties
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
spring.jpa.open-in-view=false
```

**Flyway Migrations** (recommended over `ddl-auto` in production)
```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
```

**PostgreSQL Alternative**
```properties
# spring.datasource.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:orderdb}
# spring.datasource.driver-class-name=org.postgresql.Driver
# spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

---

## 3. Redis (Caching / Session Store)

```properties
spring.redis.host=${REDIS_HOST:localhost}
spring.redis.port=${REDIS_PORT:6379}
spring.redis.password=${REDIS_PASSWORD:}
spring.redis.timeout=6000
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
```

---

## 4. Kafka Integration

```properties
spring.kafka.bootstrap-servers=${KAFKA_BROKER:localhost:9092}
```

**Producer**
```properties
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.producer.acks=all
spring.kafka.producer.retries=3
spring.kafka.producer.properties.enable.idempotence=true
```

**Consumer**
```properties
spring.kafka.consumer.group-id=order-service-group
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.properties.spring.json.trusted.packages=*
```

**Custom Topics**
```properties
app.kafka.topic.order-created=order-created-topic
app.kafka.topic.order-cancelled=order-cancelled-topic
```

---

## 5. RabbitMQ (Alternative/Additional Broker)

```properties
spring.rabbitmq.host=${RABBITMQ_HOST:localhost}
spring.rabbitmq.port=${RABBITMQ_PORT:5672}
spring.rabbitmq.username=${RABBITMQ_USER:guest}
spring.rabbitmq.password=${RABBITMQ_PASS:guest}
```

---

## 6. Service Discovery — Eureka Client

```properties
eureka.client.service-url.defaultZone=${EUREKA_URI:http://localhost:8761/eureka/}
eureka.instance.prefer-ip-address=true
eureka.client.fetch-registry=true
eureka.client.register-with-eureka=true
```

---

## 7. Config Server (Spring Cloud Config)

```properties
spring.config.import=optional:configserver:${CONFIG_SERVER_URI:http://localhost:8888}
spring.cloud.config.fail-fast=true
spring.cloud.config.retry.max-attempts=5
```

---

## 8. API Gateway Routing (Reference Only)

Defined on the gateway service itself, shown here for context:

```properties
# spring.cloud.gateway.routes[0].id=order-service
# spring.cloud.gateway.routes[0].uri=lb://order-service
# spring.cloud.gateway.routes[0].predicates[0]=Path=/api/v1/orders/**
```

---

## 9. Feign Client (Inter-Service REST Calls)

```properties
feign.client.config.default.connectTimeout=5000
feign.client.config.default.readTimeout=5000
feign.circuitbreaker.enabled=true
```

---

## 10. Resilience4j (Circuit Breaker / Retry)

```properties
resilience4j.circuitbreaker.instances.orderService.registerHealthIndicator=true
resilience4j.circuitbreaker.instances.orderService.slidingWindowSize=10
resilience4j.circuitbreaker.instances.orderService.failureRateThreshold=50
resilience4j.circuitbreaker.instances.orderService.waitDurationInOpenState=10000

resilience4j.retry.instances.orderService.maxAttempts=3
resilience4j.retry.instances.orderService.waitDuration=2000
```

---

## 11. Security (JWT / OAuth2)

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=${JWT_ISSUER_URI:http://localhost:8080/auth/realms/microservices}
app.jwt.secret=${JWT_SECRET:change-this-secret-key}
app.jwt.expiration-ms=3600000
```

---

## 12. Actuator (Health, Metrics, Monitoring)

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus,circuitbreakers
management.endpoint.health.show-details=always
management.health.circuitbreakers.enabled=true
management.metrics.tags.application=${spring.application.name}
```

---

## 13. Distributed Tracing (Zipkin / Micrometer)

```properties
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=${ZIPKIN_URI:http://localhost:9411/api/v2/spans}
```

---

## 14. Swagger / OpenAPI Docs

```properties
springdoc.api-docs.path=/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.operationsSorter=method
```

---

## 15. Docker-Specific Overrides

When running under `docker-compose`, service names replace `localhost`:

```properties
DB_HOST=mysql-db
KAFKA_BROKER=kafka:9092
REDIS_HOST=redis
EUREKA_URI=http://eureka-server:8761/eureka/
CONFIG_SERVER_URI=http://config-server:8888
ZIPKIN_URI=http://zipkin:9411/api/v2/spans
```

Apply these either via a `docker-compose.yml` `environment:` block, or an `application-docker.properties` profile activated with `spring.profiles.active=docker`.

---

## 16. Logging

```properties
logging.level.root=INFO
logging.level.org.springframework.web=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
logging.file.name=logs/order-service.log
```

---

## Companion `docker-compose.yml`

Spins up MySQL, Kafka + Zookeeper, Redis, Eureka Server, Zipkin, and the microservice itself.

```yaml
version: "3.8"

services:

  mysql-db:
    image: mysql:8.0
    container_name: mysql-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: orderdb
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - microservice-net

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    networks:
      - microservice-net

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    networks:
      - microservice-net

  redis:
    image: redis:7.2-alpine
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - microservice-net

  eureka-server:
    image: steeltoeoss/eureka-server:latest   # replace with your own custom Eureka server image
    container_name: eureka-server
    ports:
      - "8761:8761"
    networks:
      - microservice-net

  zipkin:
    image: openzipkin/zipkin:latest
    container_name: zipkin
    ports:
      - "9411:9411"
    networks:
      - microservice-net

  order-service:
    build: .
    container_name: order-service
    depends_on:
      - mysql-db
      - kafka
      - redis
      - eureka-server
      - zipkin
    environment:
      SPRING_PROFILES_ACTIVE: docker
      DB_HOST: mysql-db
      DB_PORT: 3306
      DB_NAME: orderdb
      DB_USERNAME: root
      DB_PASSWORD: root
      KAFKA_BROKER: kafka:9092
      REDIS_HOST: redis
      EUREKA_URI: http://eureka-server:8761/eureka/
      ZIPKIN_URI: http://zipkin:9411/api/v2/spans
    ports:
      - "8081:8081"
    networks:
      - microservice-net

networks:
  microservice-net:
    driver: bridge

volumes:
  mysql-data:
```

---

## Quick Reference Table

| Integration | Purpose | Key Prefix |
|---|---|---|
| MySQL/Postgres | Primary datastore | `spring.datasource.*` |
| HikariCP | Connection pooling | `spring.datasource.hikari.*` |
| Flyway | DB schema migrations | `spring.flyway.*` |
| Redis | Caching / sessions | `spring.redis.*` |
| Kafka | Async messaging / events | `spring.kafka.*` |
| RabbitMQ | Alternative message broker | `spring.rabbitmq.*` |
| Eureka | Service discovery | `eureka.*` |
| Spring Cloud Config | Centralized config | `spring.cloud.config.*` |
| Feign | Inter-service REST calls | `feign.*` |
| Resilience4j | Circuit breaker / retry | `resilience4j.*` |
| OAuth2/JWT | Security | `spring.security.oauth2.*` |
| Actuator | Health & metrics | `management.*` |
| Zipkin | Distributed tracing | `management.zipkin.*` / `management.tracing.*` |
| Swagger/OpenAPI | API docs | `springdoc.*` |
| Docker Compose | Local orchestration | `docker-compose.yml` |



# Spring Boot Project Structure Guide
### REST API (Monolith) vs Microservices

This guide covers two proven approaches to structuring a Spring Boot project:
1. **Single REST API** — layered structure (entity, dto, mapper, repository, service, controller)
2. **Microservices** — the same layered structure applied per-service, plus inter-service communication

---

# Part 1: Single REST API (Layered / Package-by-Layer)

Organized by technical role. Best for a single deployable application with one shared database.

## Project Structure

```
rest-api-project/
├── src/
│   ├── main/
│   │   ├── java/com/company/restapi/
│   │   │   ├── RestApiApplication.java              # main class
│   │   │   │
│   │   │   ├── controller/                           # REST endpoints
│   │   │   │   ├── OrderController.java
│   │   │   │   ├── CustomerController.java
│   │   │   │   └── ProductController.java
│   │   │   │
│   │   │   ├── service/                               # business logic interfaces
│   │   │   │   ├── OrderService.java
│   │   │   │   ├── CustomerService.java
│   │   │   │   └── ProductService.java
│   │   │   │
│   │   │   ├── service/impl/                          # business logic implementations
│   │   │   │   ├── OrderServiceImpl.java
│   │   │   │   ├── CustomerServiceImpl.java
│   │   │   │   └── ProductServiceImpl.java
│   │   │   │
│   │   │   ├── repository/                            # data access layer (Spring Data JPA)
│   │   │   │   ├── OrderRepository.java
│   │   │   │   ├── CustomerRepository.java
│   │   │   │   └── ProductRepository.java
│   │   │   │
│   │   │   ├── entity/                                 # JPA entities (DB tables)
│   │   │   │   ├── Order.java
│   │   │   │   ├── Customer.java
│   │   │   │   ├── Product.java
│   │   │   │   └── BaseEntity.java                     # common fields (id, createdAt, updatedAt)
│   │   │   │
│   │   │   ├── dto/                                     # request/response objects
│   │   │   │   ├── request/
│   │   │   │   │   ├── OrderRequest.java
│   │   │   │   │   ├── CustomerRequest.java
│   │   │   │   │   └── ProductRequest.java
│   │   │   │   └── response/
│   │   │   │       ├── OrderResponse.java
│   │   │   │       ├── CustomerResponse.java
│   │   │   │       └── ProductResponse.java
│   │   │   │
│   │   │   ├── mapper/                                  # entity <-> DTO conversion
│   │   │   │   ├── OrderMapper.java
│   │   │   │   ├── CustomerMapper.java
│   │   │   │   └── ProductMapper.java
│   │   │   │
│   │   │   ├── exception/                               # centralized error handling
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   ├── ResourceNotFoundException.java
│   │   │   │   ├── BadRequestException.java
│   │   │   │   └── ErrorResponse.java
│   │   │   │
│   │   │   ├── config/                                  # configuration classes
│   │   │   │   ├── SwaggerConfig.java
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   ├── WebConfig.java
│   │   │   │   └── ModelMapperConfig.java
│   │   │   │
│   │   │   ├── validation/                              # custom validators (optional)
│   │   │   │   ├── ValidEmail.java
│   │   │   │   └── EmailValidator.java
│   │   │   │
│   │   │   ├── util/                                    # helper/utility classes
│   │   │   │   ├── DateUtil.java
│   │   │   │   └── ResponseUtil.java
│   │   │   │
│   │   │   ├── constant/                                # constants/enums
│   │   │   │   ├── AppConstants.java
│   │   │   │   └── OrderStatus.java
│   │   │   │
│   │   │   └── security/                                # JWT/auth (if applicable)
│   │   │       ├── JwtFilter.java
│   │   │       ├── JwtUtil.java
│   │   │       └── UserDetailsServiceImpl.java
│   │   │
│   │   └── resources/
│   │       ├── application.properties
│   │       ├── application-dev.properties
│   │       ├── application-prod.properties
│   │       ├── db/migration/                            # Flyway scripts
│   │       │   ├── V1__init_schema.sql
│   │       │   └── V2__seed_data.sql
│   │       ├── static/
│   │       └── templates/
│   │
│   └── test/
│       └── java/com/company/restapi/
│           ├── controller/
│           │   └── OrderControllerTest.java
│           ├── service/
│           │   └── OrderServiceTest.java
│           └── repository/
│               └── OrderRepositoryTest.java
│
├── Dockerfile
├── docker-compose.yml
├── pom.xml (or build.gradle)
├── .gitignore
└── README.md
```

## Layer-by-Layer Breakdown

| Layer | Responsibility | Example |
|---|---|---|
| **Entity** | Maps directly to a DB table via JPA annotations (`@Entity`, `@Table`, `@Id`) | `Order.java` with `id`, `customerId`, `totalAmount`, `status` |
| **Repository** | Extends `JpaRepository<Entity, ID>` — handles all DB queries | `interface OrderRepository extends JpaRepository<Order, Long>` |
| **DTO (request/response)** | What actually crosses the wire — never expose entities directly | `OrderRequest` (input), `OrderResponse` (output) |
| **Mapper** | Converts Entity ↔ DTO | `OrderMapper.toEntity()`, `OrderMapper.toResponse()` |
| **Service (interface)** | Defines the business contract | `OrderService.createOrder(OrderRequest)` |
| **Service Impl** | Actual business logic, transactions, calls repository + mapper | `OrderServiceImpl implements OrderService` |
| **Controller** | HTTP layer only — routing, status codes, calls service | `@PostMapping("/orders")` |
| **Exception** | `@ControllerAdvice` catches exceptions globally, returns consistent error JSON | `ResourceNotFoundException → 404` |

## Sample Flow — Order Creation

```
Client → OrderController → OrderService (interface) → OrderServiceImpl
                                                              ↓
                                        OrderMapper.toEntity(OrderRequest)
                                                              ↓
                                              OrderRepository.save(entity)
                                                              ↓
                                        OrderMapper.toResponse(savedEntity)
                                                              ↓
                                            ← OrderResponse returned to client
```

## Practical Notes

- **Interface + Impl for services**: some teams skip the interface and use a concrete `OrderService` class directly — less boilerplate, fine unless you need multiple implementations or heavy mocking in tests.
- **Never return entities directly from controllers** — always map to a DTO. Avoids leaking DB structure, lazy-loading exceptions, and lets you version your API independently of your schema.
- **MapStruct** is worth using for the mapper layer instead of hand-writing conversions — generates mapping code at compile time, no runtime reflection overhead (unlike ModelMapper).
- **`BaseEntity`** with `@MappedSuperclass` holding `id`, `createdAt`, `updatedAt` (via `@CreatedDate`/`@LastModifiedDate` + `@EnableJpaAuditing`) saves repeating those fields in every entity.

---

# Part 2: Microservices Architecture

Same layered structure applied **inside each individual service**, plus inter-service communication layers.

## Top-Level Repo Layout

```
ecommerce-platform/
├── order-service/            # full layered structure goes INSIDE each service
├── payment-service/
├── inventory-service/
├── customer-service/
├── api-gateway/               # Spring Cloud Gateway
├── eureka-server/             # service discovery
├── config-server/             # centralized config
├── docker-compose.yml          # orchestrates all services together
└── README.md
```

## Inside ONE Microservice (e.g. `order-service/`)

```
order-service/
├── src/
│   ├── main/
│   │   ├── java/com/company/orderservice/
│   │   │   ├── OrderServiceApplication.java
│   │   │   │
│   │   │   ├── controller/
│   │   │   │   └── OrderController.java
│   │   │   │
│   │   │   ├── service/
│   │   │   │   └── OrderService.java
│   │   │   │
│   │   │   ├── service/impl/
│   │   │   │   └── OrderServiceImpl.java
│   │   │   │
│   │   │   ├── repository/
│   │   │   │   └── OrderRepository.java
│   │   │   │
│   │   │   ├── entity/
│   │   │   │   ├── Order.java
│   │   │   │   └── BaseEntity.java
│   │   │   │
│   │   │   ├── dto/
│   │   │   │   ├── request/
│   │   │   │   │   └── OrderRequest.java
│   │   │   │   └── response/
│   │   │   │       ├── OrderResponse.java
│   │   │   │       └── InventoryResponse.java        # DTO for data coming FROM another service
│   │   │   │
│   │   │   ├── mapper/
│   │   │   │   └── OrderMapper.java
│   │   │   │
│   │   │   ├── exception/
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   ├── ResourceNotFoundException.java
│   │   │   │   └── ErrorResponse.java
│   │   │   │
│   │   │   ├── config/
│   │   │   │   ├── SwaggerConfig.java
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   ├── KafkaConfig.java
│   │   │   │   └── FeignConfig.java
│   │   │   │
│   │   │   ├── client/                                 # ⭐ calls to OTHER microservices
│   │   │   │   ├── InventoryClient.java                # Feign interface -> inventory-service
│   │   │   │   ├── PaymentClient.java                  # Feign interface -> payment-service
│   │   │   │   └── fallback/
│   │   │   │       └── InventoryClientFallback.java    # circuit breaker fallback
│   │   │   │
│   │   │   ├── kafka/                                   # ⭐ async events between services
│   │   │   │   ├── producer/
│   │   │   │   │   └── OrderEventProducer.java          # publishes "OrderCreatedEvent"
│   │   │   │   ├── consumer/
│   │   │   │   │   └── PaymentEventConsumer.java        # listens for "PaymentCompletedEvent"
│   │   │   │   └── event/
│   │   │   │       ├── OrderCreatedEvent.java
│   │   │   │       └── PaymentCompletedEvent.java
│   │   │   │
│   │   │   ├── constant/
│   │   │   │   └── OrderStatus.java
│   │   │   │
│   │   │   └── util/
│   │   │       └── DateUtil.java
│   │   │
│   │   └── resources/
│   │       ├── application.properties        # has eureka, kafka, feign, config-server settings
│   │       ├── application-docker.properties
│   │       └── db/migration/
│   │           └── V1__init_order_schema.sql
│   │
│   └── test/
│       └── java/com/company/orderservice/
│           ├── controller/OrderControllerTest.java
│           ├── service/OrderServiceTest.java
│           └── integration/OrderIntegrationTest.java
│
├── Dockerfile
├── pom.xml
└── README.md
```

## REST API vs Microservice — What Changes

| Layer | Single REST API | Microservice |
|---|---|---|
| `entity/`, `dto/`, `mapper/`, `repository/`, `service/`, `controller/` | ✅ same | ✅ same — unchanged |
| **Scope of entities** | All domain entities (`Order`, `Customer`, `Product`) | Only `Order` — this service owns nothing else |
| **`client/`** | Not needed | Feign clients to call `inventory-service`, `payment-service`, etc. over HTTP |
| **`kafka/`** | Optional | Producers/consumers to publish and react to events across services |
| **`dto/response/`** | Only your own response DTOs | Also includes DTOs that mirror another service's response (e.g. `InventoryResponse`) since you can't share entities across services |
| **Database** | One shared DB | `order-service` has its own DB — never touches `payment-service`'s tables directly |
| **`application.properties`** | DB + basic config | Adds `eureka.client.*`, `spring.kafka.*`, `feign.*`, `spring.cloud.config.*` |

## Example Flow Across Services (Order → Inventory → Payment)

```
Client → OrderController → OrderService → OrderServiceImpl
                                                 │
                          ┌──────────────────────┼───────────────────────┐
                          ▼                      ▼                       ▼
                 OrderRepository        InventoryClient (Feign)   OrderEventProducer (Kafka)
                 (save to own DB)        → calls inventory-service   → publishes OrderCreatedEvent
                                                                              │
                                                                              ▼
                                                            payment-service consumes event,
                                                            processes payment, publishes
                                                            PaymentCompletedEvent
                                                                              │
                                                                              ▼
                                              order-service's PaymentEventConsumer
                                              updates Order status → COMPLETED
```

## The Core Rule That Changes Everything

In a monolith, `OrderServiceImpl` can just call `CustomerRepository` directly since it's all one app. In microservices, **`order-service` cannot import `Customer` entity or `CustomerRepository` at all** — that code doesn't even exist in this codebase. It has to either:

1. Call `customer-service` synchronously via `CustomerClient` (Feign), or
2. React to events asynchronously via Kafka (e.g. keep a local read-only copy of customer data updated via `CustomerUpdatedEvent`)

This is the core discipline of microservices: **no shared database, no shared entities** — only network calls or events cross service boundaries.

---

# Quick Decision Guide

| If you're building... | Use |
|---|---|
| One app, one team, one database, moderate scale | **Part 1 — REST API layered structure** |
| Multiple independent teams/domains, need independent scaling & deployment | **Part 2 — Microservices**, with Part 1's structure repeated per service |
| Not sure yet | Start with Part 1. It's easier to split a well-organized monolith into services later than to prematurely manage a distributed system. |

---

# Summary Table — Layer Purpose (Applies to Both)

| Layer | Purpose |
|---|---|
| `entity` | JPA-mapped DB tables |
| `repository` | Data access (Spring Data JPA) |
| `dto` | Request/response contracts exposed over the API |
| `mapper` | Entity ↔ DTO conversion |
| `service` / `service.impl` | Business logic, transactions |
| `controller` | HTTP routing, request/response mapping |
| `exception` | Centralized error handling (`@ControllerAdvice`) |
| `config` | `@Configuration` classes (security, Swagger, Kafka, etc.) |
| `client` *(microservices only)* | Feign clients for calling other services |
| `kafka` *(microservices only)* | Event producers/consumers for async communication |


# Spring Boot `pom.xml` — Detailed Guide

Covers a typical Maven `pom.xml` for a Spring Boot REST API / microservice project, section by section.

---

## Full Sample `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
        <relativePath/>
    </parent>

    <groupId>com.company</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0</version>
    <name>order-service</name>
    <description>Order microservice for e-commerce platform</description>
    <packaging>jar</packaging>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.2</spring-cloud.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Core Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Microservices: Eureka + Feign + Config -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>

        <!-- Resilience4j -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
        </dependency>

        <!-- Actuator (health/metrics) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- MapStruct -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>1.5.5.Final</version>
        </dependency>

        <!-- Swagger / OpenAPI -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.5.0</version>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

---

## Section-by-Section Explanation

### 1. `<parent>` — Spring Boot Parent POM

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>
```

Inherits sensible defaults: Java compiler settings, UTF-8 encoding, dependency version management (so you don't specify versions for most Spring dependencies), and plugin configuration. This is the biggest reason Spring Boot POMs stay short — versions are managed centrally.

### 2. Project Coordinates (GAV)

```xml
<groupId>com.company</groupId>
<artifactId>order-service</artifactId>
<version>1.0.0</version>
```

- **groupId** — your organization/company namespace (reverse domain convention)
- **artifactId** — the project/module name, becomes the JAR filename
- **version** — your app's version (`1.0.0`, or `1.0.0-SNAPSHOT` during active development)

### 3. `<packaging>`

```xml
<packaging>jar</packaging>
```

`jar` for a standalone Spring Boot app (embedded server) — the modern default. `war` only if deploying to an external servlet container like a standalone Tomcat.

### 4. `<properties>`

```xml
<java.version>17</java.version>
<spring-cloud.version>2023.0.2</spring-cloud.version>
```

Custom variables reused throughout the file. `java.version` tells the parent POM which JDK to compile against. `spring-cloud.version` must be compatible with your Spring Boot version — mismatches are a common source of dependency conflicts in microservices.

### 5. `<dependencyManagement>` — Spring Cloud BOM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

This imports Spring Cloud's Bill of Materials (BOM) — it centrally manages compatible versions for Eureka, Feign, Config Server, Gateway, etc. Only needed if you're building microservices; skip this for a plain REST API.

### 6. `<dependencies>` — Grouped by Purpose

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-web` | REST controllers, embedded Tomcat, Jackson JSON |
| `spring-boot-starter-data-jpa` | Hibernate + Spring Data repositories |
| `mysql-connector-j` | JDBC driver (scope `runtime` — only needed at runtime, not compile time) |
| `spring-boot-starter-validation` | `@Valid`, `@NotNull`, `@Email` etc. on DTOs |
| `spring-kafka` | Kafka producer/consumer support |
| `spring-cloud-starter-netflix-eureka-client` | Service registration/discovery |
| `spring-cloud-starter-openfeign` | Declarative REST clients for calling other services |
| `spring-cloud-starter-config` | Pull config from a centralized Config Server |
| `spring-cloud-starter-circuitbreaker-resilience4j` | Circuit breaker/retry patterns |
| `spring-boot-starter-actuator` | `/health`, `/metrics` endpoints |
| `spring-boot-starter-oauth2-resource-server` | JWT validation |
| `lombok` | Generates getters/setters/constructors via annotations (`optional=true` since it's compile-time only) |
| `mapstruct` | Compile-time entity↔DTO mapping |
| `springdoc-openapi-starter-webmvc-ui` | Swagger UI at `/swagger-ui.html` |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-kafka-test` | Embedded Kafka for integration tests |

**Note**: most dependencies have **no `<version>` tag** — that's the parent POM's dependency management doing its job. Only dependencies *not* managed by Spring Boot's BOM (like MapStruct, Springdoc) need an explicit version.

### 7. `<scope>` Values Commonly Used

| Scope | Meaning |
|---|---|
| *(default)* `compile` | Needed at compile time AND runtime, packaged into the JAR |
| `runtime` | Only needed when running, not for compiling your code (e.g. JDBC drivers) |
| `test` | Only available during test compilation/execution, not packaged |
| `optional=true` | Not passed transitively to projects that depend on this one (used for Lombok) |

### 8. `<build><plugins>` — Spring Boot Maven Plugin

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

This is what makes `mvn package` produce an **executable "fat" JAR** — bundling all dependencies plus an embedded Tomcat into a single runnable `.jar`. Without it, you'd get a plain JAR that can't run standalone with `java -jar`. The `<excludes>` block here keeps Lombok out of the final JAR since it's compile-time-only tooling.

---

## REST API vs Microservice — `pom.xml` Differences

| Section | Plain REST API | Microservice |
|---|---|---|
| `spring-cloud-dependencies` BOM | Not needed | Required |
| `spring-cloud-starter-netflix-eureka-client` | Skip | Required |
| `spring-cloud-starter-openfeign` | Skip | Required |
| `spring-cloud-starter-config` | Skip | Add if using centralized config |
| `spring-cloud-starter-circuitbreaker-resilience4j` | Optional | Common |
| Everything else (web, JPA, validation, actuator, security) | Same | Same |

---

## Quick Command Reference

| Command | Purpose |
|---|---|
| `mvn clean install` | Clean, compile, test, and install to local repo |
| `mvn package` | Build the executable JAR (via spring-boot-maven-plugin) |
| `mvn spring-boot:run` | Run the app directly without building a JAR first |
| `java -jar target/order-service-1.0.0.jar` | Run the built JAR |
| `mvn dependency:tree` | View full dependency tree (useful for resolving version conflicts) |