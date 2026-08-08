# Data Access with Spring Data JPA — Q&A

---

**Q1. JPA vs Hibernate vs Spring Data JPA — what's the difference?**

These three are often confused because they're layered on top of each other:

| | JPA | Hibernate | Spring Data JPA |
|---|---|---|---|
| What it is | A **specification** (interfaces/annotations defined by Jakarta EE) | An **implementation** of the JPA specification | An **abstraction layer** on top of JPA |
| Provides | `@Entity`, `EntityManager`, JPQL — the contract | Actual ORM engine — SQL generation, caching, dirty checking | Repository interfaces, derived query methods, pagination |
| Can it run alone? | No — needs an implementation | Yes — can be used directly without JPA (native Hibernate API) | No — needs a JPA provider (Hibernate) underneath |
| Analogy | An interface | A class implementing that interface | A convenience wrapper reducing boilerplate around the interface |

**In practice:** Spring Boot apps use Spring Data JPA (the convenience layer) → which uses JPA (the spec) → implemented by Hibernate (the engine that actually talks to the database). You rarely write raw Hibernate or `EntityManager` code directly in a typical Spring Boot CRUD app — Spring Data JPA repositories handle it for you.

---

**Q2. What is Entity Mapping? Explain `@Entity`, `@Table`, `@Id`, `@GeneratedValue`.**

Entity mapping is how a Java class is mapped to a database table so JPA/Hibernate can persist and retrieve it as rows.

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true)
    private String email;
}
```

- **`@Entity`** — marks the class as a JPA-managed entity, meaning it will map to a database table. Requires a no-arg constructor and a primary key field.
- **`@Table(name = "...")`** — optional; specifies the actual table name. If omitted, JPA defaults to the class name. Used when the table name differs from the class name or needs schema qualification.
- **`@Id`** — marks the field as the primary key.
- **`@GeneratedValue`** — specifies how the primary key is generated. Strategies:
    - `IDENTITY` — DB auto-increment column (common with MySQL)
    - `SEQUENCE` — uses a DB sequence object (common with PostgreSQL/Oracle)
    - `AUTO` — lets the JPA provider pick the strategy based on the DB dialect
    - `TABLE` — uses a separate table to simulate sequences (rarely used, slower)

`@Column` is also frequently paired here to control column name, nullability, uniqueness, and length constraints at the schema level.

---

**Q3. `JpaRepository` vs `CrudRepository` — what's the difference?**

Both are part of the Spring Data repository hierarchy, and `JpaRepository` extends the chain built on `CrudRepository`:

```
Repository (marker interface)
  → CrudRepository<T, ID>        — basic CRUD: save, findById, findAll, delete, count
      → PagingAndSortingRepository<T, ID>   — adds pagination and sorting
          → JpaRepository<T, ID>            — adds JPA-specific extras
```

- **`CrudRepository`** — basic create/read/update/delete methods only
- **`JpaRepository`** — extends `PagingAndSortingRepository`, adding:
    - Batch operations (`saveAll`, `deleteAllInBatch`)
    - `flush()` and `saveAndFlush()` for explicit persistence context control
    - Returns `List<T>` instead of `Iterable<T>` for `findAll()` — more convenient
    - JPA-specific bonus methods like `getReferenceById()`

**In practice:** almost every Spring Boot project uses `JpaRepository` — `CrudRepository` is mainly useful if you want to keep your repository persistence-technology-agnostic (e.g., if you might swap to a non-JPA store later).

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```
This single line gives you a fully working CRUD data access layer with zero implementation code.

---

**Q4. What are Derived Query Methods?**

