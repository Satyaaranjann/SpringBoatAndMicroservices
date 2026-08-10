# Microservices with Spring Boot — Q&A

---

**Q1. Monolith vs Microservices — what's the difference?**

| | Monolith | Microservices |
|---|---|---|
| Deployment | Single deployable unit | Independently deployable services |
| Codebase | One shared codebase | Multiple separate codebases/repos |
| Scaling | Scale the entire app together | Scale individual services independently |
| Technology | Usually one tech stack | Can mix languages/stacks per service |
| Data | Typically one shared database | Each service owns its own database (Database per Service) |
| Communication | In-process method calls | Network calls (REST, messaging) |
| Failure isolation | A bug can bring down the whole app | A failing service can be isolated (with proper resilience) |
| Complexity | Simpler to develop/test/deploy initially | Higher operational complexity (distributed systems problems) |

**Why teams move to microservices:** independent scaling, independent deployment (faster release cycles per team), technology flexibility, and fault isolation. **Trade-offs interviewers expect you to mention:** network latency, distributed transaction complexity, harder debugging (a request spans multiple services), and significant operational overhead (service discovery, config management, monitoring) that a monolith doesn't need.

**Important nuance:** microservices aren't automatically "better" — they solve organizational/scaling problems at the cost of distributed-systems complexity. Many successful systems start as a well-structured monolith and split out services only when a specific scaling or team-boundary need arises ("monolith first" approach).

---

**Q2. What is Service Communication, and how do REST and Feign fit in?**

