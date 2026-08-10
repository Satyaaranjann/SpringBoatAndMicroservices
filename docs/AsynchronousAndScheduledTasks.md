# Asynchronous & Scheduled Tasks — Q&A

---

**Q1. What are `@EnableAsync` and `@Async`?**

`@EnableAsync` activates Spring's asynchronous method execution capability at the application level; without it, `@Async` annotations are silently ignored (same category of "gotcha" as `@EnableCaching`/`@Cacheable`).

`@Async` marks a method to run on a **separate thread**, so the calling thread doesn't block waiting for it to finish.

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}

@Service
public class NotificationService {

    @Async
    public void sendEmail(String to, String subject) {
        // runs on a background thread — caller doesn't wait
        emailClient.send(to, subject);
    }
}
```

**How it works under the hood:** like `@Transactional` and `@Cacheable`, `@Async` relies on **Spring AOP proxies** — calling an `@Async` method submits the actual execution to a `TaskExecutor` (thread pool) and returns control to the caller immediately (for `void`/fire-and-forget methods) or returns a `Future`/`CompletableFuture` handle (for methods that need a result).

**Same self-invocation gotcha applies:** calling an `@Async` method from within the same class (`this.sendEmail(...)`) bypasses the proxy and runs synchronously on the calling thread — a very common bug and a favorite interview trap, identical in nature to the `@Transactional` self-invocation issue.

---

**Q2. How does `CompletableFuture` work with Async Methods?**

When an `@Async` method needs to **return a result** (not just fire-and-forget), it should return `CompletableFuture<T>` (or the older `Future<T>`, though `CompletableFuture` is preferred for its richer composition API).

```java
@Service
public class ReportService {

    @Async
    public CompletableFuture<Report> generateReport(Long userId) {
        Report report = expensiveReportGeneration(userId);
        return CompletableFuture.completedFuture(report);
    }
}
```

**Calling and composing multiple async calls:**
```java
@Service
public class DashboardService {

    @Autowired private ReportService reportService;
    @Autowired private AnalyticsService analyticsService;

    public DashboardData buildDashboard(Long userId) {
        CompletableFuture<Report> reportFuture = reportService.generateReport(userId);
        CompletableFuture<Analytics> analyticsFuture = analyticsService.fetchAnalytics(userId);

        // both run concurrently; combine results once both complete
        return CompletableFuture.allOf(reportFuture, analyticsFuture)
            .thenApply(v -> new DashboardData(reportFuture.join(), analyticsFuture.join()))
            .join();   // blocks here, waiting for both to finish
    }
}
```

**Why `CompletableFuture` over plain `Future`:**
- `Future.get()` **blocks** with no built-in way to chain further async operations
- `CompletableFuture` supports non-blocking composition — `.thenApply()`, `.thenCombine()`, `.thenCompose()`, `.exceptionally()` — letting you chain, combine, and handle errors across multiple async calls without manually blocking at each step

**Important:** `@Async` methods must **not** be `private` and must be called from **outside the class** (through the Spring-managed proxy) for the async behavior to actually apply — same proxy limitation as `@Transactional`.

---

**Q3. What are `@EnableScheduling` and `@Scheduled`?**

`@EnableScheduling` activates Spring's task scheduling infrastructure. `@Scheduled` marks a method to run automatically and repeatedly according to a fixed rate, fixed delay, or cron expression — without any external scheduler (like a system cron job or Quartz) needed for simple cases.

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
}

@Component
public class ReportScheduler {

    @Scheduled(fixedRate = 60000)          // runs every 60 seconds, measured from start to start
    public void refreshCache() { ... }

    @Scheduled(fixedDelay = 30000)         // runs 30 seconds after the PREVIOUS execution finishes
    public void syncData() { ... }

    @Scheduled(initialDelay = 5000, fixedRate = 60000)  // waits 5s before the first run, then every 60s
    public void warmUp() { ... }

    @Scheduled(cron = "0 0 2 * * *")       // every day at 2:00 AM
    public void nightlyCleanup() { ... }
}
```

**`fixedRate` vs `fixedDelay` — the key distinction (frequently tested):**
- **`fixedRate`** — measures the interval from the **start** of one execution to the **start** of the next; if a task takes longer than the rate, executions can queue up or overlap depending on the executor
- **`fixedDelay`** — measures the interval from the **end** of one execution to the **start** of the next; guarantees no overlap, since the next run only begins after the previous one fully completes

**Default execution:** without additional configuration, all `@Scheduled` methods run on a **single-threaded** default scheduler — meaning multiple scheduled tasks execute **sequentially**, not concurrently, which can silently delay other scheduled jobs if one takes a long time. This is a very common production gotcha, solved by configuring a proper thread pool (see Q5).

---

**Q4. What are Cron Expressions?**

A cron expression defines a recurring schedule using a compact string format. Spring's cron format uses **6 fields** (unlike traditional Unix cron's 5 — Spring adds a leading **seconds** field):

```
┌───────────── second (0-59)
│ ┌───────────── minute (0-59)
│ │ ┌───────────── hour (0-23)
│ │ │ ┌───────────── day of month (1-31)
│ │ │ │ ┌───────────── month (1-12)
│ │ │ │ │ ┌───────────── day of week (0-7, both 0 and 7 = Sunday)
│ │ │ │ │ │
* * * * * *
```

**Common examples:**

| Cron Expression | Meaning |
|---|---|
| `0 0 * * * *` | Every hour, on the hour |
| `0 0 2 * * *` | Every day at 2:00 AM |
| `0 0 9 * * MON-FRI` | Every weekday at 9:00 AM |
| `0 */15 * * * *` | Every 15 minutes |
| `0 0 0 1 * *` | Midnight on the 1st of every month |
| `0 30 8 * * SAT,SUN` | 8:30 AM every Saturday and Sunday |

