# 14-Day Spring Boot Interview Preparation Plan

## Goal

Prepare for Java/Spring Boot interviews in 14 days through focused study, hands-on coding, and mock interviews.

- **Duration:** 14 days
- **Daily commitment:** 3–4 hours
- **Target roles:** Java Developer, Spring Boot Developer, Backend Engineer, Technical Lead
- **Main project:** Build an inventory-management REST API using Java, Spring Boot, Spring Security, JWT, MySQL, JPA, validation, and testing.

---

## 1. The 14-Day Roadmap

### Days 1–2: Java and Spring Core

**Study**
- OOP, collections, generics, exceptions, streams, and lambdas
- Multithreading basics, `equals()` and `hashCode()`
- IoC, dependency injection, beans, bean scopes, and bean lifecycle
- Constructor injection and component scanning
- `@Component`, `@Service`, `@Repository`, and `@Configuration`

**Practice**
- Write small Java exercises using collections and streams.
- Create a simple Spring application with constructor injection.
- Explain how Spring creates and injects objects.

**Deliverable:** Explain dependency injection and the Spring bean lifecycle clearly.

### Days 3–4: Spring Boot and REST APIs

**Study**
- Spring Boot auto-configuration, starters, and embedded Tomcat
- `@SpringBootApplication`, configuration properties, and profiles
- `@RestController`, request mappings, request/response DTOs
- HTTP methods, status codes, pagination, and sorting
- Global exception handling with `@ControllerAdvice`

**Practice**
- Build CRUD REST APIs.
- Validate request bodies.
- Return appropriate HTTP status codes.
- Add consistent error responses.

**Deliverable:** A working CRUD API.

### Days 5–6: JPA, Hibernate, and SQL

**Study**
- Entity mapping, repositories, JPQL, and native queries
- One-to-one, one-to-many, and many-to-many relationships
- Lazy vs. eager loading and the N+1 query problem
- Transactions, isolation, indexes, joins, and grouping
- `@Transactional`, rollback rules, and optimistic locking

**Practice**
- Connect the application to MySQL.
- Implement entity relationships and repository queries.
- Write SQL queries for joins, aggregates, and duplicate records.
- Explain transaction boundaries and rollback behavior.

**Deliverable:** Persist and query data using MySQL and Spring Data JPA.

### Days 7–8: Spring Security and JWT

**Study**
- Authentication vs. authorization
- Password hashing and JWT access-token flow
- Security filter chain and request filters
- Roles, authorities, and method-level security
- CORS, CSRF, and HTTP 401 vs. 403

**Practice**
- Implement login and JWT generation/validation.
- Secure endpoints.
- Restrict endpoints by role.
- Test unauthenticated and unauthorised requests.

**Deliverable:** Secure APIs with role-based access control.

### Days 9–10: Testing, Async Processing, and Microservices

**Study**
- JUnit 5, Mockito, unit tests, and integration tests
- `@WebMvcTest` vs. `@SpringBootTest`
- Synchronous vs. asynchronous processing
- `@Async`, application events, and messaging fundamentals
- Service discovery, API gateway, timeouts, retries, and circuit breakers

**Practice**
- Unit-test service methods with Mockito.
- Test controller endpoints.
- Test repository behavior against a test database.
- Explain how to handle a downstream service failure.

**Deliverable:** A tested application and a clear explanation of distributed-system failure handling.

### Days 11–12: Production Readiness and System Design

**Study**
- Spring Boot Actuator and health checks
- Logging and monitoring
- Caching with Spring Cache and Redis concepts
- Connection pools, JVM memory, and thread pools
- Idempotency, rate limiting, and API versioning
- Docker, environment configuration, and CI/CD basics

**Practice**
- Add health endpoints and structured logging.
- Investigate a slow API.
- Design an inventory-management backend.
- Explain how to prevent duplicate sale or payment requests.

**Deliverable:** Explain production troubleshooting and scalable backend architecture.

### Days 13–14: Revision and Mock Interviews

**Study and practice**
- Revise frequently asked questions.
- Solve Java coding and SQL problems.
- Explain your project architecture end-to-end.
- Practise debugging and scenario-based questions.
- Complete two timed mock interviews.

**Deliverable:** Answer questions clearly without depending on notes.

---

## 2. Daily Study Timetable

| Time | Activity | What to do |
|---|---|---|
| 45 minutes | Theory | Understand concepts and prepare short interview answers. |
| 90 minutes | Hands-on coding | Implement the day's concepts in your Spring Boot project. |
| 45 minutes | Interview questions | Answer 10–15 questions aloud without notes. |
| 30 minutes | Revision | Review mistakes, write notes, and revise yesterday's concepts. |

If you only have two hours on a particular day, prioritise coding and interview practice over watching tutorials.

---

## 3. Build One Interview-Ready Project

### Project: Inventory Management REST API

**Suggested stack**
- Java 21
- Spring Boot
- MySQL
- Spring Data JPA and Hibernate
- Spring Security and JWT
- Maven
- JUnit 5 and Mockito
- Postman
- Docker (optional)

### Architecture

