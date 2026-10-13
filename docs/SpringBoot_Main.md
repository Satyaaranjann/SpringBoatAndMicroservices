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