# Reactive Programming — Q&A

---

**Q1. What is Spring WebFlux, and how is it different from Spring MVC?**

**Spring WebFlux** is Spring's **reactive**, non-blocking web framework — an alternative to Spring MVC, built to handle high-concurrency workloads efficiently using a small, fixed number of threads instead of one thread per request.

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Programming model | Imperative, blocking | Reactive, non-blocking |
| Threading model | One thread per request (thread-per-request) | Small, fixed event-loop thread pool handles many requests |
| Underlying server | Servlet API (Tomcat, Jetty) | Netty by default (also supports Servlet 3.1+ async, Undertow) |
| Return types | Plain objects, `ResponseEntity<T>` | `Mono<T>`, `Flux<T>` |
| Best suited for | Traditional CRUD apps, most business applications | High-concurrency, I/O-heavy workloads (streaming, many slow downstream calls) |
| Learning curve | Familiar, straightforward | Steeper — requires reactive-style thinking throughout the stack |

```java
// Spring MVC — blocking
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id);   // thread blocks until this returns
}

// Spring WebFlux — non-blocking
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userService.findById(id);   // returns immediately; result delivered asynchronously
}
```

**Key interview point:** WebFlux isn't "faster" for every workload — it shines specifically when your app spends most of its time **waiting on I/O** (network calls, database queries) with high concurrency, because non-blocking I/O lets a small thread pool serve thousands of concurrent in-flight requests without each one holding a thread hostage while waiting. For CPU-bound or low-concurrency workloads, the added complexity of reactive programming often isn't worth it — Spring MVC remains the right default for most applications.

---

**Q2. What are `Mono` and `Flux`?**

They're the two core reactive types from **Project Reactor** (the reactive library WebFlux is built on), both implementing the **Reactive Streams `Publisher`** interface:

- **`Mono<T>`** — represents **0 or 1** asynchronous result (like a reactive `Optional`/`CompletableFuture`)
- **`Flux<T>`** — represents **0 to N** asynchronous results, a stream of items over time (like a reactive `Stream`/`List`, but items can arrive incrementally rather than all at once)

```java
Mono<User> userMono = userRepository.findById(1L);          // 0 or 1 user
Flux<User> usersFlux = userRepository.findByActive(true);   // 0 to many users

userMono.subscribe(user -> System.out.println("Got: " + user));

usersFlux
    .filter(u -> u.getAge() > 18)
    .map(User::getName)
    .subscribe(name -> System.out.println("Adult user: " + name));
```

**Nothing happens until you subscribe** — `Mono`/`Flux` are **lazy**; defining a chain of operations (`.map()`, `.filter()`, etc.) just builds a pipeline description. Execution only begins when something **subscribes** to it (in a WebFlux controller, the framework itself subscribes when handling the HTTP response). This "cold" vs "hot" publisher distinction and the laziness itself are frequently tested — many candidates incorrectly assume operations run eagerly like a normal Java `Stream` intermediate chain (which is actually similarly lazy, a useful analogy to draw on).

**Common operators:** `map` (transform each item), `flatMap` (transform into another `Mono`/`Flux` and flatten — used when the transformation itself is async), `filter`, `zip` (combine multiple publishers), `onErrorResume`/`onErrorReturn` (error handling), `block()`/`blockFirst()` (escape hatch to get a synchronous value — generally avoided in reactive code, since it defeats the purpose).

---

**Q3. What are Reactive Streams?**

**Reactive Streams** is a **specification** (not a Spring or Project Reactor invention) for asynchronous stream processing with **non-blocking backpressure** — a standard implemented by multiple libraries (Project Reactor, RxJava, Akka Streams), ensuring interoperability between them.

**Four core interfaces:**
- **`Publisher<T>`** — produces a stream of items (`Mono`/`Flux` implement this)
- **`Subscriber<T>`** — consumes items from a `Publisher`
- **`Subscription`** — the link between a `Publisher` and `Subscriber`, used to request items and cancel the stream
- **`Processor<T,R>`** — acts as both a `Subscriber` and `Publisher` (a transformation stage)

**Backpressure** is the defining concept: a slow **consumer** can tell a fast **producer** to slow down, via `Subscription.request(n)` — explicitly requesting only as many items as it can currently handle, rather than the producer overwhelming it by pushing data as fast as possible. This is the fundamental problem Reactive Streams solves that a simple callback-based async API doesn't — without backpressure, a fast producer and slow consumer combination leads to unbounded buffering and eventual `OutOfMemoryError`.