Spring Data JPA can generate the query implementation automatically just from the **method name**, parsing keywords like `findBy`, `And`, `Or`, `Between`, `OrderBy`, etc.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    List<User> findByName(String name);
    List<User> findByNameAndAge(String name, int age);
    List<User> findByAgeGreaterThan(int age);
    List<User> findByEmailContaining(String keyword);
    List<User> findByNameOrderByAgeDesc(String name);
    boolean existsByEmail(String email);
    long countByAge(int age);
    Optional<User> findFirstByOrderByCreatedAtDesc();
}
```

Spring parses the method signature at startup, validates it against the entity's fields, and generates the corresponding JPQL query — no implementation needed. This works well for simple-to-moderate queries; for anything complex (joins, subqueries, conditional logic), `@Query` is a better fit.

---

**Q5. What is `@Query`, and what is JPQL?**

**JPQL (Java Persistence Query Language)** is an object-oriented query language — similar to SQL, but it operates on **entities and their fields**, not database tables and columns directly.

**`@Query`** lets you write an explicit query (JPQL by default) on a repository method, useful when derived query method names would get too long or can't express the logic needed.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailCustom(@Param("email") String email);

    @Query("SELECT u FROM User u JOIN u.orders o WHERE o.status = :status")
    List<User> findUsersWithOrderStatus(@Param("status") String status);

    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
    int deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);
}
```

Note: `SELECT u FROM User u` refers to the **entity name** `User`, not the table name `users` — a common point of confusion for JPQL beginners. `@Modifying` is required for `UPDATE`/`DELETE` queries, and they need to run within a transaction.

---

**Q6. What are Native Queries, and when should you use them over JPQL?**

Native queries let you write **raw SQL** directly, executed as-is against the database, bypassing JPQL's entity abstraction.

```java
@Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
Optional<User> findByEmailNative(@Param("email") String email);
```

**When to use native queries over JPQL:**
- Database-specific functions (e.g., PostgreSQL's `JSONB` operators, full-text search) not expressible in JPQL
- Complex queries where JPQL's object-graph traversal is unwieldy or inefficient
- Performance-critical queries needing hand-tuned SQL (specific index hints, window functions)

**Trade-off:** native queries are tied to a specific database dialect (less portable), and don't automatically map to entities as cleanly — you often need `@SqlResultSetMapping` or a projection interface/DTO for complex result shapes.

---

**Q7. How do Pagination and Sorting work in Spring Data JPA?**

Instead of manually writing `LIMIT`/`OFFSET` SQL, Spring Data JPA provides `Pageable` and `Sort` abstractions that plug directly into repository methods.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByActive(boolean active, Pageable pageable);
}
```

```java
@GetMapping("/users")
public Page<User> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {

    Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
    return userRepository.findByActive(true, pageable);
}
```

- **`Pageable`** — encapsulates page number, page size, and sort order
- **`Page<T>`** — the result wrapper, includes content **plus metadata**: `totalElements`, `totalPages`, `isFirst`, `isLast`
- **`Slice<T>`** — a lighter alternative to `Page` that doesn't run a `COUNT` query — useful when you only need "is there a next page?" without the total count (better performance for infinite-scroll UIs)

Spring MVC can also auto-resolve `Pageable` straight from request parameters (`?page=0&size=10&sort=name,asc`) if the controller method parameter is typed as `Pageable` directly — no manual construction needed.

---

**Q8. What are Entity Relationships? Explain `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`.**

These annotations map foreign-key relationships between tables onto object references between entities.

**`@ManyToOne`** — many rows in this table reference one row in another (most common, and usually the **owning side**):
```java
@Entity
public class Order {
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
}
```

**`@OneToMany`** — the inverse side of `@ManyToOne`; one user has many orders:
```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

**`@OneToOne`** — one-to-one relationship, e.g., a `User` has exactly one `UserProfile`:
```java
@Entity
public class User {
    @OneToOne
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
}
```

**`@ManyToMany`** — requires a join table, e.g., students and courses:
```java
@Entity
public class Student {
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private List<Course> courses;
}
```

---

**Q9. What are `mappedBy` and the Owning Side of a relationship?**

In a **bidirectional** relationship, both entities reference each other, but only one side actually controls the foreign key column in the database — that's the **owning side**. The other side is the **inverse (mapped) side**, marked with `mappedBy` to tell JPA "don't create a separate column/table for this — the relationship is already managed by the other entity's field."

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")   // inverse side — "user" refers to the field name in Order
    private List<Order> orders;
}

