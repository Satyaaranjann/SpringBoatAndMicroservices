# Transactions — Q&A

---

**Q1. What is `@Transactional`?**

`@Transactional` marks a method (or class) so that Spring wraps its execution in a database transaction — all database operations inside it either **all succeed and commit**, or **all fail and roll back** together, preserving data consistency (ACID).

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryService.reduceStock(order.getItems());
        paymentService.charge(order.getPaymentDetails());
        // if any of the above throws a RuntimeException, ALL changes roll back
    }
}
```

**How it works under the hood:** Spring uses a **proxy** (JDK dynamic proxy or CGLIB) wrapped around the bean. When a `@Transactional` method is called, the proxy intercepts the call, starts a transaction, invokes the real method, then commits or rolls back based on the outcome, before returning control to the caller.

**Important gotcha:** because it relies on a proxy, `@Transactional` **does not work** when a method calls another `@Transactional` method **on the same class internally** (`this.otherMethod()`) — the call bypasses the proxy entirely, so no transaction boundary is applied to the inner call. This is one of the most commonly tested "gotcha" interview questions.

---

**Q2. What is Transaction Propagation?**

Propagation defines **how a transactional method behaves when it's called from within another transaction** — does it join the existing one, suspend it, or always start fresh?

| Propagation | Behavior |
|---|---|
| **`REQUIRED`** (default) | Joins the existing transaction if one exists; otherwise starts a new one |
| **`REQUIRES_NEW`** | Always starts a new, independent transaction — suspends the existing one if present |
| **`NESTED`** | Starts a nested transaction (savepoint) within the existing one — can roll back independently without affecting the outer transaction |
| **`SUPPORTS`** | Joins the transaction if one exists; runs non-transactionally if not |
| **`NOT_SUPPORTED`** | Suspends any existing transaction and runs non-transactionally |
| **`MANDATORY`** | Requires an existing transaction — throws an exception if none exists |
| **`NEVER`** | Throws an exception if a transaction exists — must run non-transactionally |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logAuditEvent(String event) {
    auditRepository.save(new AuditLog(event));
    // commits independently, even if the calling transaction later rolls back
}
```

**Classic use case for `REQUIRES_NEW`:** audit logging — you want the audit record saved **even if** the main business transaction later fails and rolls back, so the failure itself gets logged.

---

**Q3. What are Transaction Isolation Levels?**

Isolation levels control **how much one transaction can "see" of another transaction's uncommitted or concurrent changes** — a trade-off between data consistency and concurrency/performance.

| Isolation Level | Prevents | Allows |
|---|---|---|
| **`READ_UNCOMMITTED`** | Nothing | Dirty reads, non-repeatable reads, phantom reads |
| **`READ_COMMITTED`** | Dirty reads | Non-repeatable reads, phantom reads |
| **`REPEATABLE_READ`** | Dirty reads, non-repeatable reads | Phantom reads |
| **`SERIALIZABLE`** | Everything (fully isolated, transactions behave as if executed sequentially) | Nothing — highest consistency, lowest concurrency |

**The three read anomalies, explained:**
- **Dirty read** — reading another transaction's **uncommitted** changes (which might later be rolled back)
- **Non-repeatable read** — reading the same row twice within a transaction and getting **different values**, because another transaction updated and committed it in between
- **Phantom read** — re-running the same query twice within a transaction and getting a **different set of rows**, because another transaction inserted/deleted rows matching the query in between

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public BigDecimal getAccountBalance(Long accountId) {
    return accountRepository.findById(accountId).getBalance();
}
```

**Default:** Spring uses `Isolation.DEFAULT`, which delegates to the underlying database's default isolation level (commonly `READ_COMMITTED` for PostgreSQL and Oracle, `REPEATABLE_READ` for MySQL/InnoDB).

**Trade-off to mention in interviews:** stricter isolation (`SERIALIZABLE`) gives the strongest consistency guarantees but reduces concurrency (more locking, more contention, potential deadlocks); looser isolation improves throughput but risks the anomalies above. Most applications use `READ_COMMITTED` as a practical balance.

---

**Q4. What are Rollback Rules?**

Rules that determine **which exceptions cause a `@Transactional` method to roll back**, versus which ones still allow the transaction to commit.

**Default behavior:**
- **Unchecked exceptions** (`RuntimeException` and its subclasses, plus `Error`) → **trigger a rollback** automatically
- **Checked exceptions** (anything extending `Exception` but not `RuntimeException`) → **do NOT trigger a rollback** by default — the transaction still commits

This default surprises many developers, since it's the opposite of what intuition might suggest.

**Customizing rollback behavior:**
```java
@Transactional(rollbackFor = InsufficientFundsException.class)
public void withdraw(Long accountId, BigDecimal amount) throws InsufficientFundsException {
    // rolls back even though InsufficientFundsException is a checked exception
}

@Transactional(noRollbackFor = MinorValidationWarning.class)
public void updateProfile(User user) {
    // this specific RuntimeException subtype will NOT trigger a rollback
}
```

- **`rollbackFor`** — forces rollback for exception types that wouldn't trigger it by default (typically checked exceptions)
- **`noRollbackFor`** — prevents rollback for exception types that would otherwise trigger it by default (typically unchecked exceptions)

**Interview tip:** always mention the default checked-vs-unchecked distinction explicitly — it's one of the most commonly asked "gotcha" questions in Spring transaction interviews, and many candidates get it backwards.

---

**Q5. Programmatic vs Declarative Transactions — what's the difference?**

**Declarative transactions** — using `@Transactional` — let you define transaction boundaries via annotation, with Spring's proxy mechanism handling the actual begin/commit/rollback logic behind the scenes. This is the standard, preferred approach in almost all Spring Boot applications.

```java
@Transactional
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    accountRepository.debit(fromId, amount);
    accountRepository.credit(toId, amount);
}
```

**Programmatic transactions** — manually controlling transaction boundaries in code using `TransactionTemplate` or `PlatformTransactionManager` directly, giving fine-grained control over exactly when a transaction starts, commits, or rolls back.

```java
@Autowired
private TransactionTemplate transactionTemplate;

