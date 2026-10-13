# Core Spring & Spring Boot Basics — Q&A

---

**Q1. What is Spring Framework?**

Spring is a lightweight, open-source Java framework used to build enterprise applications. Its core is built around **Inversion of Control (IoC)** and **Dependency Injection (DI)**, which help decouple application components. On top of that core, Spring provides modules for data access (Spring JDBC/ORM), web applications (Spring MVC), security (Spring Security), messaging, and more — all designed to work together but usable independently.

---

**Q2. What is Spring Boot?**

Spring Boot is a project built on top of the Spring Framework that makes it fast and easy to create stand-alone, production-grade Spring applications. It removes the need for extensive manual configuration by providing:
- **Auto-configuration** — sensible defaults based on what's on the classpath
- **Starter dependencies** — curated dependency bundles
- **Embedded servers** — no need to deploy a WAR to an external server
- **Production-ready features** — health checks, metrics via Actuator

In short: Spring Boot lets you go from zero to a running application in minutes, instead of hours of XML/Java config.

---

**Q3. Spring vs Spring Boot vs Spring MVC — what's the difference?**

| | Spring Framework | Spring MVC | Spring Boot |
|---|---|---|---|
| What it is | The core framework (IoC, DI, AOP, etc.) | A module of Spring for building web apps (Model-View-Controller) | A tool built on top of Spring to simplify setup and configuration |
| Configuration | Manual (XML or Java config) | Manual (DispatcherServlet, view resolvers, etc.) | Auto-configured |
| Server | You deploy to an external server | Same as Spring | Comes with an embedded server |
| Use case | Foundation for everything else | Building web/REST applications | Rapidly bootstrapping any Spring-based app |

Think of it as layers: **Spring** is the foundation, **Spring MVC** is a specific module built on that foundation for web apps, and **Spring Boot** is a convention-over-configuration layer that makes using Spring (and Spring MVC) far less painful.

---

**Q4. What is Inversion of Control (IoC)?**

IoC is a design principle where the control of creating and managing object dependencies is transferred from the application code to a container/framework. Instead of a class creating its own dependencies with `new`, the framework "injects" them. This inverts the traditional flow of control — hence the name — and results in loosely coupled, more testable code.

*Example without IoC:* `UserService` creates its own `UserRepository` instance internally.
*Example with IoC:* Spring creates the `UserRepository` bean and hands it to `UserService`.

---

**Q5. What is Dependency Injection (DI)?**

DI is the actual mechanism through which IoC is implemented in Spring. Instead of an object looking up or constructing its dependencies, they are "injected" into it — typically via constructor, setter, or field. Spring supports three types:
1. **Constructor Injection** (recommended) — dependencies passed via constructor, enables immutability
2. **Setter Injection** — dependencies set via setter methods, useful for optional dependencies
3. **Field Injection** — `@Autowired` directly on a field (discouraged — harder to test, hides dependencies)

---

**Q6. What is the Spring IoC Container?**

The IoC Container is the core of Spring — it's responsible for instantiating, configuring, and managing the lifecycle of beans. It reads configuration metadata (annotations, Java config, or XML), creates the objects (beans), wires their dependencies together, and manages them until the application shuts down. The two main container types are `BeanFactory` and `ApplicationContext`.

---

**Q7. ApplicationContext vs BeanFactory — what's the difference?**

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| Bean instantiation | Lazy (on-demand) | Eager (at startup, by default) |
| Internationalization support | No | Yes |
| Event publishing | No | Yes |
| AOP support | Limited | Full support |
| Typical usage | Rarely used directly today | Standard container used in almost all Spring apps |

In practice, `ApplicationContext` is what you use — it's a superset of `BeanFactory` with enterprise features built in. `BeanFactory` is mostly kept for lightweight/legacy scenarios.

---

**Q8. What are Spring Boot Starters?**

Starters are pre-packaged sets of dependency descriptors that pull in a curated, version-compatible group of libraries for a specific purpose, so you don't have to hunt down and match versions manually. Examples:
- `spring-boot-starter-web` → Spring MVC + embedded Tomcat + Jackson
- `spring-boot-starter-data-jpa` → Hibernate + Spring Data JPA + JDBC
- `spring-boot-starter-security` → Spring Security
- `spring-boot-starter-test` → JUnit, Mockito, AssertJ

