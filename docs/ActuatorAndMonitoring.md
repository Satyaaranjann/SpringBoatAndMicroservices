# Actuator & Monitoring — Q&A

---

**Q1. What is Spring Boot Actuator?**

Actuator is a Spring Boot module that adds **production-ready operational endpoints** to your application — exposing runtime information (health, metrics, environment, config, thread dumps) over HTTP (or JMX) without you having to build any of it yourself.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

By default, most endpoints are **disabled over HTTP** except `/actuator/health` — you explicitly opt in to expose others:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
```

**Why it matters:** it's the standard way a Spring Boot app tells the outside world (load balancers, Kubernetes, monitoring systems) whether it's healthy, how it's performing, and what's running inside it — without hand-rolling this infrastructure per project.

---

**Q2. What are Health Checks (`/actuator/health`)?**

The `/actuator/health` endpoint reports whether the application (and its critical dependencies — database, disk space, message brokers, etc.) is functioning correctly, returning an aggregated status:

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP", "details": { "database": "PostgreSQL", "validationQuery": "isValid()" } },
    "diskSpace": { "status": "UP", "details": { "total": 500000000000, "free": 100000000000 } },
    "redis": { "status": "UP" }
  }
}
```

**How it works:** Actuator auto-detects relevant `HealthIndicator` beans based on what's on the classpath (a `DataSource` present → DB health check auto-registers, Redis client present → Redis health check auto-registers) and aggregates all of them into one overall `status` (`UP`, `DOWN`, `OUT_OF_SERVICE`, `UNKNOWN`) — if **any** critical component reports `DOWN`, the aggregate status is `DOWN`.

**Custom health indicators:**
```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean gatewayUp = checkPaymentGatewayConnectivity();
        if (gatewayUp) {
            return Health.up().withDetail("gateway", "reachable").build();
        }
        return Health.down().withDetail("gateway", "unreachable").build();
    }
}
```

**Why this matters operationally:** Kubernetes **liveness and readiness probes** typically point directly at `/actuator/health` (or split into `/actuator/health/liveness` and `/actuator/health/readiness` — Spring Boot supports **health groups** for exactly this split) — a `DOWN` status can trigger a pod restart (liveness) or removal from load-balancing rotation (readiness), so health check design directly affects production behavior, not just observability.

---

**Q3. What are Metrics (`/actuator/metrics`)?**

The `/actuator/metrics` endpoint exposes numeric measurements about the application's runtime behavior — JVM memory usage, garbage collection activity, HTTP request counts/latencies, thread pool utilization, custom business metrics, and more.

```
GET /actuator/metrics                    → lists all available metric names
GET /actuator/metrics/jvm.memory.used    → detailed value + tags for that specific metric
GET /actuator/metrics/http.server.requests
```

```json
{
  "name": "http.server.requests",
  "measurements": [
    { "statistic": "COUNT", "value": 1024 },
    { "statistic": "TOTAL_TIME", "value": 45.2 },
    { "statistic": "MAX", "value": 0.89 }
  ],
  "availableTags": [
    { "tag": "method", "values": ["GET", "POST"] },
    { "tag": "status", "values": ["200", "404", "500"] }
  ]
}
```

Metrics can be filtered/drilled into by **tag** (e.g., `?tag=status:500` to isolate error-response timing specifically), which is essential for diagnosing issues at the "which specific endpoint/status combination is slow" level rather than just an aggregate number.

---

**Q4. How do you create Custom Actuator Endpoints?**

Beyond the built-in endpoints, you can expose your own application-specific operational data or actions via `@Endpoint`.

```java
@Component
@Endpoint(id = "featureFlags")
public class FeatureFlagsEndpoint {

    @Autowired
    private FeatureFlagService featureFlagService;

    @ReadOperation
    public Map<String, Boolean> getFeatureFlags() {
        return featureFlagService.getAllFlags();
    }

    @WriteOperation
    public void toggleFlag(@Selector String flagName, boolean enabled) {
        featureFlagService.setFlag(flagName, enabled);
    }
}
```