@Entity
public class Order {
    @ManyToOne
    @JoinColumn(name = "user_id")   // owning side — this actually creates the FK column
    private User user;
}
```

**Rule of thumb:** the side with `@JoinColumn` (or the join table definition in `@ManyToMany`) is the owning side; the side with `mappedBy` is the inverse side. Getting this backwards is a classic source of confusing bugs — like updates to the "wrong" side silently not persisting, since JPA only looks at the owning side to determine what SQL to generate.

---

**Q10. FetchType: LAZY vs EAGER — what's the difference?**

Controls **when** related entities are loaded from the database relative to the parent entity:

- **`EAGER`** — related entity/collection is loaded immediately, in the same query (or an immediate follow-up query) as the parent
- **`LAZY`** — related entity/collection is loaded **only when accessed** (a proxy is returned initially, and the actual query fires on first access)

| Relationship | Default FetchType |
|---|---|
| `@ManyToOne` | EAGER |
| `@OneToOne` | EAGER |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

```java
@ManyToOne(fetch = FetchType.LAZY)   // commonly overridden to LAZY even though default is EAGER
@JoinColumn(name = "user_id")
private User user;
```

**Best practice:** default almost everything to `LAZY` explicitly, and fetch what you need intentionally (via `JOIN FETCH` or `@EntityGraph`) rather than relying on `EAGER`, which can silently pull in large object graphs and cause performance issues.

**Common pitfall:** accessing a `LAZY` association **outside** an active persistence context/transaction (e.g., after the controller has returned the entity and Hibernate's session is closed) throws `LazyInitializationException`.

---

**Q11. What is the N+1 Select Problem, and how do you fix it?**

Occurs when fetching a list of **N** parent entities triggers **1** query for the parents, plus **N additional queries** — one per parent — to fetch each one's lazily-loaded related entities. Total: N+1 queries instead of ideally 1 or 2.

**Example of the problem:**
```java
List<User> users = userRepository.findAll();    // 1 query
for (User user : users) {
    user.getOrders().size();                     // N queries — one per user, triggered lazily
}
```

**Fixes:**
1. **`JOIN FETCH` in JPQL** — forces the related entities to load in the same query:
   ```java
   @Query("SELECT u FROM User u JOIN FETCH u.orders")
   List<User> findAllWithOrders();
   ```
2. **`@EntityGraph`** — declarative alternative to `JOIN FETCH` (see Q12)
3. **Batch fetching** — `@BatchSize(size = 20)` on the association, so Hibernate loads related entities in batches (e.g., `WHERE user_id IN (...)`) instead of one query per parent
4. Enable Hibernate's `hibernate.default_batch_fetch_size` property globally

This is one of the **most frequently asked** Spring Data JPA interview questions — expect a follow-up asking you to demonstrate the fix in code, not just describe it.

---

**Q12. What is `@EntityGraph`?**

A declarative way to specify which related entities/associations should be eagerly fetched **for a specific query**, without changing the entity's default `FetchType` globally (and without hand-writing `JOIN FETCH` JPQL).

```java
public interface UserRepository extends JpaRepository<User, Long> {

    @EntityGraph(attributePaths = {"orders", "profile"})
    List<User> findAll();

    @EntityGraph(attributePaths = "orders")
    Optional<User> findWithOrdersById(Long id);
}
```

**Why prefer it over always setting `FetchType.EAGER`:** `@EntityGraph` lets you fetch eagerly **only for queries that need it**, keeping the entity's default as `LAZY` (safe, performant) everywhere else — giving you per-query control instead of an all-or-nothing global setting.

---

**Q13. How does Auditing work with `@CreatedDate` and `@LastModifiedDate`?**

Spring Data JPA can automatically populate audit fields (creation/modification timestamps, and optionally who created/modified a record) without manual code.

```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class Auditable {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}

@Entity
public class User extends Auditable {
    // inherits createdAt, updatedAt, createdBy, updatedBy
}
```

**Setup requirements:**
1. `@EnableJpaAuditing` on a `@Configuration` class
2. `@EntityListeners(AuditingEntityListener.class)` on the entity (or a shared base class)
3. For `@CreatedBy`/`@LastModifiedBy`, you must also provide an `AuditorAware` bean telling Spring how to determine the "current user" (e.g., from the Spring Security context)

```java
@Bean
public AuditorAware<String> auditorProvider() {
    return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                          .map(Authentication::getName);
}
```

---

**Q14. What are Database Migrations, and how do Flyway and Liquibase work?**

Database migration tools manage **schema changes** (creating tables, adding columns, altering constraints) in a versioned, repeatable, team-shareable way — instead of relying on Hibernate's `ddl-auto: update` (which is fine for local dev, but risky/unpredictable for production).

**Flyway:**
- Migrations are plain **SQL files**, named with a version prefix: `V1__create_users_table.sql`, `V2__add_email_column.sql`
- Flyway tracks which migrations have run (in a `flyway_schema_history` table) and applies only new ones, in order, on startup
- Simple, SQL-first, easy to reason about

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE
);
```