```java
Flux.range(1, 1000)
    .subscribe(new Subscriber<Integer>() {
        private Subscription subscription;

        @Override
        public void onSubscribe(Subscription s) {
            this.subscription = s;
            s.request(10);   // only ask for 10 items at a time — backpressure in action
        }

        @Override
        public void onNext(Integer item) {
            process(item);
            subscription.request(1);   // request one more after processing
        }

        @Override
        public void onError(Throwable t) { }

        @Override
        public void onComplete() { }
    });
```

In practice, most application code doesn't implement `Subscriber` manually — Project Reactor's operators handle backpressure internally, but understanding the concept is essential for explaining *why* reactive programming exists at all, beyond "it's async."

---

**Q4. How does `WebClient` work for Reactive Calls?**

`WebClient` is Spring's non-blocking HTTP client (replacing `RestTemplate`), returning `Mono`/`Flux` so calls can be composed reactively without blocking a thread while waiting for a response.

```java
@Service
public class UserClient {

    private final WebClient webClient;

    public UserClient(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://user-service").build();
    }

    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, response -> Mono.error(new UserNotFoundException()))
            .bodyToMono(User.class);
    }

    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/api/users")
            .retrieve()
            .bodyToFlux(User.class);
    }
}
```

**Composing multiple reactive calls (this is where WebClient's real value shows):**
```java
public Mono<OrderSummary> buildOrderSummary(Long orderId) {
    Mono<Order> orderMono = orderClient.getOrder(orderId);
    return orderMono.flatMap(order ->
        userClient.getUser(order.getUserId())
            .zipWith(inventoryClient.checkStock(order.getProductId()))
            .map(tuple -> new OrderSummary(order, tuple.getT1(), tuple.getT2()))
    );
    // both downstream calls execute without blocking any thread while waiting
}
```

**Important:** `WebClient` can be used in a **traditional Spring MVC application** too (not just WebFlux apps) — you just call `.block()` at the boundary to get a synchronous value when needed. Spring recommends `WebClient` over `RestTemplate` for all new code, regardless of whether the rest of the app is reactive.

---

**Q5. What is R2DBC, and how does it enable Reactive Database Access?**

**R2DBC (Reactive Relational Database Connectivity)** is a specification for **non-blocking** access to relational databases — solving a gap that traditional JDBC can't: JDBC is fundamentally **blocking** at the driver level (a thread blocks while waiting for a query to return), which defeats the purpose of an otherwise fully non-blocking WebFlux application if the database call blocks a thread anyway.

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByActive(boolean active);

    @Query("SELECT * FROM users WHERE age > :age")
    Flux<User> findByAgeGreaterThan(int age);
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public Mono<User> getUser(Long id) {
        return userRepository.findById(id);
    }

    public Flux<User> getActiveUsers() {
        return userRepository.findByActive(true);
    }
}
```

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: user
    password: pass
```

**Key differences from Spring Data JPA:**
- Returns `Mono`/`Flux` instead of entities/lists/`Optional`
- No first-level cache or dirty-checking/persistence-context magic like JPA/Hibernate — R2DBC is a much thinner, more explicit layer
- No lazy-loading of relationships — you fetch exactly what you query for, explicitly (a deliberate design choice, since lazy-loading fundamentally conflicts with non-blocking reactive semantics)
- Ecosystem/tooling maturity is behind JPA — fewer database driver options, less feature parity (e.g., complex entity graph mapping is more manual)

**When to use R2DBC vs JPA:** only when the entire call chain is genuinely reactive/WebFlux-based and needs to avoid blocking the database call specifically — using R2DBC inside an otherwise-blocking Spring MVC application provides no benefit and adds complexity for nothing.

---

**Q6. Blocking vs Non-Blocking I/O — what's the fundamental difference?**

**Blocking I/O** — when a thread makes an I/O call (network request, DB query, file read), it **stops and waits** until the operation completes before doing anything else. The thread is occupied/idle for the entire duration, unable to do other work.

**Non-blocking I/O** — a thread **initiates** the I/O operation and immediately moves on to other work; when the I/O operation eventually completes, a **callback/event** notifies the system, and processing resumes (often on a different thread from a small pool) — no thread sits idle waiting.

**Why this matters at scale — the concrete numbers argument:**
- **Thread-per-request (blocking, Spring MVC/Tomcat default)** — each concurrent request holds its own thread for its entire duration, including all the time it's waiting on a slow database or downstream API. Threads are relatively expensive (each has its own stack, typically ~1MB), so a server can practically support maybe a few hundred to low-thousands of concurrent connections before running out of threads/memory, even though the CPU itself is mostly idle (waiting on I/O, not computing).
- **Event-loop (non-blocking, WebFlux/Netty)** — a small, fixed number of threads (often close to the CPU core count) handle **all** concurrent requests, because no thread blocks waiting on I/O — while one request's I/O is in flight, the same thread services other requests. This allows a single server to handle far more concurrent connections with the same hardware, **specifically for I/O-bound workloads**.

