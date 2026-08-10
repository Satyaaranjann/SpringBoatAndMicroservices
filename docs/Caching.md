# Caching — Q&A

---

**Q1. What is the Spring Cache Abstraction?**

Spring's Cache Abstraction is a layer that lets you add caching to your application **declaratively** (via annotations) without coupling your code to a specific caching provider. You write `@Cacheable`, `@CachePut`, `@CacheEvict` on methods, and Spring transparently intercepts calls to store/retrieve results — the actual cache implementation underneath (in-memory `ConcurrentHashMap`, Redis, Caffeine, EhCache, Hazelcast) is swappable without touching business logic.

```java
@Service
public class ProductService {

    @Cacheable("products")
    public Product getProductById(Long id) {
        // expensive DB call — only runs on a cache miss
        return productRepository.findById(id).orElseThrow();
    }
}
```

**Why it matters:** it decouples "what to cache and when" (your annotations) from "how caching is actually implemented" (the provider), so switching from a simple in-memory cache in dev to Redis in production requires **zero changes to service code** — only configuration.

---

**Q2. What does `@EnableCaching` do?**

It's the switch that activates Spring's caching infrastructure — without it, `@Cacheable`/`@CachePut`/`@CacheEvict` annotations are silently ignored (a very common "why isn't my cache working" interview/debugging question).

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("products", "users");
    }
}
```

Under the hood, `@EnableCaching` triggers Spring to register the AOP infrastructure (proxies, interceptors) needed to intercept calls to cache-annotated methods — conceptually similar to how `@EnableTransactionManagement` or `@Transactional` requires proxying to work.

---

**Q3. What do `@Cacheable`, `@CachePut`, and `@CacheEvict` do? How do they differ?**

| Annotation | Behavior | Typical use |
|---|---|---|
| **`@Cacheable`** | Checks the cache first; if present (**cache hit**), returns the cached value **without executing the method**. If absent (**cache miss**), executes the method and stores the result. | Read operations (GET-style) |
| **`@CachePut`** | **Always executes** the method, then updates the cache with the result — never skips execution. | Write/update operations where you want the cache refreshed |
| **`@CacheEvict`** | Removes one or more entries from the cache; doesn't affect method execution otherwise. | Delete operations, or forcing a cache refresh |

```java
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    return productRepository.findById(id).orElseThrow();
}

@CachePut(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepository.save(product);   // always runs, then updates cache
}

@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    productRepository.deleteById(id);
}