Adding one starter to your `pom.xml`/`build.gradle` is enough to get a fully working, compatible dependency set.

---

**Q9. What is Auto-Configuration in Spring Boot?**

Auto-Configuration is Spring Boot's mechanism for automatically configuring beans based on:
1. What's present on the classpath (e.g., if `H2` is on the classpath, it auto-configures an in-memory `DataSource`)
2. What beans already exist in the context (it backs off if you've defined your own)
3. Property values in `application.properties`/`application.yml`

It's driven by `@EnableAutoConfiguration` (bundled inside `@SpringBootApplication`) and implemented via classes registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, each guarded by conditional annotations like `@ConditionalOnClass` and `@ConditionalOnMissingBean`.

---

**Q10. What does `@SpringBootApplication` do?**

It's a convenience annotation that bundles three annotations into one:
- **`@Configuration`** — marks the class as a source of bean definitions
- **`@EnableAutoConfiguration`** — turns on Spring Boot's auto-configuration mechanism
- **`@ComponentScan`** — scans the current package and sub-packages for Spring components (`@Component`, `@Service`, etc.)

It's typically placed on the main class that contains the `main()` method, and its package location matters — component scanning starts from there downward, so the main class is conventionally placed at the project's root package.

---

**Q11. What is the typical Spring Boot Project Structure?**

```
src/
 └── main/
     ├── java/
     │    └── com/example/app/
     │         ├── AppApplication.java      (main class)
     │         ├── controller/              (REST endpoints)
     │         ├── service/                 (business logic)
     │         ├── repository/              (data access)
     │         ├── model/ or entity/        (domain objects)
     │         ├── dto/                     (data transfer objects)
     │         ├── config/                  (configuration classes)
     │         └── exception/               (custom exceptions, handlers)
     └── resources/
          ├── application.yml
          ├── static/                       (CSS, JS, images — for MVC apps)
          └── templates/                    (Thymeleaf/JSP views, if used)
src/test/java/...                            (mirrors main structure for tests)
```

This layered structure (controller → service → repository) keeps concerns separated and is the de-facto convention across most Spring Boot codebases.

---

**Q12. What happens internally when `SpringApplication.run()` is called?**

At a high level:
1. **Determine application type** — Servlet (web), Reactive, or none — based on classpath
2. **Load `ApplicationContextInitializers` and `ApplicationListeners`** registered via `spring.factories`/auto-configuration imports
3. **Create the `ApplicationContext`** appropriate to the app type (e.g., `AnnotationConfigServletWebServerApplicationContext`)
4. **Prepare the environment** — load property sources, active profiles
5. **Run auto-configuration** — apply all matching `@Configuration` classes based on conditions
6. **Register and instantiate beans** — component scanning, dependency injection
7. **Start the embedded server** (if it's a web app) — binds to the configured port
8. **Run `CommandLineRunner` / `ApplicationRunner` beans** — post-startup hooks
9. **Publish `ApplicationReadyEvent`** — application is now ready to serve requests

# Spring Boot `SpringApplication.run()` — Internal Call Flow

```text
main()
  │
  ▼
SpringApplication.run(MyApplication.class, args)
  │
  ├── new SpringApplication(...)
  │      │
  │      ├── deduceWebApplicationType()
  │      ├── getBootstrapRegistryInitializers()
  │      ├── getApplicationContextFactory()
  │      ├── getApplicationListeners()
  │      └── getSpringFactoriesInstances(...)
  │
  ▼
SpringApplication.run(String... args)
  │
  ├── StopWatch.start()
  │
  ├── createBootstrapContext()
  │
  ├── configureHeadlessProperty()
  │
  ├── getRunListeners(args)
  │       └── SpringApplicationRunListeners
  │
  ├── listeners.starting(...)
  │
  ├── prepareEnvironment(...)
  │       │
  │       ├── createEnvironment()
  │       ├── configureEnvironment()
  │       ├── ConfigurationPropertySources.attach(...)
  │       ├── getPropertySources()
  │       └── EnvironmentPostProcessor
  │
  ├── createApplicationContext()
  │       │
  │       └── AnnotationConfigServletWebServerApplicationContext
  │
  ├── prepareContext(...)
  │       │
  │       ├── context.setEnvironment()
  │       ├── postProcessApplicationContext()
  │       ├── applyInitializers()
  │       ├── listeners.contextPrepared(...)
  │       ├── load(...)
  │       │    └── register @Configuration class
  │       └── listeners.contextLoaded(...)
  │
  ├── refreshContext(context)
  │       │
  │       └── AbstractApplicationContext.refresh()
  │              │
  │              ├── prepareRefresh()
  │              ├── obtainFreshBeanFactory()
  │              ├── prepareBeanFactory()
  │              ├── postProcessBeanFactory()
  │              ├── invokeBeanFactoryPostProcessors()
  │              │      │
  │              │      ├── ConfigurationClassPostProcessor
  │              │      ├── @ComponentScan
  │              │      ├── @Bean
  │              │      ├── @Import
  │              │      └── AutoConfiguration
  │              │
  │              ├── registerBeanPostProcessors()
  │              ├── initMessageSource()
  │              ├── initApplicationEventMulticaster()
  │              ├── onRefresh()
  │              │      └── start embedded Tomcat
  │              │
  │              ├── registerListeners()
  │              ├── finishBeanFactoryInitialization()
  │              │      └── instantiate singleton beans
  │              │
  │              └── finishRefresh()
  │
  ├── afterRefresh(...)
  │
  ├── listeners.started(...)
  │
  ├── callRunners(...)
  │       ├── ApplicationRunner
  │       └── CommandLineRunner
  │
  └── listeners.ready(...)
         │
         ▼
   ApplicationReadyEvent
```
# Spring Boot `SpringApplication.run()` — Simplified Internal Flow

```text
SpringApplication.run()
        │
        ▼
Create SpringApplication
        │
        ▼
Determine WebApplicationType
        │
        ▼
Create Environment
        │
        ▼
Load application.properties / yaml
        │
        ▼
Create ApplicationContext
        │
        ▼
Apply ApplicationContextInitializers
        │
        ▼
Load @SpringBootApplication
        │
        ▼
context.refresh()
        │
        ├── Component Scan
        ├── Auto Configuration
        ├── Bean Definitions
        ├── BeanFactoryPostProcessors
        ├── BeanPostProcessors
        ├── Dependency Injection
        ├── Singleton Creation
        └── Embedded Tomcat
        │
        ▼
ApplicationRunner / CommandLineRunner
        │
        ▼
ApplicationReadyEvent
        │
        ▼
Application is READY
```

---

**Q13. What are Embedded Servers, and why does Spring Boot use them?**

Traditionally, a Java web app was packaged as a WAR file and deployed to an external servlet container (like a standalone Tomcat installation). Spring Boot flips this: it **embeds** the server (Tomcat by default) directly inside the application as a dependency, so the app itself is a runnable, self-contained JAR.

- **Tomcat** — default, servlet-based, most widely used
- **Jetty** — lightweight alternative, sometimes preferred for lower memory footprint
- **Undertow** — high-performance, non-blocking, often used with WebFlux

Benefits: no separate server installation/configuration, consistent runtime environment across dev/prod, and the app can be run with a single `java -jar app.jar` command — which is a major enabler for containerization (Docker) and cloud-native deployment.

---
# Spring Boot Main Class — Line-by-Line Explanation

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringBoatAndMicroservicesApplication {

    public static void main(String[] args) {

        SpringApplication.run(SpringBoatAndMicroservicesApplication.class, args);

    }
}
```

---

## Line 1–2: Imports

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
```

- **`SpringApplication`** — a utility class that bootstraps and launches a Spring application. It sets up the Spring `ApplicationContext`, starts the embedded server (Tomcat by default), and wires everything together.
- **`SpringBootApplication`** — an annotation that marks this class as the entry point and enables Spring Boot's auto-configuration magic.

---

## Line 3: `@SpringBootApplication`

This single annotation is actually a **combination of three annotations**:

| Annotation | What It Does |
|---|---|
| `@Configuration` | Marks this class as a source of Spring bean definitions — it can contain `@Bean` methods |
| `@EnableAutoConfiguration` | Tells Spring Boot to automatically configure beans based on what's on the classpath (e.g., if `spring-boot-starter-web` is present, it auto-configures Tomcat, DispatcherServlet, Jackson, etc.) |
| `@ComponentScan` | Tells Spring to scan this package and all sub-packages for components (`@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`) and register them as beans |

This is exactly the "convention over configuration" magic that makes Spring Boot easier than plain Spring — one annotation replaces dozens of lines of manual XML/Java config.

---

## Line 4: Class Declaration

```java
public class SpringBoatAndMicroservicesApplication {
```

A plain public Java class. By convention, its name matches the project name and it typically sits in the **root package** of your application (e.g., `com.company.springbootandmicroservices`) — this matters because `@ComponentScan` only scans this package and everything below it. If a `@Service` or `@Controller` class lives in a sibling/parent package, it won't be picked up.

---

## Line 5: The `main` Method

```java
public static void main(String[] args) {
```

The standard Java entry point — the JVM looks for this exact signature (`public static void main(String[] args)`) to start any Java application. This is not Spring-specific; it's plain Java. `args` holds any command-line arguments passed when running the JAR (e.g., `java -jar app.jar --server.port=9090`).

---

## Line 6: Bootstrapping Spring

```java
SpringApplication.run(SpringBoatAndMicroservicesApplication.class, args);
```

This single line does a lot of work under the hood:

1. Creates a `SpringApplication` instance
2. Determines the application type (servlet web app, reactive web app, or none) based on classpath dependencies
3. Creates the `ApplicationContext` (Spring's IoC container)
4. Runs all `@EnableAutoConfiguration` logic to configure beans automatically
5. Performs component scanning (finds and registers all `@Component`, `@Service`, `@Repository`, `@Controller` classes)
6. Starts the embedded web server (Tomcat/Jetty/Undertow) if it's a web application
7. Publishes application startup events

**Parameters:**
- `SpringBoatAndMicroservicesApplication.class` — tells Spring which class is the primary configuration source (needed to determine the base package for component scanning)
- `args` — forwards any command-line arguments into the Spring application context, so they can be read as configuration properties

---

## Line 7–8: Closing Braces

End of `main` method, end of class.

---

## What Actually Happens When You Run This

```
java -jar app.jar
        │
        ▼
main() is invoked by the JVM
        │
        ▼
SpringApplication.run() bootstraps Spring
        │
        ├─→ Creates ApplicationContext
        ├─→ Auto-configures beans (DB, web, security, etc. based on classpath)
        ├─→ Component-scans for @Controller, @Service, @Repository beans
        ├─→ Starts embedded Tomcat server
        └─→ App is now running and ready to accept requests
```

---

## Why This Is Enough to Start a Whole Application

This ~8-line class, combined with the right starter dependencies in `pom.xml` (like `spring-boot-starter-web`), is genuinely all you need to have a fully working REST API — no XML, no manual servlet configuration, no manually starting a server. This is the "less boilerplate" benefit of Spring Boot compared to plain Spring Framework, where the same setup would require significant manual configuration.
## Extra / Bonus Questions

**Q14. Can you disable a specific auto-configuration? How?**

Yes — via the `exclude` attribute:
```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```
Or in `application.properties`:
```
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```
Useful when you want to manage a particular concern (e.g., a custom `DataSource`) yourself.

---

**Q15. What is the difference between `@ComponentScan` default behavior and specifying `basePackages`?**

By default, `@ComponentScan` (via `@SpringBootApplication`) scans the package of the annotated class and all sub-packages. If your beans live outside that package tree, you must explicitly specify:
```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.example.app", "com.external.lib"})
```
Missing this is a common cause of "bean not found" errors when beans live in a separate module/package.

---

**Q16. What is the difference between `spring-boot-starter-parent` and `spring-boot-dependencies`?**

- `spring-boot-starter-parent` — a Maven **parent POM** that provides dependency management, plugin configuration, default Java version, and resource filtering. Your project inherits from it directly.
- `spring-boot-dependencies` — a **BOM (Bill of Materials)** you import (via `<dependencyManagement>`) when your project already has a different parent POM and can't inherit from `spring-boot-starter-parent`, but still wants Spring Boot's version-managed dependencies.

---

**Q17. Why is constructor injection generally preferred over field injection?**

- Enables `final` fields → immutability and thread safety
- Makes dependencies explicit and visible in the constructor signature
- Fails fast at startup if a required dependency is missing (vs. a runtime NPE)
- Easier to unit test — you can instantiate the class with `new` and pass mocks directly, without needing a Spring context or reflection-based injection

---

**Q18. Can a Spring Boot application run without an embedded web server?**

Yes. If the app doesn't need to serve HTTP requests (e.g., a batch job or CLI tool), Spring Boot detects the absence of web-related classes/dependencies (or you explicitly set `spring.main.web-application-type=none`) and skips starting an embedded server entirely — the app runs, executes its logic (often via a `CommandLineRunner`), and exits.

---

**Q19. What is the difference between `@RestController` and using `@Controller` + `@ResponseBody`?**

They're functionally equivalent — `@RestController` is a convenience annotation that combines `@Controller` and `@ResponseBody`, so every handler method's return value is written directly to the HTTP response body (e.g., as JSON) instead of being resolved as a view name.

---

**Q20. How does Spring Boot decide which embedded server to use if multiple are on the classpath?**

It follows classpath precedence — if `spring-boot-starter-web` is used (which defaults to Tomcat) and you haven't excluded it, Tomcat wins. To switch, you exclude the default and add the alternative starter, e.g.:
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
  <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

---

*Tip: In interviews, after a definition-style answer, be ready to follow up with a one-line "why it matters" — e.g., "constructor injection is preferred because it makes testing easier and dependencies explicit," not just "it's the recommended approach."*

# Spring Boot Main Class: Line-by-Line Internals

```java
@SpringBootApplication
@RequiredArgsConstructor
@EnableConfigurationProperties(JwtProperties.class)
public class BhandarOrbitApplication {

    public static void main(String[] args) {
        SpringApplication.run(
            BhandarOrbitApplication.class,
            args
        );
    }
}
```

---

## 1. `@SpringBootApplication`

A **meta-annotation**, shorthand for three annotations:

| Annotation | What it does |
|---|---|
| `@SpringBootConfiguration` | A specialized `@Configuration`. Marks this class as a source of bean definitions. |
| `@EnableAutoConfiguration` | Tells Spring Boot to configure beans automatically based on what is on the classpath. |
| `@ComponentScan` | Scans this class's package and all sub-packages for `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController` and `@Configuration` classes. |

### Internals

**Component scan**
- The base package is the package of `BhandarOrbitApplication`.
- A `@Service` outside that package tree will not be found. This is why the main class normally sits in the root package.

**Auto-configuration**
- Boot reads a list of candidate config classes from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` inside the Boot jars.
- Each candidate is guarded by conditions:
    - `@ConditionalOnClass`: is Tomcat or Hibernate on the classpath?
    - `@ConditionalOnMissingBean`: did the developer already define one?
    - `@ConditionalOnProperty`: is a property set?
- If `spring-boot-starter-web` is present, you get `DispatcherServlet`, embedded Tomcat and Jackson.
- If `spring-boot-starter-data-jpa` is present, you get `DataSource`, `EntityManagerFactory` and so on.
- Your own beans always win because of `@ConditionalOnMissingBean`.

---

## 2. `@RequiredArgsConstructor`

This is **Lombok**, not Spring. At compile time, Lombok's annotation processor modifies the class bytecode and generates a constructor for all `final` fields and `@NonNull` fields.

It is normally used on `@Service` or `@Controller` classes for constructor injection.

> **On this main class it does nothing useful.** There are no `final` fields, so it only generates an empty constructor. It can be safely deleted along with the Lombok import.

---

## 3. `@EnableConfigurationProperties(JwtProperties.class)`

Internally it does two things:

1. It `@Import`s `EnableConfigurationPropertiesRegistrar`, which **registers `JwtProperties` as a Spring bean**.
2. It activates `ConfigurationPropertiesBindingPostProcessor`, which binds values from `application.yml`, `application.properties`, environment variables or command-line args onto that bean's fields.

Example `JwtProperties`:

```java
@ConfigurationProperties(prefix = "jwt")
public class JwtProperties {
    private String secret;
    private long expirationMs;
    // getters/setters (or a record with constructor binding)
}
```

With this config:

```properties
jwt.secret=abc
jwt.expiration-ms=3600000
```

the values are bound automatically. Relaxed binding maps `expiration-ms` to `expirationMs`. You can then inject `JwtProperties` anywhere.

> **Gotcha:** do not also annotate `JwtProperties` with `@Component`. It would be registered twice. Alternatively, `@ConfigurationPropertiesScan` can register all such classes automatically.

---

## 4. `public class BhandarOrbitApplication`

A plain Java class. Its roles are:
- To hold the annotations above (it becomes the *primary source* of configuration).
- To hold `main`.

---

## 5. `public static void main(String[] args)`

The standard JVM entry point. When you run `java -jar app.jar`, the JVM calls this method.

`args` carries command-line arguments such as `--server.port=9090`, which Spring turns into properties.

---

## 6. `SpringApplication.run(BhandarOrbitApplication.class, args)`

This is where all the work happens, in order:

1. **Create a `SpringApplication`** with your class as the primary source.
2. **Deduce the application type** from the classpath: Servlet (Spring MVC), Reactive (WebFlux) or None.
3. **Load factories and listeners:** `ApplicationContextInitializer`s and `ApplicationListener`s are read from `META-INF/spring.factories`.
4. **Prepare the `Environment`:** merge property sources in priority order (command-line args, env vars, `application.yml`, defaults) and activate profiles.
5. **Print the banner** (the Spring ASCII art).
6. **Create the `ApplicationContext`.** For a servlet app this is `AnnotationConfigServletWebServerApplicationContext`.
7. **Prepare the context:** register your main class as a bean definition and apply the initializers.
8. **`refresh()`**, the heart of Spring:
    - `ConfigurationClassPostProcessor` parses your annotations, runs the component scan, processes `@Import`s and loads the auto-configuration candidates. This is where sections 1 and 3 above actually take effect.
    - `BeanPostProcessor`s are registered (they handle `@Autowired`, `@Transactional`, AOP proxies and so on).
    - **`onRefresh()` creates the embedded web server** (Tomcat by default).
    - All singleton beans are instantiated, with dependencies injected and lifecycle callbacks like `@PostConstruct` run.
    - **`finishRefresh()` starts Tomcat**, which begins listening on the port.
9. **Call runners:** any `CommandLineRunner` or `ApplicationRunner` beans execute.
10. **Publish `ApplicationReadyEvent`.** The app is now live and `run()` returns the `ConfigurableApplicationContext`.

---

## Summary Flow

```
main() -> SpringApplication.run()
   -> build Environment (props/profiles)
   -> create ApplicationContext
   -> scan + auto-configure + register beans (refresh)
   -> instantiate & wire beans
   -> start embedded Tomcat
   -> run runners -> app ready
```

---

## Interview Quick Answers

- **What does `@SpringBootApplication` combine?** `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`.
- **How does auto-configuration decide what to create?** Conditional annotations evaluated against the classpath, existing beans and properties.
- **Why must the main class be in the root package?** `@ComponentScan` defaults to the main class's package and its sub-packages.
- **Where is the embedded server created and started?** Created in `onRefresh()`, started in `finishRefresh()`.
- **Why `@EnableConfigurationProperties` instead of `@Component` on the properties class?** It registers the class as a bean and enables binding without double registration.

# Internal Call Flow of `SpringApplication.run()`

This is the Spring Boot 3.x call sequence, starting from:

```java
SpringApplication.run(BhandarOrbitApplication.class, args);
```

---

## Phase 1: Static `run()` and Constructor

```
SpringApplication.run(BhandarOrbitApplication.class, args)
 └─ new SpringApplication(primarySources).run(args)

    new SpringApplication(...)
     ├─ WebApplicationType.deduceFromClasspath()       // SERVLET / REACTIVE / NONE
     ├─ load BootstrapRegistryInitializers             // from spring.factories
     ├─ load ApplicationContextInitializers            // from spring.factories
     ├─ load ApplicationListeners                      // from spring.factories
     └─ deduceMainApplicationClass()                   // walks the stack trace for main()
```

---

## Phase 2: Instance `run(args)`

```
run(args)
 ├─ createBootstrapContext()
 ├─ getRunListeners(args)                  // EventPublishingRunListener
 ├─ listeners.starting()                   // ApplicationStartingEvent
 ├─ prepareEnvironment()
 │    ├─ getOrCreateEnvironment()          // ApplicationServletEnvironment
 │    ├─ configureEnvironment()            // args + profiles
 │    ├─ listeners.environmentPrepared()   // ApplicationEnvironmentPreparedEvent
 │    │    └─ EnvironmentPostProcessorApplicationListener
 │    │         └─ ConfigDataEnvironmentPostProcessor   // loads application.yml/.properties
 │    └─ bindToSpringApplication()
 ├─ printBanner()
 ├─ createApplicationContext()             // AnnotationConfigServletWebServerApplicationContext
 ├─ prepareContext()
 │    ├─ context.setEnvironment()
 │    ├─ applyInitializers()               // ApplicationContextInitializers run
 │    ├─ listeners.contextPrepared()
 │    ├─ load()                            // BeanDefinitionLoader registers your main class
 │    └─ listeners.contextLoaded()
 ├─ refreshContext()  ──────────────►  context.refresh()   (Phase 3)
 ├─ afterRefresh()
 ├─ listeners.started()                    // ApplicationStartedEvent
 ├─ callRunners()                          // ApplicationRunner, then CommandLineRunner
 └─ listeners.ready()                      // ApplicationReadyEvent, app is live
```

---

## Phase 3: `AbstractApplicationContext.refresh()`

```
refresh()
 ├─ 1.  prepareRefresh()                   // init property sources, validate required props
 ├─ 2.  obtainFreshBeanFactory()           // DefaultListableBeanFactory
 ├─ 3.  prepareBeanFactory()               // classloader, SpEL, Aware processors
 ├─ 4.  postProcessBeanFactory()           // servlet scopes (request/session)
 ├─ 5.  invokeBeanFactoryPostProcessors()  // where most of the magic happens
 │        └─ ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
 │             └─ ConfigurationClassParser.parse()
 │                  ├─ @ComponentScan     → ClassPathBeanDefinitionScanner.doScan()
 │                  ├─ @Import            → EnableConfigurationPropertiesRegistrar
 │                  │                       (registers JwtProperties)
 │                  ├─ @Bean methods      → collected
 │                  └─ AutoConfigurationImportSelector   // deferred, processed LAST
 │                       └─ reads AutoConfiguration.imports
 │                          → filters with @Conditional*
 │             └─ loadBeanDefinitions()    // all definitions registered
 ├─ 6.  registerBeanPostProcessors()       // Autowired, Transactional, AOP, ConfigProps binder
 ├─ 7.  initMessageSource()
 ├─ 8.  initApplicationEventMulticaster()
 ├─ 9.  onRefresh()
 │        └─ createWebServer()
 │             └─ TomcatServletWebServerFactory.getWebServer()   // Tomcat built, not yet serving
 ├─ 10. registerListeners()
 ├─ 11. finishBeanFactoryInitialization()
 │        └─ preInstantiateSingletons()    // see Phase 4
 └─ 12. finishRefresh()
          ├─ publish ContextRefreshedEvent
          └─ WebServerStartStopLifecycle.start()
               └─ webServer.start()         // Tomcat connector begins accepting requests
```

---

## Phase 4: Creating Each Singleton Bean

```
preInstantiateSingletons()
 └─ getBean(name)
     └─ doGetBean()
         └─ createBean()
             └─ doCreateBean()
                 ├─ createBeanInstance()        // constructor injection happens here
                 ├─ populateBean()              // field/setter @Autowired
                 └─ initializeBean()
                      ├─ invokeAwareMethods()
                      ├─ postProcessBeforeInitialization()   // @PostConstruct
                      ├─ afterPropertiesSet() / init-method
                      └─ postProcessAfterInitialization()    // AOP/@Transactional proxies created
```

---

## Key Points

- **Order matters:** component scan and auto-config run in step 5, the web server object is built in step 9, beans are created in step 11, and Tomcat only starts listening in step 12. No request can arrive before all beans are ready.
- **`JwtProperties`** is registered in step 5 (by the `@Import` registrar) and bound to your config by `ConfigurationPropertiesBindingPostProcessor`.
- **Auto-configuration is deferred** so your own `@Bean` definitions are already registered when `@ConditionalOnMissingBean` is evaluated.
- **Proxies** (`@Transactional`, `@Async`, `@Cacheable`) are created in the last stage of bean initialization, so you inject the proxy, not the raw object.

---

## Interview Quick Answers

- **Where does component scanning happen?** In `ConfigurationClassPostProcessor`, during `invokeBeanFactoryPostProcessors()`.
- **When is the embedded server created vs started?** Created in `onRefresh()`, started in `finishRefresh()`.
- **Why is auto-configuration processed last?** `AutoConfigurationImportSelector` is a `DeferredImportSelector`, so user-defined beans take priority.
- **Where does `@PostConstruct` run?** In `postProcessBeforeInitialization()` via `CommonAnnotationBeanPostProcessor`.
- **When do runners execute?** After the context refresh, before `ApplicationReadyEvent`.