```text
Client / Postman
       |
       | HTTP requests with JSON
       v
Controller Layer
  - REST endpoints
  - DTOs
  - Request validation
       |
       v
Service Layer
  - Business logic
  - Transactions
  - Authorization checks
       |
       v
Repository Layer
  - Spring Data JPA
  - Hibernate
  - SQL queries
       |
       v
MySQL Database
  - Users
  - Products
  - Inventory
  - Purchases
  - Sales
```

### Required features

- User registration and login with JWT.
- Product CRUD APIs with validation and pagination.
- Role-based access for admin, stock manager, and sales staff.
- Purchase and sales transactions that update stock safely.
- Global exception handling and consistent API responses.
- Unit and integration tests.
- Logging, health checks, and externalised configuration.

### Critical interview exercise

**Scenario:** Two users attempt to sell the last item at the same time. How do you prevent stock from becoming negative?

Prepare to explain:
- Database transactions
- Optimistic vs. pessimistic locking
- Atomic stock updates
- Validation and rollback behavior
- How to test concurrent requests

---

## 4. Twenty Essential Interview Questions

Use the checklist to track whether you can answer each question without notes. For every question, prepare:
1. A definition you can explain in about 30 seconds.
2. A practical example from your project.
3. A follow-up answer explaining trade-offs.

### Spring Core and Spring Boot

- [ ] What are IoC and dependency injection?
- [ ] How does Spring Boot auto-configuration work?
- [ ] What happens internally when a Spring Boot application starts?
- [ ] What is the difference between `@Component`, `@Service`, and `@Repository`?
- [ ] How do you manage environment-specific configuration?

### REST and Database

- [ ] What is the difference between PUT and PATCH?
- [ ] How do you implement global exception handling?
- [ ] What is the N+1 query problem, and how do you solve it?
- [ ] Explain lazy loading vs. eager loading.
- [ ] How does `@Transactional` work, and why can self-invocation cause problems?

### Security and Testing

- [ ] Explain JWT authentication from login to API access.
- [ ] What is the difference between authentication and authorization?
- [ ] How do you implement role-based access control?
- [ ] What is the difference between `@Mock` and `@InjectMocks`?
- [ ] When would you use `@WebMvcTest` vs. `@SpringBootTest`?

### Production and System Design

- [ ] How do you troubleshoot a slow REST API?
- [ ] How would you prevent duplicate payment or sale requests?
- [ ] What happens when a downstream microservice is unavailable?
- [ ] How would you design a scalable inventory API?
- [ ] How do you investigate high CPU usage or memory leaks in a Java service?

---

## 5. Java Coding and SQL Practice

Spend at least 20–30 minutes daily on coding. Spring Boot interviews often test Java fundamentals as well as framework knowledge.

### Java coding exercises

- [ ] Find duplicate elements in a list.
- [ ] Count character frequency using streams.
- [ ] Sort employees by salary.
- [ ] Find the second-highest number.
- [ ] Use `Optional` correctly.
- [ ] Explain `HashMap` internals.
- [ ] Compare `ArrayList` and `LinkedList`.
- [ ] Compare `HashMap` and `ConcurrentHashMap`.
- [ ] Explain checked vs. unchecked exceptions.
- [ ] Compare `synchronized` and locks.
- [ ] Explain `CompletableFuture`.

### SQL exercises

- [ ] Find the second-highest salary.
- [ ] Find duplicate records.
- [ ] Join employees and departments.
- [ ] Use `GROUP BY` with `HAVING`.
- [ ] Find the top N records per group.
- [ ] Add indexes and analyse query plans.

---

## 6. Official Learning Resources

Use official documentation as a reference. Avoid switching between too many courses during these 14 days.

- **Spring Boot:** https://docs.spring.io/spring-boot/reference/
    - Focus on auto-configuration, configuration, web applications, and production features.
- **Spring Security:** https://docs.spring.io/spring-security/reference/
    - Focus on authentication, authorisation, and the security filter chain.
- **Spring Data JPA:** https://docs.spring.io/spring-data/jpa/reference/
    - Focus on repositories, queries, entity relationships, and database access.
- **JUnit 5:** https://junit.org/junit5/docs/current/user-guide/
    - Focus on unit tests, assertions, and lifecycle methods.

---

## 7. Final Interview Readiness Checklist

- [ ] I can build a REST API without following a tutorial.
- [ ] I can explain Spring Boot startup and dependency injection.
- [ ] I can write JPA queries and explain transaction boundaries.
- [ ] I can secure endpoints with JWT and roles.
- [ ] I can write Mockito unit tests and integration tests.
- [ ] I can solve Java coding and SQL questions under time pressure.
- [ ] I can explain my project architecture and debug a production issue.
- [ ] I have completed two mock interviews.

---

## Final Recommendation

Prioritise **Spring Core, REST APIs, JPA/SQL, Spring Security, testing, and project explanation**. Learn the fundamentals deeply before spending too much time on advanced microservices.

### Personalise this plan before starting

Decide:
- **Target role:** Java/Spring Boot Developer, Senior Developer/Technical Lead, or Microservices/Backend Architect.
- **Current experience:** Beginner with mostly Java experience, some REST API experience, or experienced with a real project.
- **Daily study time:** 2 hours, 3–4 hours, or 5+ hours.

Use these choices to adjust the amount of coding, system design, and mock interview practice.
