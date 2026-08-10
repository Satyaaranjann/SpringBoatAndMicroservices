# Spring Boot Fundamentals

1 . ## [Core Spring & Spring Boot Basics](SpringBootBasic.md)

* What is Spring Framework
* What is Spring Boot
* Spring vs Spring Boot vs Spring MVC
* Inversion of Control (IoC)
* Dependency Injection (DI)
* Spring IoC Container
* ApplicationContext vs BeanFactory
* Spring Boot Starters
* Auto-Configuration
* `@SpringBootApplication` Annotation
* Spring Boot Project Structure
* `SpringApplication.run()` Internals
* Embedded Servers (Tomcat, Jetty, Undertow)

2 . ## [Beans & Configuration](BeansAndConfiguration.md)

* What is a Bean
* Bean Lifecycle
* Bean Scopes (Singleton, Prototype, Request, Session)
* `@Component`, `@Service`, `@Repository`, `@Controller`
* `@Bean` and `@Configuration`
* `@Autowired`
* `@Qualifier` and `@Primary`
* Constructor vs Setter vs Field Injection
* `@PostConstruct` and `@PreDestroy`
* `@Lazy` Initialization
* Circular Dependency Problem

3 . ## [Application Properties & Profiles](ApplicationPropertiesAndProfiles.md)

* `application.properties` vs `application.yml`
* Externalized Configuration
* `@Value` Annotation
* `@ConfigurationProperties`
* Spring Profiles
* `@Profile` Annotation
* Environment-Specific Configuration
* Property Precedence Order

4 . ## [REST API Development](RESTAPIDevelopment.md)

* `@RestController` vs `@Controller`
* `@RequestMapping` and HTTP Method Mappings
* `@PathVariable`
* `@RequestParam`
* `@RequestBody` and `@ResponseBody`
* `ResponseEntity`
* HTTP Status Codes
* Content Negotiation
* API Versioning Strategies
* Building CRUD REST APIs

5 . ## [Validation & Exception Handling](ValidationAndExceptionHandling.md)

* Bean Validation (`@Valid`, `@NotNull`, `@Size`, `@Pattern`)
* Custom Validators
* `@ControllerAdvice` and `@RestControllerAdvice`
* `@ExceptionHandler`
* Custom Exception Classes
* Global Error Response Structure

6 . ## [Data Access with Spring Data JPA](DataAccesswithSpringDataJPA.md)

* JPA vs Hibernate vs Spring Data JPA
* Entity Mapping (`@Entity`, `@Table`, `@Id`, `@GeneratedValue`)
* `JpaRepository` and `CrudRepository`
* Derived Query Methods
* `@Query` and JPQL
* Native Queries
* Pagination and Sorting
* Entity Relationships (`@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`)
* `mappedBy` and Owning Side
* FetchType: LAZY vs EAGER
* N+1 Select Problem
* `@EntityGraph`
* Auditing (`@CreatedDate`, `@LastModifiedDate`)
* Database Migrations (Flyway, Liquibase)

7 . ## [Transactions](Transactions.md)

* `@Transactional`
* Transaction Propagation
* Transaction Isolation Levels
* Rollback Rules
* Programmatic vs Declarative Transactions

8 . ## [Spring Security](SpringSecurity.md)

* Authentication vs Authorization
* Spring Security Filter Chain
* `SecurityFilterChain` Configuration
* In-Memory vs Database Authentication
* Password Encoding (`BCryptPasswordEncoder`)
* JWT Authentication
* OAuth2 and OpenID Connect
* Role-Based Access Control
* `@PreAuthorize` and `@Secured`
* CORS Configuration
* CSRF Protection

9 . ## [Caching](Caching.md)

* Spring Cache Abstraction
* `@EnableCaching`
* `@Cacheable`, `@CachePut`, `@CacheEvict`
* Redis Integration
* Cache Eviction Strategies

10 . ## [Asynchronous & Scheduled Tasks](AsynchronousAndScheduledTasks.md)

* `@EnableAsync` and `@Async`
* `CompletableFuture` with Async Methods
* `@EnableScheduling` and `@Scheduled`
* Cron Expressions
* Thread Pool Configuration

11 . ## [Testing](Testing.md)

* Unit Testing with JUnit 5
* Mockito Basics
* `@SpringBootTest`
* `@WebMvcTest`
* `@DataJpaTest`
* MockMvc
* Testcontainers
* Integration Testing Strategies
* Test Coverage Tools

12 . ## [Microservices with Spring Boot](MicroserviceswithSpringBoot.md)

* Monolith vs Microservices
* Service Communication (REST, Feign)
* `RestTemplate` vs `WebClient`
* Service Discovery (Eureka)
* Spring Cloud Config
* API Gateway (Spring Cloud Gateway)
* Load Balancing
* Circuit Breaker (Resilience4j)
* Retry and Rate Limiting
* Distributed Tracing (Sleuth, Zipkin)
* Event-Driven Architecture
* Kafka Integration
* RabbitMQ Integration
* Saga Pattern
* Outbox Pattern

13 . ## [Reactive Programming](ReactiveProgramming.md)

* Spring WebFlux Basics
* Mono and Flux
* Reactive Streams
* WebClient for Reactive Calls
* R2DBC for Reactive Database Access
* Blocking vs Non-Blocking I/O

14 . ## [Actuator & Monitoring](ActuatorAndMonitoring.md)

* Spring Boot Actuator
* Health Checks (`/actuator/health`)
* Metrics (`/actuator/metrics`)
* Custom Actuator Endpoints
* Micrometer
* Prometheus and Grafana Integration
* Application Logging (SLF4J, Logback)
* Structured/JSON Logging

15 . ## [Build, Packaging & Deployment](BuildPackagingDeployment.md)

* Maven vs Gradle for Spring Boot
* Executable JAR vs WAR
* Layered JARs
* Dockerizing a Spring Boot Application
* Multi-Stage Docker Builds
* Kubernetes Basics (Deployments, Services, ConfigMaps)
* CI/CD Pipelines (GitHub Actions, Jenkins)
* Environment Variables and Secrets Management

16 . ## [Advanced Topics](AdvancedTopics.md)

* Spring AOP (Aspect-Oriented Programming)
* Custom Annotations
* Spring Batch
* GraphQL with Spring for GraphQL
* Multi-Module Project Structuring
* Domain-Driven Design in Spring Boot
* Hexagonal / Clean Architecture
* Performance Tuning and Connection Pooling (HikariCP)

17 . ## [Annotations — Complete Guide & Spring Boot Usage](Annotations.md)

18 . ## [MySQL](MySql.md)

19 . ## [Top MNC Interview Question Answer](TopMNCInterviewQuestionAnswer.md)

20 . ## [Miscellaneous](Miscellaneous.md)
---

**Reference format inspired by:** [Java Fundamentals](https://github.com/Satyaaranjann/JavaDocs/blob/main/JavaFundamentals.md)