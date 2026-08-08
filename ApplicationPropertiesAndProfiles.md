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