public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    transactionTemplate.execute(status -> {
        try {
            accountRepository.debit(fromId, amount);
            accountRepository.credit(toId, amount);
            return null;
        } catch (Exception ex) {
            status.setRollbackOnly();
            throw ex;
        }
    });
}
```

| | Declarative (`@Transactional`) | Programmatic (`TransactionTemplate`) |
|---|---|---|
| Code style | Clean, minimal boilerplate | Verbose, more explicit control |
| Flexibility | Limited to annotation attributes | Full control — conditional transaction logic, mixing transactional/non-transactional code in one method |
| Testability | Easy | Easy |
| Common usage | Default choice, 95%+ of real-world code | Rare — used only when you need transaction boundaries that don't align cleanly with method boundaries |

**When programmatic makes sense:** when only *part* of a method needs to be transactional (declarative applies to the whole method), or when transaction behavior needs to be decided dynamically at runtime based on conditions that can't be expressed in an annotation.

---

## Extra / Important Interview Questions

**Q6. Why doesn't `@Transactional` work when calling a transactional method from within the same class ("self-invocation")?**

```java
@Service
public class UserService {

    public void updateUser(User user) {
        saveWithAudit(user);   // called internally — bypasses the proxy!
    }

    @Transactional
    public void saveWithAudit(User user) {
        userRepository.save(user);
    }
}
```

Spring's `@Transactional` relies on **proxying** — when an external caller calls `saveWithAudit()`, it goes through the proxy, which starts the transaction. But when `updateUser()` calls `saveWithAudit()` on `this` internally, that call is a plain Java method call, not routed through the proxy — so no transaction is started, and `@Transactional` on `saveWithAudit()` is silently ignored.

**Fixes:**
- Move the transactional method to a **separate bean/service** and inject it, so the call goes through the proxy
- Inject a **self-reference** via `ApplicationContext` or `@Lazy @Autowired UserService self` and call `self.saveWithAudit(user)`
- Use **AspectJ compile-time/load-time weaving** instead of proxy-based AOP (removes the proxy limitation entirely, but adds build complexity)

---

**Q7. What's the difference between `Propagation.REQUIRED` and `Propagation.REQUIRES_NEW` in terms of rollback behavior?**

With `REQUIRED`, an inner method joins the outer transaction — if the inner method throws and triggers a rollback, the **entire** transaction (outer + inner) rolls back together, since they're the same physical transaction. With `REQUIRES_NEW`, the inner method runs in a **completely separate** transaction — if the inner one fails and rolls back, the outer transaction is unaffected and can still commit independently (and vice versa).

---

**Q8. Can a `@Transactional` method call a `private` method and still have it be transactional?**

Only indirectly — `@Transactional` itself doesn't work on `private` methods at all (Spring's proxy-based AOP can't intercept private method calls, since they're not visible/overridable to a subclass proxy). If you annotate a private method with `@Transactional`, Spring silently ignores it — no exception, no transaction, which makes this a particularly dangerous, hard-to-notice bug in code review.

---

**Q9. What happens if you set `readOnly = true` on a `@Transactional` method — is it just documentation, or does it do something?**

It's a real optimization hint, not just documentation. With `@Transactional(readOnly = true)`:
- Hibernate can skip **dirty checking** (the process of comparing entity state to detect changes to flush) since no writes are expected, improving performance
- Some databases/drivers can route the connection differently (e.g., to a read replica) based on this flag
- It also signals intent clearly to other developers reading the code

```java
@Transactional(readOnly = true)
public List<User> getAllUsers() {
    return userRepository.findAll();
}
```
It doesn't strictly *prevent* a write inside the method at the JPA level (behavior here can vary by provider/driver), but it's a strong performance and correctness signal, and should be applied to all pure read operations as a best practice.

---

**Q10. If a `@Transactional` method catches an exception internally and doesn't rethrow it, does the transaction still roll back?**

No. Spring's transaction rollback mechanism is triggered by an exception **propagating out of the proxied method call**. If you catch a `RuntimeException` inside the method and swallow it (don't rethrow), Spring never sees the exception — the transaction proceeds to **commit** normally, even though something clearly went wrong inside. This is a common source of silent data-consistency bugs, and a favorite "spot the bug" interview question:

```java
@Transactional
public void process(Order order) {
    try {
        riskyOperation(order);
    } catch (Exception e) {
        log.error("Failed", e);   // swallowed — transaction still commits!
    }
}
```
Fix: either rethrow, or explicitly call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` inside the catch block.

---

**Q11. What is a deadlock in the context of database transactions, and how might isolation level choice contribute to it?**

A deadlock occurs when two transactions each hold a lock the other needs, and both wait indefinitely for the other to release it — the database detects this and typically aborts one transaction to break the cycle. Higher isolation levels (`REPEATABLE_READ`, `SERIALIZABLE`) hold locks for longer and more broadly, increasing the *likelihood* of lock contention and deadlocks under concurrent load, compared to `READ_COMMITTED`. This is why choosing the lowest isolation level that still meets your consistency requirements is generally the right default, rather than defaulting to the strictest for "safety."

---

*Interview tip: Transaction questions are almost always paired with "what's wrong with this code" snippets — self-invocation, swallowed exceptions, private methods, or checked-exception rollback defaults. Practice spotting these bugs quickly, since that's usually a stronger signal to interviewers than reciting propagation-type definitions.*