Microservices need to talk to each other over the network since they no longer share a process. The two broad styles are **synchronous** (REST/gRPC — caller waits for a response) and **asynchronous** (messaging — caller doesn't wait).

**REST-based synchronous communication** is the most common starting point — one service calls another's HTTP API directly.

**Feign** is a **declarative REST client** — instead of manually building HTTP requests, you define an interface and Spring generates the implementation at runtime.

```java
@FeignClient(name = "inventory-service", url = "${inventory.service.url}")
public interface InventoryClient {

    @GetMapping("/api/inventory/{productId}")
    InventoryResponse checkStock(@PathVariable("productId") Long productId);
}

@Service
public class OrderService {
    @Autowired
    private InventoryClient inventoryClient;

    public void placeOrder(Order order) {
        InventoryResponse stock = inventoryClient.checkStock(order.getProductId());
        // ...
    }
}
```

**Why Feign over manually using `RestTemplate`/`WebClient`:** far less boilerplate — no manual URL building, serialization, or error handling per call; it reads like a normal method call while Spring Cloud OpenFeign handles the HTTP mechanics, and it integrates directly with service discovery (calling `inventory-service` by name instead of a hardcoded URL) and load balancing.

---

**Q3. `RestTemplate` vs `WebClient` — which should you use today?**

| | `RestTemplate` | `WebClient` |
|---|---|---|
| Status | **Maintenance mode** (not actively enhanced) | Actively developed, recommended going forward |
| Model | **Blocking**, synchronous only | **Non-blocking**, supports both reactive and synchronous usage |
| Module | Spring Web (MVC) | Spring WebFlux |
| Thread usage | One thread blocked per in-flight call | Efficient, doesn't block a thread waiting for I/O |
| API style | Traditional method calls | Reactive, fluent, chainable (`Mono`/`Flux`) |

```java
// RestTemplate (blocking, legacy)
ResponseEntity<User> response = restTemplate.getForEntity("/users/1", User.class);

// WebClient (non-blocking, modern)
Mono<User> userMono = webClient.get()
    .uri("/users/1")
    .retrieve()
    .bodyToMono(User.class);

User user = userMono.block();  // can block if used in a traditional (non-reactive) app
```

**Interview-level nuance:** you don't need a fully reactive application (WebFlux end-to-end) to benefit from `WebClient` — it can be used in a traditional Spring MVC app too, and Spring officially recommends it as `RestTemplate`'s replacement even outside reactive contexts, since `RestTemplate` receives no new features.

---

**Q4. What is Service Discovery, and how does Eureka work?**

In a dynamic microservices environment, service instances scale up/down and their network locations (IP/port) change constantly — hardcoding URLs doesn't work. **Service Discovery** solves this: each service instance **registers itself** with a central registry, and other services **look it up by name** instead of a fixed address.

**Eureka** (Netflix, part of Spring Cloud Netflix) is the classic implementation:

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApplication { ... }
```

```yaml
# Eureka Client (any microservice)
spring:
  application:
    name: order-service
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

**How it works:**
1. Each service instance registers itself with Eureka on startup (name + IP/port)
2. It sends periodic **heartbeats** to prove it's still alive
3. If heartbeats stop, Eureka removes the instance from the registry after a timeout
4. Other services query Eureka (or use a Feign client / load-balanced `WebClient`) to resolve `order-service` to an actual, currently-live instance address — often with client-side load balancing across multiple instances

**Modern alternative:** in Kubernetes-based deployments, built-in DNS-based service discovery often replaces Eureka entirely, since Kubernetes already tracks and load-balances pod instances natively.

---

**Q5. What is Spring Cloud Config?**

A **centralized, externalized configuration server** for distributed systems. Instead of each microservice bundling its own `application.yml`, all services fetch their configuration from a shared Config Server (typically backed by a Git repository) at startup.

```yaml
# Config Server
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
```

```yaml
# Client microservice
spring:
  application:
    name: order-service
  config:
    import: "configserver:http://localhost:8888"
```

**Benefits:**
- Configuration changes don't require rebuilding/redeploying services
- Centralized visibility and version history (via Git) of every environment's config
- Supports **dynamic refresh** — combined with `@RefreshScope`, beans can pick up config changes at runtime without a restart, triggered via an Actuator `/actuator/refresh` endpoint (or automatically via Spring Cloud Bus across all instances)

---

**Q6. What is an API Gateway, and how does Spring Cloud Gateway work?**

An **API Gateway** is a single entry point that sits in front of all microservices, routing external requests to the appropriate backend service, and centralizing cross-cutting concerns (authentication, rate limiting, logging, request/response transformation) so individual services don't each reimplement them.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
```

**Why it matters in a microservices architecture:**
- Clients don't need to know internal service addresses or topology
- Centralizes auth (validate JWT once at the gateway, not in every service)
- Enables cross-cutting policies (rate limiting, circuit breaking, request logging) in one place
- `lb://order-service` shows the gateway integrating directly with service discovery + load balancing

---

**Q7. What is Load Balancing, and how is it typically done in a Spring Cloud microservices setup?**

Load balancing distributes incoming requests across **multiple instances** of a service, preventing any single instance from being overwhelmed and enabling horizontal scaling.

**Client-side load balancing** (common in Spring Cloud, via **Spring Cloud LoadBalancer**, which replaced the older Netflix Ribbon):
```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

// calling by logical service name — the load balancer resolves it to a specific instance
webClient.get().uri("http://order-service/api/orders/1")...
```

The calling service (via its embedded load balancer, integrated with Eureka/service discovery) picks which specific instance to call — commonly using a **round-robin** strategy by default, though other strategies (weighted, least-connections) are configurable.

**Server-side load balancing** — an external load balancer (Nginx, a cloud load balancer, or Kubernetes' Service abstraction) sits in front of instances and distributes traffic; the calling service doesn't need any load-balancing logic itself, just calls a single stable address.

---

**Q8. What is a Circuit Breaker, and how does Resilience4j implement it?**

A **Circuit Breaker** prevents a failing downstream service from cascading failures across the whole system. It monitors calls to a dependency and, after a threshold of failures, "opens" the circuit — failing fast (without even attempting the call) for a cooldown period, instead of letting every caller keep hitting a service that's already struggling (which would make recovery harder and waste resources on doomed calls).

**Resilience4j** is the modern standard library (replacing Netflix Hystrix, which is in maintenance mode):

```java
@Service
public class InventoryService {

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackCheckStock")
    public InventoryResponse checkStock(Long productId) {
        return inventoryClient.checkStock(productId);
    }

    public InventoryResponse fallbackCheckStock(Long productId, Throwable t) {
        return new InventoryResponse(productId, false, "Inventory service unavailable");
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
```

**Three states:**
- **Closed** — normal operation, calls pass through, failures are counted
- **Open** — failure threshold exceeded; calls fail immediately via the fallback, without attempting the real call
- **Half-Open** — after the wait duration, a limited number of trial calls are allowed through to test if the downstream service has recovered; success → closes the circuit again, failure → reopens it

---

**Q9. What are Retry and Rate Limiting, and how do they complement Circuit Breakers?**

**Retry** — automatically re-attempts a failed call a configured number of times before giving up, useful for **transient** failures (a momentary network blip) that are likely to succeed on a second try.

```java
@Retry(name = "inventoryService", fallbackMethod = "fallbackCheckStock")
@CircuitBreaker(name = "inventoryService")
public InventoryResponse checkStock(Long productId) {
    return inventoryClient.checkStock(productId);
}
```
```yaml
resilience4j:
  retry:
    instances:
      inventoryService:
        max-attempts: 3
        wait-duration: 500ms
```

**Rate Limiting** — restricts how many calls are allowed within a time window, protecting a service (or a downstream dependency) from being overwhelmed, whether by legitimate high traffic or abuse.

```java
@RateLimiter(name = "inventoryService")
public InventoryResponse checkStock(Long productId) { ... }
```
```yaml
resilience4j:
  ratelimiter:
    instances:
      inventoryService:
        limit-for-period: 10
        limit-refresh-period: 1s
        timeout-duration: 0
```

**How they work together (order matters):** typically **Retry** wraps the innermost call (retry a few times on transient failure), **Circuit Breaker** wraps around that (stop retrying entirely once the failure rate crosses a threshold, avoiding retry storms against an already-struggling service), and **Rate Limiter** can apply independently to cap outbound call volume regardless of failures. Combining retry blindly *without* a circuit breaker is a common anti-pattern — retrying against a genuinely down service just multiplies load on it.

---

**Q10. What is Distributed Tracing, and how do Sleuth and Zipkin fit in?**

In a microservices architecture, a single user request often flows through **many services** — distributed tracing lets you follow that request's full path, timing, and any failures across all of them, which is essential for debugging latency issues or failures that span service boundaries (something logs from a single service can't show you alone).

- **Spring Cloud Sleuth** (now largely superseded by **Micrometer Tracing** in newer Spring Boot versions) — automatically instruments requests with a **Trace ID** (shared across the entire request's journey through all services) and a **Span ID** (unique per individual operation/hop), and propagates these IDs through HTTP headers as a request moves between services.
- **Zipkin** — a tracing backend/UI that **collects and visualizes** the trace data emitted by Sleuth/Micrometer Tracing, letting you see a timeline/waterfall view of exactly how long each service spent handling its part of a request.

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # trace 100% of requests (lower in production for high-traffic services)
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

With this configured, log lines automatically include `[order-service,traceId,spanId]`, making it possible to correlate logs across services for the same logical request — and Zipkin's UI shows exactly which service in the chain was the slow one.

---

**Q11. What is Event-Driven Architecture?**

An architectural style where services communicate by **publishing and consuming events** (asynchronously via a message broker) rather than calling each other directly and waiting for a response. A service publishes an event ("OrderPlaced") without knowing or caring which services consume it; interested services subscribe and react independently.

**Why use it over direct REST calls:**
- **Decoupling** — the publisher doesn't need to know which/how many consumers exist, or wait for them
- **Resilience** — if a consumer is temporarily down, messages queue up and are processed once it recovers, instead of the request failing outright
- **Scalability** — consumers can process events at their own pace, and multiple consumer instances can share the load

**Trade-off:** harder to reason about (eventual consistency instead of immediate consistency), harder to debug (no simple request/response trace), and requires careful handling of message ordering, duplicate delivery, and failure/retry semantics.

---

**Q12. How does Kafka Integration work in Spring Boot?**

**Apache Kafka** is a distributed, high-throughput event streaming platform — messages are published to **topics**, retained for a configurable period (not deleted on consumption, unlike traditional queues), and can be replayed by multiple independent consumer groups.

```java
// Producer
@Service
public class OrderEventProducer {
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderPlaced(OrderEvent event) {
        kafkaTemplate.send("order-events", event.getOrderId().toString(), event);
    }
}

// Consumer
@Service
public class InventoryEventConsumer {
    @KafkaListener(topics = "order-events", groupId = "inventory-group")
    public void handleOrderPlaced(OrderEvent event) {
        inventoryService.reserveStock(event.getProductId(), event.getQuantity());
    }
}
```

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: inventory-group
      auto-offset-reset: earliest
```

**Key Kafka concepts:** **Topic** (a named stream of events), **Partition** (a topic is split into partitions for parallelism/scaling), **Consumer Group** (multiple consumer instances sharing a group ID split the partitions between them, so each message is processed by exactly one consumer within the group), **Offset** (each consumer tracks its position in a partition, enabling replay).

**Best fit:** high-throughput event streaming, event sourcing, log aggregation, and scenarios needing message replay/history.

---

**Q13. How does RabbitMQ Integration work in Spring Boot?**

**RabbitMQ** is a traditional **message broker** implementing the AMQP protocol — messages are routed via **exchanges** to **queues**, and once consumed and acknowledged, they're typically removed (unlike Kafka's retention model).

```java
// Producer
@Service
public class OrderEventProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void publishOrderPlaced(OrderEvent event) {
        rabbitTemplate.convertAndSend("order-exchange", "order.placed", event);
    }
}

// Consumer
@Component
public class InventoryEventConsumer {
    @RabbitListener(queues = "inventory-queue")
    public void handleOrderPlaced(OrderEvent event) {
        inventoryService.reserveStock(event.getProductId(), event.getQuantity());
    }
}
```

```java
@Configuration
public class RabbitMQConfig {
    @Bean
    public Queue inventoryQueue() { return new Queue("inventory-queue"); }

    @Bean
    public TopicExchange orderExchange() { return new TopicExchange("order-exchange"); }

    @Bean
    public Binding binding(Queue inventoryQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(inventoryQueue).to(orderExchange).with("order.*");
    }
}
```

**Kafka vs RabbitMQ — the core interview distinction:**
- **Kafka** — built for high-throughput event **streaming**, retains messages for replay, partition-based parallelism; favored for event sourcing, analytics pipelines, and very high message volume
- **RabbitMQ** — built for flexible **message routing** (complex exchange/binding patterns), simpler point-to-point/pub-sub messaging, strong support for priority queues and per-message routing logic; favored for traditional task queues and moderate-throughput messaging with rich routing needs

---

**Q14. What is the Saga Pattern?**

A pattern for managing **distributed transactions** across multiple microservices, where each service owns its own database (so a single ACID transaction spanning all of them isn't possible). A Saga breaks a business transaction into a sequence of **local transactions**, each in a single service, with **compensating transactions** defined to undo prior steps if a later step fails.

**Two coordination styles:**
- **Choreography** — each service publishes an event after completing its local transaction, and the next service reacts to that event; no central coordinator, fully decentralized, but harder to track the overall flow as the number of steps grows
- **Orchestration** — a central orchestrator service explicitly tells each participant what to do and in what order, and handles compensating actions on failure; easier to understand/monitor the overall flow, but introduces a central coordinating component

**Example (order placement, choreography style):**
1. `OrderService` creates an order (pending) → publishes `OrderCreated`
2. `PaymentService` charges the customer → publishes `PaymentCompleted` (or `PaymentFailed`)
3. `InventoryService` reserves stock → publishes `StockReserved` (or `StockUnavailable`)
4. If any step fails, a **compensating event** triggers prior steps to undo their work (e.g., `PaymentFailed` → `OrderService` cancels the order and triggers a refund if payment had already partially succeeded)

**Why not just use a distributed transaction (2PC)?** Two-Phase Commit requires all participating services/databases to be locked and coordinated synchronously, which doesn't scale well and conflicts with the database-per-service principle in microservices — Sagas trade strict ACID consistency for **eventual consistency**, which is the accepted norm in distributed microservice systems.

---

**Q15. What is the Outbox Pattern?**

Solves a subtle but important **dual-write problem**: when a service needs to both (1) update its own database **and** (2) publish an event about that change, doing these as two separate operations risks inconsistency — e.g., the database update succeeds but the message broker publish fails (or the service crashes in between), leaving the database and other services out of sync.

**How it works:**
1. Instead of publishing directly to the message broker, the service writes the event to an **"outbox" table in the same database**, as part of the **same local transaction** as the actual business data change — this is now atomic (both succeed or both roll back together)
2. A separate process (a polling job, or **Change Data Capture** via a tool like Debezium reading the database's transaction log) reads new rows from the outbox table and publishes them to the message broker (Kafka/RabbitMQ)
3. Once successfully published, the outbox row is marked as processed (or deleted)

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);                              // business data change
    outboxRepository.save(new OutboxEvent("OrderPlaced", order.toJson()));  // same transaction
    // both committed together, or both rolled back together — no dual-write gap
}
```

**Why it matters:** it guarantees that an event is published **if and only if** the corresponding database change actually succeeded — closing the gap that a naive "save to DB, then call `kafkaTemplate.send()`" approach leaves open. This is a genuinely senior-level topic, and bringing it up unprompted when discussing event-driven microservices is a strong interview signal.

---

## Extra / Important Interview Questions

**Q16. In a Saga, why is idempotency critical for each participating service?**

Because messages can be **redelivered** (network retries, consumer crashes before acknowledging, at-least-once delivery guarantees common in message brokers), a service might receive the same event more than once. If `reserveStock` isn't idempotent, processing the same "OrderCreated" event twice could double-reserve inventory. Each step in a Saga must be designed so that processing the same message multiple times produces the same end result as processing it once (e.g., checking if the action was already applied before applying it again, often via a unique event/message ID tracked per consumer).

---

**Q17. What's the practical difference between "at-least-once" and "exactly-once" message delivery, and which is more realistic?**

**At-least-once** — the broker guarantees a message is delivered one or more times (never lost, but possibly duplicated); this is the realistic, practically achievable guarantee in almost all real message broker configurations. **Exactly-once** — the message is delivered precisely once, no duplicates, no loss; extremely difficult to guarantee end-to-end across a distributed system (Kafka offers exactly-once semantics **within** its own ecosystem under specific configurations, but true exactly-once across arbitrary producer→broker→consumer→external-side-effect chains is rarely fully achievable). **Practical takeaway:** design consumers to be idempotent and assume at-least-once delivery, rather than relying on exactly-once guarantees holding in every failure scenario.

---

**Q18. Why might you choose Choreography over Orchestration for a Saga, or vice versa?**

**Choreography** fits well for **simple** sagas with few steps — it avoids a central point of failure/complexity and keeps services fully decoupled. It becomes hard to manage as the number of steps grows, because there's no single place to see or reason about the whole flow (each service only knows about the events immediately before/after it). **Orchestration** fits better for **complex** sagas with many steps or conditional branching — a central orchestrator makes the overall business process explicit, testable, and easier to monitor/modify, at the cost of introducing a coordinating component that itself needs to be reliable and scalable.

---

**Q19. If a Circuit Breaker is open and requests are failing fast via the fallback method, is that actually a "successful" response to the end user?**

It depends on what the fallback does — a well-designed fallback should return a **degraded but honest** response (e.g., "recommendations are temporarily unavailable" with an empty list, or cached/stale data with an indication it might be outdated), not silently pretend everything succeeded. The circuit breaker prevents cascading failure and keeps the caller responsive, but the fallback strategy itself needs deliberate design — returning a generic empty success response when the real intent was, say, a payment confirmation would be actively dangerous, whereas a read-only "show cached data" fallback is often perfectly acceptable.

---

**Q20. How would you decide whether a specific piece of inter-service communication should be synchronous (REST/Feign) or asynchronous (Kafka/RabbitMQ)?**

Use **synchronous** when the caller genuinely needs an immediate response to proceed (e.g., checking real-time inventory before confirming a purchase) — the trade-off is tighter coupling and the caller being blocked/affected if the callee is slow or down. Use **asynchronous/event-driven** when the caller doesn't need to wait for the result to continue (e.g., "send a confirmation email after order placement," "update analytics"), or when multiple services need to react to the same event independently — the trade-off is eventual consistency and added architectural complexity (the Outbox pattern, idempotency, message ordering). A common real-world pattern is a **hybrid**: synchronous REST for the immediate, user-facing critical path, and asynchronous events for everything that can happen "after the fact."

---

*Interview tip: Microservices questions at a senior level are rarely "define Kafka" — they're "why did you choose Kafka over RabbitMQ here," "how do you keep the Saga's steps idempotent," or "what happens if this circuit breaker's fallback lies to the user." Ground every answer in a concrete trade-off or failure scenario, not just a definition.*