**The critical caveat interviewers listen for:** non-blocking I/O only helps when the bottleneck is genuinely I/O wait time (network/DB latency), not CPU work. If your workload is CPU-intensive (heavy computation, not waiting on external calls), the event-loop model provides no advantage and can even hurt — a long-running CPU task on an event-loop thread **blocks that thread from servicing any other request**, which is a serious anti-pattern in reactive code (accidentally running blocking/CPU-heavy code on the small reactive thread pool defeats the entire model and can stall the whole application).

---

## Extra / Important Interview Questions

**Q7. What happens if you accidentally call a blocking method (e.g., a legacy JDBC call, or `Thread.sleep()`) inside a WebFlux reactive chain?**

It blocks one of the small, fixed number of event-loop threads — since WebFlux relies on **never** blocking those threads, a single blocking call can stall the processing of **many other concurrent requests** sharing that same thread pool, potentially degrading the entire application's throughput far more severely than it would in a traditional thread-per-request model (where blocking only affects that one request's dedicated thread). This is arguably the single most important operational rule in reactive programming: **never block on a reactive thread**. If you must call blocking code, it should be explicitly offloaded to a separate bounded elastic thread pool via `.subscribeOn(Schedulers.boundedElastic())`.

---

**Q8. What's the difference between `.subscribeOn()` and `.publishOn()` in Project Reactor?**

- **`.subscribeOn()`** — controls which thread/scheduler the **entire subscription chain starts on** (affects the source/beginning of the chain), regardless of where it's placed in the chain
- **`.publishOn()`** — switches the thread/scheduler for **everything downstream** of where it's placed in the chain, letting you change execution context partway through a pipeline

```java
Mono.fromCallable(() -> blockingLegacyCall())
    .subscribeOn(Schedulers.boundedElastic())   // the blocking call runs on a dedicated elastic pool, not the event loop
    .map(this::transform)                        // runs on that same boundedElastic thread
    .publishOn(Schedulers.parallel())
    .map(this::furtherProcessing);                // this and everything after runs on the parallel scheduler
```

This distinction is a genuinely advanced/senior-level question — many candidates who can define `Mono`/`Flux` haven't dug into scheduler control at this level.

---

**Q9. Can you mix Spring MVC (blocking) and WebFlux (reactive) code in the same application? Should you?**

Technically, `WebClient` can be used inside a Spring MVC app (blocking at the boundary via `.block()`), so limited mixing is possible and common. However, running **both** `spring-webmvc` and `spring-webflux` as competing web frameworks in the same app is explicitly unsupported/discouraged — Spring Boot auto-configuration expects one or the other, and mixing them fully creates ambiguity about which server/threading model actually handles requests. The realistic pattern is: pick one model for the application's web layer, and use `WebClient` (with `.block()` if needed) as an HTTP client library within either.

---

**Q10. Why would a team choose to migrate an existing Spring MVC application to WebFlux, and what's the realistic cost of doing so?**

**Legitimate reasons:** the application is I/O-bound with very high concurrency requirements (e.g., a gateway/proxy service, or a service making many concurrent downstream calls per request) where thread-per-request genuinely becomes the scaling bottleneck, and the team has verified this with actual load testing/profiling — not just "reactive sounds more modern."

**Realistic costs:** the entire call chain must become non-blocking to see the benefit — a single blocking JDBC call or synchronous library buried three layers deep silently negates the advantage (and actively risks the thread-starvation problem from Q7). This typically means migrating from JPA to R2DBC (losing ecosystem maturity), rewriting business logic in a more complex reactive style (harder to read/debug — stack traces are less intuitive, and step-through debugging is harder), and retraining the team. **The strong interview answer:** most applications should *not* migrate to WebFlux — the operational and cognitive cost is real, and Spring MVC (with virtual threads in newer JDK/Spring versions, which capture much of the throughput benefit without the reactive programming complexity) is often now the better answer to "how do I handle high concurrency" than a full reactive rewrite.

---

*Interview tip: Reactive programming questions reward knowing **when not to use it** as much as how it works. A candidate who says "WebFlux only helps for I/O-bound, high-concurrency workloads, and blocking calls on the event loop can actively hurt performance" demonstrates far deeper understanding than one who can only recite `Mono`/`Flux` operator syntax.*