**Liquibase:**
- Migrations can be written in **XML, YAML, JSON, or SQL** (more format flexibility)
- Supports "changesets" with rollback definitions built in
- Better suited for teams needing database-agnostic definitions or fine-grained rollback control

**Why use either over `ddl-auto: update`:**
- `ddl-auto: update` is unpredictable in production — it can silently make destructive or unintended schema changes and offers no history/versioning
- Migrations give you a reviewable, version-controlled history of every schema change, the ability to roll back, and consistent schema state across dev/staging/prod and every team member's local environment

**Standard practice:** `ddl-auto=validate` (or `none`) in production, with Flyway/Liquibase owning all schema changes — Hibernate only validates the entity mappings match the actual schema, it never modifies it.

---

## Extra / Important Interview Questions

**Q15. Why is `ddl-auto=update` discouraged in production?**

It can generate unintended or destructive changes (e.g., dropping a column it thinks is unused, altering types incompatibly), has no versioning or rollback mechanism, and behaves differently across Hibernate versions/dialects — making schema state unpredictable and hard to audit across environments.

---

**Q16. What is the difference between `Page<T>` and `Slice<T>`?**

`Page<T>` runs an additional `COUNT(*)` query to compute `totalElements`/`totalPages`. `Slice<T>` skips that count query and only knows whether there's a next page (by fetching `size + 1` records internally) — cheaper when you don't need total counts, common in infinite-scroll UIs.

---

**Q17. Can you use both JPQL derived methods and native queries in the same repository?**

Yes — a single `JpaRepository` interface can mix derived query methods, `@Query` JPQL methods, and `@Query(nativeQuery = true)` methods freely; Spring Data JPA handles each independently based on the method's annotations.

---

**Q18. What's the difference between `orphanRemoval = true` and `CascadeType.REMOVE`?**

- `CascadeType.REMOVE` — deleting the parent also deletes associated children
- `orphanRemoval = true` — additionally deletes a child **the moment it's removed from the parent's collection**, even if the parent itself isn't deleted (e.g., `user.getOrders().remove(order)` triggers a delete on `order`)

```java
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Order> orders;
```

---

**Q19. Why might Hibernate execute an unexpected `SELECT` before an `INSERT` when you call `save()`?**

Because Spring Data JPA's `save()` method has to first determine whether the entity is **new** (needs `INSERT`) or **existing** (needs `UPDATE`). If the ID generation strategy doesn't clearly indicate "this is a new record" (common with manually-assigned IDs, or `@Id` without `@GeneratedValue`), Hibernate issues a `SELECT` to check existence first. This is avoidable by implementing `Persistable<ID>` on the entity to explicitly signal "new" state, or by using a proper auto-generation strategy.

---

**Q20. If you change `FetchType` from LAZY to EAGER to "fix" a `LazyInitializationException," what's the risk?**

It fixes the symptom but not the root cause, and introduces a new problem: every query that loads the parent entity now **always** loads the association too — even in code paths that never needed it — potentially causing unnecessary joins, larger result sets, and its own N+1-style performance issues at scale. The better fix is almost always to keep `LAZY` and explicitly fetch what's needed per-query (`JOIN FETCH`, `@EntityGraph`), or ensure the access happens within an open transaction/session (e.g., via `@Transactional` on the service method).

---

*Interview tip: JPA questions are frequently asked as "explain X, then debug this broken code" pairs — e.g., "what's LAZY vs EAGER" followed by "here's a LazyInitializationException stack trace, what's wrong and how do you fix it." Practice explaining concepts AND diagnosing the resulting bugs, not just definitions.*