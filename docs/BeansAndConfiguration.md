# Beans & Configuration — Q&A

---

**Q1. What is a Bean?**

A Bean is simply a Java object that is instantiated, assembled, and managed by the Spring IoC container. Instead of your code calling `new` to create an object, you declare it (via annotation or Java config), and Spring takes over its full lifecycle — creation, dependency injection, initialization, and eventual destruction. Any object registered with the container becomes a "bean" and is retrievable by type or name.

---

**Q2. What is the Bean Lifecycle in Spring?**

The lifecycle describes the stages a bean goes through from creation to destruction:

1. **Instantiation** — Spring creates the bean instance (via constructor)
2. **Populate properties** — dependencies are injected (constructor/setter/field)
3. **`BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`** callbacks (if implemented)
4. **`@PostConstruct`** / `InitializingBean.afterPropertiesSet()` — custom init logic runs
5. **Custom `init-method`** (if configured) runs
6. **Bean is ready** — fully initialized and available for use
7. **`@PreDestroy`** / `DisposableBean.destroy()` — runs when the container shuts down
8. **Custom `destroy-method`** (if configured) runs

Understanding this order matters when you need to run logic right after dependencies are injected, or clean up resources (closing connections, threads) before shutdown.

---

**Q3. What are Bean Scopes? Explain Singleton, Prototype, Request, and Session.**

Scope defines how many instances of a bean the container creates and how long they live:

| Scope | Behavior |
|---|---|
| **Singleton** (default) | One instance per Spring container, shared everywhere it's injected |
| **Prototype** | A new instance is created every time the bean is requested/injected |
| **Request** | One instance per HTTP request (web-aware, only valid in a web context) |
| **Session** | One instance per HTTP session (web-aware) |

`@Scope("prototype")` or `@Scope("singleton")` is used to declare it explicitly. Singleton is by far the most common — use prototype only when a bean holds request-specific mutable state that shouldn't be shared.

---

**Q4. Difference between `@Component`, `@Service`, `@Repository`, `@Controller`?**

All four are specializations of `@Component`, meaning all are detected by component scanning and registered as beans. The difference is semantic meaning plus, in one case, actual added behavior:

- **`@Component`** — generic stereotype, used when a class doesn't fit the other categories
- **`@Service`** — marks the business/service layer (purely semantic, no extra behavior)
- **`@Repository`** — marks the data access layer; Spring adds automatic **exception translation**, converting JDBC/JPA-specific exceptions into Spring's unified `DataAccessException` hierarchy
- **`@Controller`** — marks a web layer class handling HTTP requests, used with Spring MVC's view resolution

Using the right annotation makes code more readable and lets tools/frameworks apply layer-specific behavior automatically.

---

**Q5. Difference between `@Bean` and `@Configuration`?**

- **`@Configuration`** — marks a class as a source of bean definitions, equivalent to an XML config file in Java form
- **`@Bean`** — placed on a method inside a `@Configuration` class; the method's return value is registered as a bean, with the method name (by default) becoming the bean name

```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

`@Bean` is typically used when you need to configure third-party classes (that you don't own and can't annotate with `@Component`) or when bean creation needs custom logic.

---

**Q6. What is `@Autowired`?**

`@Autowired` tells Spring to automatically resolve and inject a matching bean into a constructor, field, or setter. Spring first tries to match **by type**; if multiple beans of the same type exist, it falls back to matching **by name** or requires disambiguation via `@Qualifier`. By default it's required (throws an exception if no matching bean is found) unless `@Autowired(required = false)` is set.

---

**Q7. What are `@Qualifier` and `@Primary`? How do they differ?**

Both solve the same problem — multiple beans of the same type causing ambiguity during injection — but in different ways:

- **`@Primary`** — marks one bean as the default choice when multiple candidates exist; used when there's a clear "default" implementation
- **`@Qualifier("beanName")`** — explicitly specifies which bean to inject at the injection point, used when the choice depends on context rather than a fixed default

```java
@Bean
@Primary
public PaymentService creditCardPayment() { ... }

@Bean
public PaymentService paypalPayment() { ... }

// Injects paypalPayment specifically, overriding @Primary
@Autowired
@Qualifier("paypalPayment")
private PaymentService paymentService;
```

`@Qualifier` at the injection point always wins over `@Primary` when both are present.

---

**Q8. Constructor vs Setter vs Field Injection — compare all three.**

| Type | How it works | Pros | Cons |
|---|---|---|---|
| **Constructor** | Dependencies passed via constructor params | Immutable (`final`), dependencies explicit, fails fast, best for testing | Can get verbose with many dependencies (often a design smell) |
| **Setter** | Dependencies set via setter methods | Good for optional dependencies, allows reconfiguration | Object can exist in a partially-initialized state before setters run |
| **Field** | `@Autowired` directly on the field | Least boilerplate | Hides dependencies, can't be `final`, harder to unit test without a container |

**Recommended practice:** constructor injection for required dependencies, setter injection for optional ones, and avoid field injection in production code.

---

**Q9. What are `@PostConstruct` and `@PreDestroy`?**

Lifecycle callback annotations (from `jakarta.annotation`) used to hook into bean initialization and destruction:

- **`@PostConstruct`** — method runs once, right after dependency injection is complete, before the bean is put into use. Common for validation, warming up caches, or opening resources.
- **`@PreDestroy`** — method runs just before the bean is removed from the container (on context shutdown). Common for closing connections, releasing resources, or graceful cleanup.

```java
@Component
public class CacheService {
    @PostConstruct
    public void init() { loadCache(); }