```java
@Scheduled(cron = "0 0 9 * * MON-FRI")
public void sendDailyReport() { ... }
```

**Best practice:** externalize cron expressions into `application.yml` rather than hardcoding them, so schedules can be changed per environment without recompiling:
```java
@Scheduled(cron = "${report.schedule.cron}")
```

---

**Q5. How do you configure a Thread Pool for `@Async` and `@Scheduled` tasks?**

By default, both `@Async` and `@Scheduled` use a minimal default executor (`@Async` uses `SimpleAsyncTaskExecutor`, which creates a **new thread per task** with no pooling/reuse — inefficient and unbounded; `@Scheduled` defaults to a **single-threaded** scheduler). For any real application, you should configure a proper, bounded thread pool explicitly.

**For `@Async`:**
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) -> log.error("Async error in {}: {}", method.getName(), throwable.getMessage());
    }
}
```

**For `@Scheduled`:**
```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);           // allows multiple @Scheduled methods to run concurrently
        scheduler.setThreadNamePrefix("Scheduled-");
        scheduler.initialize();
        taskRegistrar.setTaskScheduler(scheduler);
    }
}
```

**Key pool sizing parameters (for `ThreadPoolTaskExecutor`):**
- **`corePoolSize`** — minimum number of threads kept alive, even if idle
- **`maxPoolSize`** — maximum number of threads allowed under load
- **`queueCapacity`** — how many tasks can wait in the queue once all core threads are busy, before new threads (up to `maxPoolSize`) are spun up
- **`RejectedExecutionHandler`** — policy for what happens when the queue is full **and** `maxPoolSize` is reached (default: throws `RejectedExecutionException`)

**Why this matters:** using multiple named executors (e.g., a separate pool for email-sending vs report-generation) lets you isolate slow/heavy tasks from fast ones, preventing one overloaded task type from starving others by consuming all available threads.

---

## Extra / Important Interview Questions

**Q6. Why does `@Async` on a `void` method behave differently from `@Async` returning `CompletableFuture`, in terms of error handling?**

A `void` `@Async` method is truly "fire and forget" — if it throws an exception, the caller **never sees it** (there's no return value to propagate the failure through). These exceptions must be handled via a registered `AsyncUncaughtExceptionHandler` (see Q5's example), or they're simply logged by Spring's default handler and otherwise silently lost. A `CompletableFuture`-returning method, by contrast, **captures** the exception in the future itself — the caller can inspect it via `.exceptionally()`, `.handle()`, or by catching it when calling `.get()`/`.join()`. This is a strong interview signal question — many candidates don't realize `void` async methods lose exceptions by default.

---

**Q7. What happens if the default single-threaded scheduler is used and one `@Scheduled` method takes a very long time to run?**

All other `@Scheduled` methods in the application are **delayed**, because they share the single scheduling thread — the scheduler processes tasks sequentially, one at a time, by default. A long-running job (e.g., a nightly report that takes 20 minutes) can silently push back every other scheduled task queued behind it. This is precisely why configuring a proper `ThreadPoolTaskScheduler` with multiple threads (Q5) is considered a production essential, not an optional optimization.

---

**Q8. What's the difference between `@Scheduled(fixedRate = ...)` and running the same logic via a Unix cron job that calls an endpoint?**

`@Scheduled` runs **inside the JVM process** of the application itself — if the app isn't running, the schedule doesn't fire, and if you scale to multiple instances, **each instance runs its own copy** of the schedule (risking the same job running multiple times simultaneously unless you add distributed locking). An external Unix cron job (or `Quartz` with a shared database) hitting an endpoint runs independently of any single app instance, and can be centrally controlled — but adds an external moving part outside the app. For multi-instance deployments where a job must run **exactly once** regardless of how many instances are up, `@Scheduled` alone is insufficient without additional coordination (e.g., ShedLock, a distributed lock library, or delegating to a single elected instance).

---

**Q9. Can `@Transactional` and `@Async` be combined on the same method? What's the catch?**

Yes, but with an important caveat: since `@Async` runs the method on a **new thread**, and Spring's default transaction management is typically thread-bound (using `ThreadLocal` to track the current transaction), an `@Async @Transactional` method effectively starts its own **independent** transaction on that new thread — it does **not** automatically join or inherit any transaction from the calling thread. This surprises developers expecting the async method's DB operations to be part of the same transaction as the caller — they never are, by design, because there's no way to safely share a transaction across threads.

---

**Q10. How would you test a method annotated with `@Async`?**

Since the method returns immediately (or asynchronously) rather than blocking, naive tests can finish and assert results **before** the async work actually completes, leading to flaky false failures. Approaches:
- If it returns `CompletableFuture<T>`, call `.get()` (with a timeout) in the test to block until completion before asserting
- For `void` async methods, use a synchronization mechanism like `CountDownLatch`, or `Awaitility`'s `await().until(...)` polling, to wait for a side effect to occur before asserting
- Alternatively, configure a **synchronous** `TaskExecutor` (e.g., `SyncTaskExecutor`) specifically in the test profile, so `@Async` methods effectively run synchronously during tests — trading realism for simpler, non-flaky assertions

---

*Interview tip: Async/scheduling questions reward understanding what happens when things go wrong under load — a slow task blocking others, lost exceptions in fire-and-forget methods, or duplicate job runs across scaled instances. These production-failure scenarios are usually the actual differentiator in interviews, more than syntax recall.*