This registers `GET /actuator/featureFlags` (read) and `POST /actuator/featureFlags/{flagName}` (write, via `@Selector` binding a path segment). You must also explicitly expose it:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, featureFlags
```

**Common real-world use cases:** exposing feature-flag state, cache statistics, a "warm up caches now" trigger endpoint, or custom business-health signals not covered by the built-in indicators.

**Security note (frequently tested):** custom endpoints — especially `@WriteOperation` ones that can *change* application state — must be properly secured (via Spring Security rules on `/actuator/**`), since an unsecured write-capable Actuator endpoint is a real production vulnerability, not just an inconvenience.

---

**Q5. What is Micrometer, and how does it relate to Actuator?**

**Micrometer** is a **metrics facade/instrumentation library** — think of it as "SLF4J, but for metrics" — providing a vendor-neutral API for recording metrics, with pluggable backends (Prometheus, Datadog, New Relic, CloudWatch, and others) so your instrumentation code doesn't need to change if you switch monitoring vendors.

Actuator's `/actuator/metrics` endpoint is actually **backed by Micrometer** underneath — Micrometer collects and stores the metrics; Actuator just exposes a subset of them via its endpoint, while a Micrometer **registry** (e.g., `PrometheusMeterRegistry`) formats and exports them in a vendor-specific way.

**Recording custom metrics:**
```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer processingTimer;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.placed")
            .description("Total number of orders placed")
            .tag("channel", "web")
            .register(registry);

        this.processingTimer = Timer.builder("orders.processing.time")
            .register(registry);
    }

    public void placeOrder(Order order) {
        processingTimer.record(() -> {
            orderRepository.save(order);
            orderCounter.increment();
        });
    }
}
```

**Core metric types:** `Counter` (monotonically increasing, e.g., total requests), `Gauge` (a value that can go up or down, e.g., current queue size), `Timer` (records duration + count of timed events), `DistributionSummary` (records the distribution of event magnitudes, e.g., request payload sizes).

---

**Q6. How does Prometheus and Grafana Integration work?**

**Prometheus** is a time-series monitoring system that **pulls (scrapes)** metrics from applications at regular intervals, storing them for querying and alerting. **Grafana** is a visualization/dashboarding tool that queries Prometheus (or other data sources) to render graphs and dashboards.

**Setup:**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus
  metrics:
    tags:
      application: order-service
```

This exposes `GET /actuator/prometheus`, returning metrics in Prometheus's plain-text exposition format:
```
http_server_requests_seconds_count{method="GET",status="200",uri="/api/orders"} 1024.0
jvm_memory_used_bytes{area="heap",id="G1 Eden Space"} 5.2428800E7
```

**Prometheus configuration** (on the Prometheus server side) tells it where to scrape:
```yaml
scrape_configs:
  - job_name: 'order-service'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['order-service:8080']
```

**Grafana** then connects to Prometheus as a data source and builds dashboards (request rate, error rate, latency percentiles, JVM heap usage) using **PromQL** queries against the scraped metrics — and can be configured to fire **alerts** based on thresholds (e.g., error rate > 5% for 5 minutes).

**The flow end-to-end:** App (Micrometer records metrics) → exposes via `/actuator/prometheus` → Prometheus scrapes and stores → Grafana queries and visualizes/alerts. This is the de-facto standard observability stack for Spring Boot microservices.

---

**Q7. What is Application Logging with SLF4J and Logback?**

**SLF4J (Simple Logging Facade for Java)** is a **logging facade** — an abstraction that lets your code call a consistent logging API without being tied to a specific logging implementation. **Logback** is the actual logging implementation Spring Boot uses **by default**, and SLF4J calls are routed to it.

```java
@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void placeOrder(Order order) {
        log.info("Placing order for user {}", order.getUserId());
        try {
            processPayment(order);
        } catch (PaymentException e) {
            log.error("Payment failed for order {}", order.getId(), e);
        }
        log.debug("Order details: {}", order);
    }
}
```

**Why the facade pattern matters:** you (or a library you depend on) can swap the underlying implementation (Logback → Log4j2, for instance) without changing any application logging code — everything still calls the same SLF4J API.

**Log levels (ascending severity):** `TRACE` < `DEBUG` < `INFO` < `WARN` < `ERROR`. Configuring a level (e.g., `INFO`) means that level **and everything above it** are logged; `TRACE`/`DEBUG` are suppressed by default in production.

```yaml
logging:
  level:
    root: INFO
    com.example.app: DEBUG      # more verbose logging just for your own package
    org.hibernate.SQL: DEBUG    # see actual SQL queries Hibernate generates
```

**Key practice:** always use the `{}` placeholder syntax (`log.info("User {} logged in", userId)`) rather than string concatenation (`log.info("User " + userId + " logged in")`) — the placeholder version avoids building the string at all if the log level is disabled, which matters for performance in hot code paths.

---

**Q8. What is Structured/JSON Logging, and why use it over plain text logs?**

**Structured logging** outputs log entries as **machine-parseable data** (typically JSON) rather than free-form text — each field (timestamp, level, message, trace ID, custom context) is a distinct, queryable property instead of being embedded in an unstructured sentence.

**Plain text (traditional):**
```
2026-08-09 10:15:30 INFO  [order-service,a1b2c3,x9y8z] OrderService - Placing order for user 42
```

**Structured JSON:**
```json
{
  "timestamp": "2026-08-09T10:15:30.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Placing order for user 42",
  "traceId": "a1b2c3",
  "spanId": "x9y8z",
  "service": "order-service",
  "userId": 42
}
```

**Why it matters at scale:** in a microservices environment, logs from many services get aggregated into a centralized system (ELK stack — Elasticsearch/Logstash/Kibana, or Splunk, Datadog Logs, etc.). JSON logs can be **filtered, searched, and aggregated by field** directly (e.g., "show me all ERROR logs from `order-service` where `userId=42`") — plain text logs require fragile regex parsing to extract the same structured information, which breaks easily if the log message format changes.

**Configuring JSON output with Logback (`logback-spring.xml`):**
```xml
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>
    <root level="INFO">
        <appender-ref ref="JSON" />
    </root>
</configuration>
```
(using the `logstash-logback-encoder` library)

**Adding contextual fields via MDC (Mapped Diagnostic Context):** the `traceId`/`spanId`/`userId` fields above typically come from Slf4j's **MDC**, which lets you attach request-scoped context that automatically appears on every log line for that request, without manually passing it into every log statement:
```java
MDC.put("userId", String.valueOf(userId));
log.info("Processing request");   // userId automatically included in structured output
MDC.clear();   // important — clear after the request to avoid leaking context into unrelated logs
```

---

## Extra / Important Interview Questions

**Q9. Why should `/actuator/**` endpoints never be left fully open/unsecured in production?**

Actuator endpoints can expose **sensitive information** — `/actuator/env` can leak configuration values (potentially including secrets if not properly masked), `/actuator/heapdump` can expose an entire memory snapshot (which may contain credentials, session tokens, or PII in memory), and write-capable custom endpoints could let an attacker change application behavior. Best practice: expose only the minimum necessary endpoints publicly (`health`, sometimes `info`), keep everything else restricted to an internal network/VPN, and apply Spring Security rules requiring authentication (often a separate admin role) for any endpoint beyond basic health.

```java
.requestMatchers("/actuator/health").permitAll()
.requestMatchers("/actuator/**").hasRole("ADMIN")
```

---

**Q10. What's the difference between liveness and readiness probes, and how does `/actuator/health` support both?**

- **Liveness** — "is the application in a state where it should be **restarted**?" (e.g., deadlocked, unrecoverable internal state) — a failing liveness probe typically triggers a **pod restart** in Kubernetes
- **Readiness** — "is the application currently able to **serve traffic**?" (e.g., still warming up, or a downstream dependency is temporarily unavailable) — a failing readiness probe removes the pod from load-balancer rotation **without restarting it**, since the issue might be transient

Spring Boot supports this natively via **health groups**:
```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
management.health.livenessstate.enabled: true
management.health.readinessstate.enabled: true
```
Exposing `/actuator/health/liveness` and `/actuator/health/readiness` separately — conflating the two (using one health check for both) is a common misconfiguration that causes unnecessary pod restarts for what should just be a temporary "not ready" state.

---

**Q11. Why might a `Counter` metric alone be misleading without also tracking a corresponding `Timer` or rate?**

A raw counter (e.g., `orders.placed` total = 50,000) tells you volume but nothing about **rate or health** — 50,000 orders over a year is very different from 50,000 orders in the last hour, and the counter alone can't distinguish a healthy high-traffic day from a runaway retry loop generating spurious duplicate records. This is why metrics dashboards typically graph **rate of change** (`rate(orders_placed_total[5m])` in PromQL) rather than raw cumulative counter values, and why counters are usually paired with a `Timer` (to catch performance degradation) or an error-rate counter (to catch correctness issues) rather than relied upon alone.

---

**Q12. If MDC context (like `traceId`) isn't cleared after a request, what problem can occur?**

Because MDC is stored in a `ThreadLocal`, and application servers/thread pools **reuse threads** across requests, failing to clear MDC at the end of a request can cause a **subsequent, unrelated request** (handled by the same reused thread) to inherit stale context from the previous request — e.g., logs for Request B incorrectly showing Request A's `traceId` or `userId`. This is a subtle, hard-to-notice bug that corrupts log correlation silently; the fix is ensuring MDC is cleared in a `finally` block or via a filter/interceptor that wraps every request lifecycle.

---

**Q13. How would you use custom tags on a metric to answer a specific operational question, e.g., "which payment provider has the highest failure rate"?**

Tag the metric at the point of recording with the relevant dimension, rather than creating separate metric names per provider:
```java
Counter.builder("payment.attempts")
    .tag("provider", provider.name())
    .tag("outcome", success ? "success" : "failure")
    .register(registry)
    .increment();
```
This lets you query/filter/aggregate by `provider` and `outcome` independently in Prometheus/Grafana (`payment_attempts_total{outcome="failure"}` grouped by `provider`), without needing a separate counter per provider — tags are the mechanism for slicing one logical metric along multiple dimensions, which is a core Micrometer/Prometheus design principle worth demonstrating explicit understanding of.

---

*Interview tip: Actuator/monitoring questions increasingly probe **operational judgment** — securing endpoints, liveness vs readiness distinctions, why a raw counter can mislead — rather than just "what does `/actuator/health` return." Demonstrating you've thought about what happens when these tools are misconfigured is what separates a strong answer.*