    @PreDestroy
    public void cleanup() { closeConnections(); }
}
```

Note: `@PreDestroy` only fires for **singleton** beans on container shutdown — it does **not** fire for prototype-scoped beans, since Spring doesn't manage their full lifecycle after creation.

---

**Q10. What is `@Lazy` Initialization?**

By default, singleton beans are created **eagerly** — at application startup. `@Lazy` defers bean creation until it's first requested/injected somewhere.

```java
@Component
@Lazy
public class HeavyResourceService { ... }
```

**Use cases:**
- Expensive-to-create beans that aren't always needed
- Breaking circular dependency issues
- Faster application startup when a bean's initialization isn't required immediately

**Trade-off:** errors in a lazy bean's configuration surface later (at first use) rather than at startup, which can delay discovering configuration problems.

---

**Q11. What is the Circular Dependency Problem, and how do you resolve it?**

A circular dependency occurs when Bean A depends on Bean B, and Bean B depends on Bean A (directly or through a chain). With **constructor injection**, this causes Spring to throw a `BeanCurrentlyInCreationException` at startup, because neither bean can be fully constructed first.

**Ways to resolve it:**
1. **Refactor the design** (best option) — circular dependencies usually indicate poor separation of concerns; extract shared logic into a third bean that both depend on
2. **Use `@Lazy`** on one of the injected dependencies — defers resolution until first use, breaking the startup-time cycle
3. **Switch to setter/field injection** for one side — Spring can construct both beans first, then wire the circular reference afterward (works because setter injection happens after construction)
4. **Use `ApplicationContext` directly** to fetch the bean lazily inside a method (generally discouraged — hides the dependency)

```java
// Option: breaking cycle with @Lazy
@Component
public class ServiceA {
    private final ServiceB serviceB;
    public ServiceA(@Lazy ServiceB serviceB) { this.serviceB = serviceB; }
}
```

**Interview tip:** always mention that the *first* fix should be reconsidering the design — `@Lazy` is a workaround, not a solution to the underlying coupling problem.

---

## Extra / Important Interview Questions

**Q12. What happens if two beans of the same type exist and neither `@Primary` nor `@Qualifier` is used?**

Spring throws a `NoUniqueBeanDefinitionException` at startup — it can't decide which bean to inject and refuses to guess.

---

**Q13. Can a prototype-scoped bean be injected into a singleton bean? What's the catch?**

Yes, but by default the prototype bean is only created **once** — at the time the singleton is created — because the singleton is created only once and its dependencies are resolved then. To get a fresh prototype instance on every use, you need:
- **`@Lookup` method injection**, or
- Injecting an `ObjectFactory<T>` / `Provider<T>` and calling `.getObject()` / `.get()` each time you need a new instance

This is a common interview "gotcha" question.

---

**Q14. What's the difference between `@Resource` and `@Autowired`?**

- `@Autowired` (Spring-specific) — matches **by type** first, then by name if needed
- `@Resource` (JSR-250/Jakarta standard) — matches **by name** first, then by type

`@Resource` is less commonly used today but appears in codebases favoring standard Java annotations over Spring-specific ones.

---

**Q15. Does `@PostConstruct` run before or after the constructor?**

After. The order is: **constructor runs → dependencies injected (via setters/fields, or already passed via constructor) → `@PostConstruct` runs**. This is important because `@PostConstruct` is the correct place to use injected dependencies for initialization logic — they aren't guaranteed to be available inside the constructor itself if using setter/field injection.

---

**Q16. If a class isn't annotated with `@Component` (or a stereotype), can it still be a bean?**

Yes — via an explicit `@Bean` method inside a `@Configuration` class. This is exactly how you register beans for third-party classes you don't own and can't annotate directly (e.g., `RestTemplate`, `ObjectMapper`, `DataSource`).

---

**Q17. Why is `@Primary` sometimes considered risky in larger codebases?**

Because it's a "silent" default — someone reading an `@Autowired` field has no indication of which bean gets injected without hunting down which one is marked `@Primary`. In large codebases, being explicit with `@Qualifier` at each injection point is often preferred for clarity, even though it's more verbose.

---

**Q18. What is the difference between `@Component` scanning and manually declaring a bean with `@Bean`?**

- `@Component` (+ stereotypes) — Spring scans the classpath and auto-detects annotated classes; you own the class source code
- `@Bean` — you manually declare the bean creation logic inside a `@Configuration` class; used when you don't own the class, need conditional logic, or want fine-grained control over instantiation

---

**Q19. Can `@PreDestroy` be relied upon in all shutdown scenarios (e.g., `kill -9`)?**

No. `@PreDestroy` only fires on a **graceful shutdown** of the Spring context (normal JVM shutdown hook, `context.close()`). If the process is forcibly killed (`kill -9`), the JVM terminates immediately without running shutdown hooks, so `@PreDestroy` logic is skipped. This is worth mentioning for questions about resource cleanup reliability.

---

**Q20. How would you debug a `NoSuchBeanDefinitionException`?**

Checklist to walk through in an interview:
1. Is the class annotated with `@Component`/`@Service`/etc., or registered via `@Bean`?
2. Is it within the package scanned by `@ComponentScan` (default: same package as `@SpringBootApplication` and below)?
3. Is there a missing `@Import` if the config lives in a different module?
4. Is the bean conditional (`@ConditionalOnProperty`, `@Profile`) and the condition isn't met?
5. Is the bean actually being requested by the correct type/name?

---

*Interview tip: Questions on beans often chain together — e.g., "explain `@Autowired`" → "what if there are two beans?" → "how do you fix it?" → "what's the risk of `@Primary`?" Be ready to go a level deeper on any answer.*