@CacheEvict(value = "products", allEntries = true)
public void clearAllProducts() {
    productRepository.deleteAll();
}
```

**Key distinction interviewers probe:** `@Cacheable` can **skip** method execution entirely on a hit; `@CachePut` **never** skips execution — it's specifically for keeping the cache in sync after a write, not for avoiding work.

---

**Q4. How does Redis Integration work with Spring Boot caching?**

Redis is an in-memory, distributed key-value store commonly used as a caching layer because it's fast, supports expiration natively, and — unlike a local in-memory cache (`ConcurrentHashMap`) — is **shared across multiple application instances**, which matters for horizontally-scaled deployments.

**Setup:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
spring:
  cache:
    type: redis
  data:
    redis:
      host: localhost
      port: 6379
```

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
}
```

Once configured, the **exact same** `@Cacheable`/`@CachePut`/`@CacheEvict` annotations work unchanged — only the `CacheManager` implementation switches from local memory to Redis. This is the core value proposition of the abstraction: business code doesn't know or care which backend is used.

**Local cache (e.g., `ConcurrentMapCacheManager`) vs Redis — when to use which:**
- **Local/in-memory** — simplest, fastest (no network hop), but **not shared** across instances — each app instance has its own separate cache, risking stale/inconsistent data in a multi-instance deployment
- **Redis** — shared across all instances, supports TTL natively, survives app restarts (data persists in Redis, not the JVM) — the standard choice for any production, horizontally-scaled Spring Boot service

---

**Q5. What are common Cache Eviction Strategies?**

Eviction determines **when and how** entries are removed from the cache — either because they're stale, or because the cache is full and needs space.

**1. Time-based expiration (TTL — Time To Live):**
Entries automatically expire after a fixed duration, regardless of usage. Simple, predictable, but can serve stale data right up until expiry, or evict still-useful data unnecessarily.
```java
.entryTtl(Duration.ofMinutes(10))
```

**2. Manual/explicit eviction (`@CacheEvict`):**
Entries are removed explicitly when the underlying data changes (e.g., after an update/delete), keeping the cache consistent with the source of truth rather than relying purely on time.

**3. Size-based eviction policies** (when the cache has a maximum capacity):
- **LRU (Least Recently Used)** — evicts the entry that hasn't been accessed for the longest time; assumes recently-used data is likely to be used again
- **LFU (Least Frequently Used)** — evicts the entry with the fewest total accesses; better for workloads with a stable "hot set" of frequently accessed items
- **FIFO (First In, First Out)** — evicts the oldest-inserted entry regardless of usage; simplest but least "smart"

**4. Write-through vs Write-behind (for caches backed by a database):**
- **Write-through** — every write goes to the cache **and** the database synchronously, keeping them always in sync (higher write latency, strong consistency)
- **Write-behind (write-back)** — writes go to the cache immediately and are asynchronously flushed to the database later (lower write latency, risk of data loss if the cache fails before flushing)

**Choosing a strategy in practice:** combine TTL (as a safety net against staleness) with explicit `@CacheEvict` on writes (for immediate consistency) — this is the most common real-world pattern, rather than relying on just one mechanism alone.

---

## Extra / Important Interview Questions

**Q6. What happens if `@Cacheable` is applied to a method, but that method is called from within the same class ("self-invocation")?**

Just like `@Transactional`, `@Cacheable` relies on **Spring AOP proxies**. Calling a `@Cacheable` method internally via `this.method()` bypasses the proxy entirely — the caching logic is silently skipped, and the method executes fresh every time, even on what should be a "hit." Same fix as the transactional case: move the method to a separate bean, or inject a self-reference.

---

**Q7. Why is choosing the right cache `key` critical, and what happens with methods that have multiple parameters?**

By default, Spring generates a cache key from **all** method parameters combined. If a method takes multiple parameters, you often need to specify the key explicitly using SpEL, or entries with different parameter combinations that should logically share a cache entry won't hit correctly:

```java
@Cacheable(value = "orders", key = "#userId + '-' + #status")
public List<Order> getOrders(Long userId, String status) { ... }
```

Getting the key wrong is a common source of subtle bugs — e.g., using only part of the relevant parameters as the key can cause **incorrect** cache hits (returning data for the wrong combination of inputs).

---

**Q8. How would you conditionally cache a method result — e.g., only cache if the result isn't null or empty?**

Use the `condition` and `unless` attributes:
```java
@Cacheable(value = "products", key = "#id", unless = "#result == null")
public Product getProduct(Long id) {
    return productRepository.findById(id).orElse(null);
}

@Cacheable(value = "products", condition = "#id > 0")
public Product getProduct(Long id) { ... }
```
- **`condition`** — evaluated **before** method execution; if false, caching is skipped entirely (method still runs normally)
- **`unless`** — evaluated **after** method execution, based on the result; if true, the result is **not cached**, even though it was computed

This distinction (before vs after execution) is a good "do you actually understand SpEL here, or just copy-pasting annotations" interview probe.

---

**Q9. What's the risk of caching data that changes frequently, or caching without any eviction strategy at all?**

**No eviction strategy** → unbounded cache growth, risking memory exhaustion (`OutOfMemoryError`) in a local/in-memory cache, since entries are never removed. **Caching volatile/frequently-changing data** without a short enough TTL or explicit eviction on write → serving **stale data** to users, which can range from a minor UX issue (an outdated product price) to a serious bug (showing an outdated account balance). The general principle: cache data that's expensive to compute/fetch **and** doesn't change often relative to how frequently it's read — caching highly volatile data usually isn't worth the consistency risk.

---

**Q10. In a multi-instance (horizontally scaled) deployment, why can a local in-memory cache cause data inconsistency?**

Each application instance has its **own separate** local cache (e.g., `ConcurrentMapCacheManager`) — there's no coordination between instances. If Instance A evicts/updates its local cache entry after a write, Instance B's local cache still holds the **old** value until its own TTL expires or it's separately evicted — meaning different users hitting different instances (behind a load balancer) can see inconsistent data simultaneously. This is precisely why **Redis** (a single shared, external cache) is the standard choice for scaled deployments — all instances read/write the same cache, eliminating this inconsistency.

---

**Q11. Can you use multiple cache providers in the same Spring Boot application (e.g., local cache for one feature, Redis for another)?**

Yes — you can define multiple `CacheManager` beans and explicitly designate which cache names route to which manager (e.g., via `@Primary` on the default, or a custom `CacheResolver`), or use different caching annotations pointed at different named caches backed by different managers. This is less common but useful when, for example, a very hot, small dataset benefits from ultra-fast local caching, while larger or shared datasets go through Redis.

---

*Interview tip: Caching questions frequently pair a concept with "what goes wrong if you get this wrong" — stale data, self-invocation silently no-op'ing, unbounded memory growth, cross-instance inconsistency. Leading with the failure mode alongside the definition demonstrates production experience, not just annotation syntax.*