# MySQL — Complete Details & Interview Questions

---

# Part 1: MySQL Fundamentals

## 1. What is MySQL?
MySQL is an open-source **Relational Database Management System (RDBMS)** that uses SQL (Structured Query Language). It's widely used with Java/Spring Boot, PHP, Node.js applications.

## 2. MySQL Data Types

| Category | Types |
|---|---|
| **Numeric** | `INT`, `TINYINT`, `SMALLINT`, `BIGINT`, `FLOAT`, `DOUBLE`, `DECIMAL` |
| **String** | `CHAR`, `VARCHAR`, `TEXT`, `BLOB`, `ENUM`, `SET` |
| **Date/Time** | `DATE`, `TIME`, `DATETIME`, `TIMESTAMP`, `YEAR` |
| **Others** | `JSON`, `BOOLEAN` (alias for `TINYINT(1)`) |

## 3. SQL Command Categories

| Category | Commands | Purpose |
|---|---|---|
| **DDL** | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` | Define/modify schema |
| **DML** | `INSERT`, `UPDATE`, `DELETE`, `SELECT` | Manipulate data |
| **DCL** | `GRANT`, `REVOKE` | Permissions |
| **TCL** | `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `SET TRANSACTION` | Transaction management |

## 4. Basic Syntax Examples

```sql
-- DDL
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE
);

ALTER TABLE employees ADD COLUMN email VARCHAR(100);
ALTER TABLE employees MODIFY COLUMN salary DECIMAL(12,2);
ALTER TABLE employees DROP COLUMN email;
DROP TABLE employees;
TRUNCATE TABLE employees;

-- DML
INSERT INTO employees (name, department, salary, hire_date)
VALUES ('John Doe', 'Engineering', 75000, '2023-01-15');

UPDATE employees SET salary = 80000 WHERE id = 1;
DELETE FROM employees WHERE id = 1;

SELECT * FROM employees WHERE department = 'Engineering' ORDER BY salary DESC;
```

## 5. Joins

```sql
-- INNER JOIN
SELECT e.name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

-- LEFT JOIN
SELECT e.name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;

-- RIGHT JOIN
SELECT e.name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;

-- FULL OUTER JOIN (MySQL doesn't support directly — use UNION)
SELECT e.name, d.department_name FROM employees e LEFT JOIN departments d ON e.department_id = d.id
UNION
SELECT e.name, d.department_name FROM employees e RIGHT JOIN departments d ON e.department_id = d.id;

-- SELF JOIN
SELECT a.name AS employee, b.name AS manager
FROM employees a
JOIN employees b ON a.manager_id = b.id;
```
# SQL JOINs — Complete Guide with Examples

---

## 1. Sample Tables Used Throughout

**employees**

| id  | name  | department_id | manager_id | salary |
| --- | ----- | -------------- | ---------- | ------ |
| 1   | Alice | 1              | NULL       | 90000  |
| 2   | Bob   | 1              | 1          | 70000  |
| 3   | Carol | 2              | 1          | 75000  |
| 4   | Dave  | 2              | 3          | 60000  |
| 5   | Eve   | NULL           | 1          | 65000  |

**departments**

| id  | department_name |
| --- | ---------------- |
| 1   | Engineering       |
| 2   | Sales             |
| 3   | Marketing         |

Note: Eve has no `department_id` (NULL), and Marketing (id=3) has no employees. These two gaps are used to show how each JOIN type behaves differently below.

---

## 2. INNER JOIN

Returns **only rows with matching values in both tables**.

```sql
SELECT e.name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
```

**Result:**

| name  | department_name |
| ----- | ---------------- |
| Alice | Engineering       |
| Bob   | Engineering       |
| Carol | Sales             |
| Dave  | Sales             |

Eve is excluded (no matching department_id). Marketing is excluded (no matching employee).

---

## 3. LEFT JOIN (LEFT OUTER JOIN)

Returns **all rows from the left table**, plus matched rows from the right table. Unmatched right-side columns show `NULL`.

```sql
SELECT e.name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

**Result:**

| name  | department_name |
| ----- | ---------------- |
| Alice | Engineering       |
| Bob   | Engineering       |
| Carol | Sales             |
| Dave  | Sales             |
| Eve   | NULL              |

Eve is included (from the left table) even though she has no department.

**Common use case — finding unmatched rows (employees with no department):**

```sql
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.id IS NULL;
```

Result: `Eve`

---

## 4. RIGHT JOIN (RIGHT OUTER JOIN)

Returns **all rows from the right table**, plus matched rows from the left table. Unmatched left-side columns show `NULL`. Mirror image of LEFT JOIN.

```sql
SELECT e.name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

**Result:**

| name  | department_name |
| ----- | ---------------- |
| Alice | Engineering       |
| Bob   | Engineering       |
| Carol | Sales             |
| Dave  | Sales             |
| NULL  | Marketing         |

Marketing is included (from the right table) even with zero employees.

**Common use case — finding departments with no employees:**

```sql
SELECT d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id
WHERE e.id IS NULL;
```

Result: `Marketing`

Note: `RIGHT JOIN` is rarely used in practice — the same result can always be written as a `LEFT JOIN` by swapping the table order, which most style guides prefer for readability.

---

## 5. FULL OUTER JOIN

Returns **all rows from both tables**, matched where possible, `NULL` where not. MySQL does **not support `FULL OUTER JOIN` directly** — it must be emulated with `UNION`:

```sql
SELECT e.name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id

UNION

SELECT e.name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

**Result:**

| name  | department_name |
| ----- | ---------------- |
| Alice | Engineering       |
| Bob   | Engineering       |
| Carol | Sales             |
| Dave  | Sales             |
| Eve   | NULL              |
| NULL  | Marketing         |

`UNION` (not `UNION ALL`) is used to avoid duplicating the matched rows that appear in both the LEFT and RIGHT results.

---

## 6. CROSS JOIN

Returns the **Cartesian product** — every row from the left table combined with every row from the right table. No `ON` condition needed.

```sql
SELECT e.name, d.department_name
FROM employees e
CROSS JOIN departments d;
```

**Result:** 5 employees × 3 departments = **15 rows** (every possible combination).

Practical use case: generating all possible combinations, e.g. a "sizes × colors" product matrix, or a calendar table cross-joined with a list of stores.

---

## 7. SELF JOIN

A table joined **with itself**, typically using aliases. Common for hierarchical data (e.g. employee → manager).

```sql
SELECT a.name AS employee, b.name AS manager
FROM employees a
LEFT JOIN employees b ON a.manager_id = b.id;
```

**Result:**

| employee | manager |
| -------- | ------- |
| Alice    | NULL    |
| Bob      | Alice   |
| Carol    | Alice   |
| Dave     | Carol   |
| Eve      | Alice   |

`LEFT JOIN` (not `INNER JOIN`) is used so that Alice — who has no manager — still appears in the result.

---

## 8. Joining More Than Two Tables

You can chain multiple JOINs together.

```sql
SELECT o.id AS order_id, c.name AS customer, p.product_name, oi.quantity
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p ON oi.product_id = p.id
WHERE o.order_date >= '2026-01-01';
```

Each JOIN adds one more related table. Order matters for readability but not for correctness, as long as each `ON` condition correctly links its pair of tables.

---

## 9. JOIN with Aggregate Functions

```sql
SELECT d.department_name, COUNT(e.id) AS employee_count, AVG(e.salary) AS avg_salary
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.department_name;
```

**Result:**

| department_name | employee_count | avg_salary |
| ---------------- | --------------- | ---------- |
| Engineering       | 2                | 80000      |
| Sales             | 2                | 67500      |
| Marketing         | 0                | NULL       |

`LEFT JOIN` from `departments` ensures Marketing still shows up with a count of 0, which an `INNER JOIN` would have excluded entirely.

---

## 10. JOIN vs Subquery (Performance Note)

Both queries below return the same result:

```sql
-- Using JOIN (generally faster, lets the optimizer pick the best plan)
SELECT e.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
WHERE d.department_name = 'Engineering';
```

```sql
-- Using Subquery (can be slower, especially if correlated)
SELECT name FROM employees
WHERE department_id = (SELECT id FROM departments WHERE department_name = 'Engineering');
```

JOINs are typically preferred for combining columns from multiple tables. Subqueries are more natural for existence checks (`EXISTS`) or single-value comparisons.

---

## 11. Spring Boot / JPA Equivalent

**JPQL JOIN:**

```java
@Query("SELECT e FROM Employee e JOIN e.department d WHERE d.name = :deptName")
List<Employee> findByDepartmentName(@Param("deptName") String deptName);
```

**JOIN FETCH (avoids N+1 problem, eagerly loads association):**

```java
@Query("SELECT d FROM Department d JOIN FETCH d.employees")
List<Department> findAllWithEmployees();
```

**LEFT JOIN in JPQL:**

```java
@Query("SELECT e FROM Employee e LEFT JOIN e.department d WHERE d IS NULL")
List<Employee> findEmployeesWithNoDepartment();
```

**Native SQL JOIN:**

```java
@Query(value = "SELECT e.* FROM employees e INNER JOIN departments d ON e.department_id = d.id WHERE d.department_name = :name",
       nativeQuery = true)
List<Employee> findByDeptNative(@Param("name") String name);
```

---

## 12. Quick Comparison Table

| JOIN Type          | Returns |
| ------------------- | ------- |
| `INNER JOIN`         | Only matching rows from both tables |
| `LEFT JOIN`           | All left rows + matched right rows (NULL if unmatched) |
| `RIGHT JOIN`          | All right rows + matched left rows (NULL if unmatched) |
| `FULL OUTER JOIN`     | All rows from both, matched or not (emulated via UNION in MySQL) |
| `CROSS JOIN`          | Cartesian product — every combination of rows |
| `SELF JOIN`           | Table joined with itself (via alias) |

---

## 13. Common Interview Questions on JOINs

**Q1: What happens if you JOIN without an ON clause?**

Without `ON`, it behaves like a `CROSS JOIN` (Cartesian product) if using comma syntax (`FROM a, b`), producing every possible row combination — usually unintended and a common source of duplicate-row bugs.

**Q2: Can you use multiple conditions in an ON clause?**

Yes: `ON e.dept_id = d.id AND e.active = 1`. This differs from adding the second condition in `WHERE` when using an outer join — putting it in `ON` filters before the join, in `WHERE` it filters after (which can turn a LEFT JOIN back into effectively an INNER JOIN for that condition).

**Q3: Why does adding a WHERE condition on the right table turn a LEFT JOIN into an INNER JOIN?**

```sql
-- This looks like a LEFT JOIN but behaves like an INNER JOIN
SELECT e.name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.department_name = 'Engineering';
```

Because unmatched rows have `d.department_name = NULL`, and `WHERE NULL = 'Engineering'` evaluates to false/unknown, those rows get filtered out — eliminating the "outer" NULL rows the LEFT JOIN was meant to preserve. To keep them, move the condition into the `ON` clause instead.

**Q4: How do you find rows that exist in one table but not another?**

```sql
-- Employees with no matching department (anti-join pattern)
SELECT e.name FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.id IS NULL;
```

**Q5: Is JOIN order important for correctness or performance?**

Not for correctness (INNER JOIN is commutative). It can matter for performance — though modern query optimizers (like MySQL's) usually reorder joins internally based on table statistics and indexes, regardless of the order written in the SQL.
## 6. Aggregate Functions & Grouping

```sql
SELECT department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING COUNT(*) > 5
ORDER BY avg_salary DESC;
```
- `WHERE` filters rows before grouping
- `HAVING` filters groups after aggregation

## 7. Indexes

```sql
CREATE INDEX idx_department ON employees(department);
CREATE UNIQUE INDEX idx_email ON employees(email);
DROP INDEX idx_department ON employees;
```

## 8. Constraints

| Constraint | Purpose |
|---|---|
| `PRIMARY KEY` | Uniquely identifies each row |
| `FOREIGN KEY` | Enforces referential integrity |
| `UNIQUE` | No duplicate values allowed |
| `NOT NULL` | Column cannot be null |
| `CHECK` | Validates values against a condition |
| `DEFAULT` | Sets a default value |

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    amount DECIMAL(10,2) CHECK (amount > 0),
    status VARCHAR(20) DEFAULT 'PENDING',
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

## 9. Subqueries

```sql
-- Subquery in WHERE
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated subquery
SELECT e.name FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE department = e.department);

-- EXISTS
SELECT name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);
```

## 10. Transactions

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;
-- or ROLLBACK; on failure
```

## 11. Views

```sql
CREATE VIEW high_earners AS
SELECT name, salary FROM employees WHERE salary > 100000;

SELECT * FROM high_earners;
```

## 12. Stored Procedures & Functions

```sql
DELIMITER //
CREATE PROCEDURE GetEmployeesByDept(IN dept_name VARCHAR(50))
BEGIN
    SELECT * FROM employees WHERE department = dept_name;
END //
DELIMITER ;

CALL GetEmployeesByDept('Engineering');
```

## 13. Triggers

```sql
CREATE TRIGGER before_employee_insert
BEFORE INSERT ON employees
FOR EACH ROW
SET NEW.hire_date = COALESCE(NEW.hire_date, CURDATE());
```

## 14. Window Functions

```sql
SELECT name, salary, department,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_in_dept,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS overall_rank,
       LAG(salary) OVER (ORDER BY hire_date) AS prev_salary
FROM employees;
```

---

# Part 2: MySQL Interview Questions & Answers

## Basics

**Q1: Difference between `CHAR` and `VARCHAR`?**
> `CHAR` is fixed-length (padded with spaces), faster for fixed-size data. `VARCHAR` is variable-length, stores only what's needed plus a length prefix — more storage-efficient for variable data.

**Q2: Difference between `DELETE`, `TRUNCATE`, and `DROP`?**
> `DELETE` removes rows (can use `WHERE`, logged, can rollback, triggers fire). `TRUNCATE` removes all rows quickly (resets AUTO_INCREMENT, minimal logging, can't use WHERE). `DROP` removes the entire table structure.

**Q3: Difference between `WHERE` and `HAVING`?**
> `WHERE` filters individual rows before grouping/aggregation. `HAVING` filters grouped results after `GROUP BY`/aggregation is applied.

**Q4: What is a Primary Key vs Foreign Key vs Unique Key?**
> Primary Key uniquely identifies a row (no NULLs, one per table). Foreign Key references a Primary Key in another table (enforces referential integrity). Unique Key ensures no duplicate values but allows one NULL.

**Q5: What's the difference between `UNION` and `UNION ALL`?**
> `UNION` combines result sets and removes duplicates (slower, does a distinct sort). `UNION ALL` combines and keeps duplicates (faster, no dedup).

## Joins & Relationships

**Q6: Explain the different types of JOINs.**
> `INNER JOIN` — matching rows only. `LEFT JOIN` — all left rows + matched right (NULL if no match). `RIGHT JOIN` — all right rows + matched left. `FULL OUTER JOIN` — all rows from both (MySQL emulates via `UNION` of LEFT+RIGHT). `SELF JOIN` — table joined with itself. `CROSS JOIN` — Cartesian product.

**Q7: What is a Composite Key?**
> A primary key made of two or more columns combined to uniquely identify a row, used when no single column is unique enough (e.g., `order_id + product_id` in an order-items table).

## Indexing & Performance

**Q8: What is an index and how does it improve performance?**
> An index is a data structure (typically B-Tree) that speeds up row lookup by avoiding full table scans. Trade-off: faster reads, but slower writes (INSERT/UPDATE/DELETE) since indexes must also be updated.

**Q9: What is a Clustered vs Non-Clustered Index?**
> Clustered index determines the physical storage order of table data (InnoDB's primary key is always clustered — one per table). Non-clustered index is a separate structure pointing back to the actual row (multiple allowed per table).

**Q10: How would you optimize a slow SQL query?**
> - Check `EXPLAIN` output for full table scans
> - Add appropriate indexes on WHERE/JOIN/ORDER BY columns
> - Avoid `SELECT *` — fetch only needed columns
> - Avoid functions on indexed columns in WHERE clause
> - Use `LIMIT` for pagination
> - Denormalize/cache for read-heavy workloads
> - Analyze query execution plan and rewrite subqueries as JOINs where beneficial

**Q11: What is `EXPLAIN` used for?**
> Shows the query execution plan — how MySQL will scan tables, which indexes it will use, join order, and estimated rows examined. Used to diagnose slow queries.

## Transactions & ACID

**Q12: What are ACID properties?**
> - **Atomicity** — transaction is all-or-nothing
> - **Consistency** — DB moves from one valid state to another
> - **Isolation** — concurrent transactions don't interfere with each other
> - **Durability** — committed changes survive system failure

**Q13: What are Transaction Isolation Levels?**
> - `READ UNCOMMITTED` — dirty reads possible
> - `READ COMMITTED` — no dirty reads, but non-repeatable reads possible
> - `REPEATABLE READ` — MySQL InnoDB default; no dirty/non-repeatable reads, but phantom reads possible
> - `SERIALIZABLE` — strictest, fully isolated, slowest

**Q14: What is a Deadlock and how do you handle it?**
> Occurs when two transactions hold locks the other needs, causing infinite wait. MySQL's InnoDB engine detects deadlocks automatically and rolls back one transaction. Prevention: consistent lock ordering, keep transactions short, use appropriate indexes to reduce lock scope.

## Normalization

**Q15: What is Normalization? Explain 1NF, 2NF, 3NF.**
> Process of organizing data to reduce redundancy.
> - **1NF** — atomic columns, no repeating groups
> - **2NF** — 1NF + no partial dependency on part of a composite key
> - **3NF** — 2NF + no transitive dependency (non-key columns depend only on the key)

**Q16: What is Denormalization and when would you use it?**
> Intentionally introducing redundancy (e.g., duplicating data across tables) to improve read performance, typically in reporting/analytics systems where read speed matters more than write efficiency or storage.

## Storage Engines

**Q17: Difference between InnoDB and MyISAM?**
> `InnoDB` — supports transactions, foreign keys, row-level locking, crash recovery (default since MySQL 5.5). `MyISAM` — no transaction support, table-level locking, faster for read-heavy/no-transaction workloads, no foreign key enforcement.

## Advanced

**Q18: What is the difference between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()`?**
> `ROW_NUMBER()` — unique sequential number, no ties. `RANK()` — same rank for ties, skips next rank(s) (1,2,2,4). `DENSE_RANK()` — same rank for ties, no skip (1,2,2,3).

**Q19: What is a Correlated Subquery vs a regular Subquery?**
> A regular subquery executes once, independently. A correlated subquery references a column from the outer query and re-executes once per outer row — generally slower, can often be rewritten as a JOIN for better performance.

**Q20: How do you find the second highest salary in a table?**
```sql
-- Using LIMIT/OFFSET
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;

-- Using subquery
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Using DENSE_RANK
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t WHERE rnk = 2;
```

**Q21: What is the difference between `NOW()`, `CURDATE()`, and `CURTIME()`?**
> `NOW()` returns current date+time. `CURDATE()` returns only date. `CURTIME()` returns only time.

**Q22: What are triggers and when would you use them?**
> Triggers are stored procedures that auto-execute on `INSERT`/`UPDATE`/`DELETE` events. Used for audit logging, enforcing complex business rules, auto-updating related tables, or maintaining derived/summary data.

**Q23: How do you prevent SQL Injection in raw queries?**
> Always use parameterized/prepared statements (`?` placeholders) instead of string concatenation. Never trust or directly embed user input into SQL strings.

**Q24: What is the difference between a View and a Table?**
> A table stores actual data physically. A view is a virtual table — a saved SELECT query that computes results dynamically each time it's queried; it doesn't store data itself (unless it's a materialized view, which MySQL doesn't natively support).

**Q25: What's the difference between `GROUP BY` and `PARTITION BY`?**
> `GROUP BY` collapses rows into aggregated groups (one row per group). `PARTITION BY` (used with window functions) keeps all individual rows but performs calculations within partitions — no row collapsing.

# SQL Some More Interview Questions

---

## 1. Sample Table Used Throughout

**employees**

| id  | name    | department | salary | manager_id |
| --- | ------- | ---------- | ------ | ---------- |
| 1   | Alice   | Engineering | 95000  | NULL       |
| 2   | Bob     | Engineering | 85000  | 1          |
| 3   | Carol   | Sales       | 85000  | 1          |
| 4   | Dave    | Sales       | 70000  | 3          |
| 5   | Eve     | Marketing   | 60000  | 1          |
| 6   | Frank   | Engineering | 70000  | 1          |

Note: Bob and Carol both earn 85000 (used to illustrate duplicate-handling below).

---

## 2. Second Highest Salary

**Method 1 — LIMIT/OFFSET (simplest):**

```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

**Method 2 — Subquery with MAX:**

```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

**Method 3 — DENSE_RANK (best when duplicates should count as one rank):**

```sql
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 2;
```

Result with the sample data above: **85000**

Why `DISTINCT` or `DENSE_RANK` matters: without it, `LIMIT 1 OFFSET 1` on non-distinct salaries could return 85000 twice (Bob's and Carol's) instead of skipping to the actual 2nd *distinct* value.

---

## 3. Nth Highest Salary (Generalized)

**Method 1 — LIMIT/OFFSET (N = 3 example):**

```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;   -- OFFSET = N-1
```

**Method 2 — DENSE_RANK (most flexible, handles ties correctly):**

```sql
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 3;   -- N = 3
```

**Method 3 — Correlated subquery (classic, works on older MySQL without window functions):**

```sql
SELECT DISTINCT salary
FROM employees e1
WHERE 2 = (
    SELECT COUNT(DISTINCT salary)
    FROM employees e2
    WHERE e2.salary >= e1.salary
);
-- Change "2" to N for the Nth highest
```

**RANK() vs DENSE_RANK() vs ROW_NUMBER() — the key difference for salary problems:**

| Salary | ROW_NUMBER() | RANK() | DENSE_RANK() |
| ------ | ------------- | ------ | ------------- |
| 95000  | 1             | 1      | 1             |
| 85000  | 2             | 2      | 2             |
| 85000  | 3             | 2      | 2             |
| 70000  | 4             | 4      | 3             |
| 70000  | 5             | 4      | 3             |
| 60000  | 6             | 6      | 4             |

Use `DENSE_RANK()` when duplicate salaries should share a rank without skipping the next number (most common interview expectation for "Nth highest distinct salary").

---

## 4. Second/Nth Highest Salary **Per Department** (common follow-up)

```sql
SELECT department, name, salary FROM (
    SELECT department, name, salary,
           DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 2;
```

`PARTITION BY` resets the ranking for each department, so you get the 2nd highest within Engineering, Sales, and Marketing separately.

---

## 5. DISTINCT

Removes duplicate rows from the result set.

```sql
SELECT DISTINCT department FROM employees;
```

**Result:** Engineering, Sales, Marketing (no duplicates even though multiple employees share a department)

**DISTINCT across multiple columns** (uniqueness applies to the combination, not each column individually):

```sql
SELECT DISTINCT department, salary FROM employees;
```
This returns unique `(department, salary)` pairs — Bob and Carol have the same salary but different departments, so both rows remain.

**Counting distinct values:**

```sql
SELECT COUNT(DISTINCT department) AS dept_count FROM employees;
```

---

## 6. GROUP BY

Groups rows sharing a value into summary rows, typically used with aggregate functions.

```sql
SELECT department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary, MAX(salary) AS top_salary
FROM employees
GROUP BY department;
```

**Result:**

| department  | emp_count | avg_salary | top_salary |
| ------------ | --------- | ---------- | ---------- |
| Engineering  | 3         | 83333.33   | 95000      |
| Sales        | 2         | 77500      | 85000      |
| Marketing    | 1         | 60000      | 60000      |

**GROUP BY with HAVING** (filters groups after aggregation):

```sql
SELECT department, COUNT(*) AS emp_count
FROM employees
GROUP BY department
HAVING COUNT(*) > 1;
```
Only Engineering and Sales qualify (Marketing has just 1 employee).

**Key rule:** every non-aggregated column in `SELECT` must appear in `GROUP BY`, or MySQL will throw an error under `ONLY_FULL_GROUP_BY` mode (enabled by default in modern MySQL).

---

## 7. WHERE Clause

Filters individual rows **before** grouping/aggregation.

```sql
SELECT name, salary FROM employees
WHERE salary > 70000 AND department = 'Engineering';
```

**WHERE vs HAVING — side by side:**

```sql
-- WHERE: filters rows first, then groups the remainder
SELECT department, COUNT(*) FROM employees
WHERE salary > 65000
GROUP BY department;

-- HAVING: groups everything first, then filters the resulting groups
SELECT department, COUNT(*) AS emp_count FROM employees
GROUP BY department
HAVING COUNT(*) > 1;
```

You **cannot** use aggregate functions directly in `WHERE` (e.g. `WHERE COUNT(*) > 1` is invalid) — that's exactly why `HAVING` exists.

---

## 8. ORDER BY

```sql
SELECT name, salary FROM employees
ORDER BY department ASC, salary DESC;
```
Sorts by department alphabetically, then by salary descending within each department.

---

## 9. Duplicate Rows — Find and Delete

**Find duplicates:**

```sql
SELECT salary, COUNT(*) AS occurrences
FROM employees
GROUP BY salary
HAVING COUNT(*) > 1;
```

**Delete duplicates, keeping the lowest id:**

```sql
DELETE e1 FROM employees e1
INNER JOIN employees e2
WHERE e1.id > e2.id AND e1.salary = e2.salary AND e1.department = e2.department;
```

---

## 10. Employees Earning More Than Their Manager (classic self-join question)

```sql
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

---

## 11. Running Total / Cumulative Sum

```sql
SELECT name, salary,
       SUM(salary) OVER (ORDER BY id) AS running_total
FROM employees;
```

---

## 12. Percentage of Total

```sql
SELECT department,
       SUM(salary) AS dept_total,
       ROUND(SUM(salary) * 100.0 / (SELECT SUM(salary) FROM employees), 2) AS pct_of_total
FROM employees
GROUP BY department;
```

---

## 13. Pagination Pattern (used constantly in Spring Boot APIs)

```sql
SELECT * FROM employees
ORDER BY id
LIMIT 10 OFFSET 20;   -- page 3, page size 10
```

---

# Part 2: Spring Boot / JPA Equivalents

## A. Nth Highest Salary

**Using @Query (JPQL):**
```java
@Query("SELECT DISTINCT e.salary FROM Employee e ORDER BY e.salary DESC")
List<Double> findDistinctSalariesDesc(Pageable pageable);

// Usage: get 2nd highest
Pageable pageable = PageRequest.of(1, 1); // page index 1 = 2nd row, size 1
List<Double> result = employeeRepository.findDistinctSalariesDesc(pageable);
```

**Using native SQL with window function:**
```java
@Query(value = """
    SELECT salary FROM (
        SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
        FROM employees
    ) t WHERE rnk = :n
    """, nativeQuery = true)
Double findNthHighestSalary(@Param("n") int n);
```

## B. DISTINCT in Spring Data JPA

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<String> findDistinctDepartmentBy();  // SELECT DISTINCT department FROM employees
}
```

## C. GROUP BY equivalent (JPQL projection)

```java
public interface DeptSummary {
    String getDepartment();
    Long getEmpCount();
    Double getAvgSalary();
}

@Query("SELECT e.department AS department, COUNT(e) AS empCount, AVG(e.salary) AS avgSalary " +
       "FROM Employee e GROUP BY e.department")
List<DeptSummary> getDepartmentSummary();
```

## D. WHERE equivalent (derived query methods)

```java
List<Employee> findBySalaryGreaterThanAndDepartment(double salary, String department);
// Generates: WHERE salary > ? AND department = ?
```

## E. HAVING equivalent

```java
@Query("SELECT e.department, COUNT(e) FROM Employee e GROUP BY e.department HAVING COUNT(e) > :min")
List<Object[]> findDepartmentsWithMoreThan(@Param("min") long min);
```

## F. Pagination (Spring Data's built-in mechanism)

```java
Page<Employee> page = employeeRepository.findAll(PageRequest.of(2, 10, Sort.by("salary").descending()));
// page index 2, size 10 → generates LIMIT 10 OFFSET 20 under the hood
```

## G. Self-Join equivalent (employees earning more than manager)

```java
@Query("SELECT e FROM Employee e JOIN Employee m ON e.managerId = m.id WHERE e.salary > m.salary")
List<Employee> findEmployeesEarningMoreThanManager();
```

---

# Part 3: Rapid-Fire Interview Q&A

**Q1: How do you get the Nth highest salary without window functions (older MySQL)?**
> Use a correlated subquery counting how many distinct salaries are `>=` each row's salary, and match where that count equals N.

**Q2: Why use `DISTINCT` before `LIMIT/OFFSET` for Nth salary problems?**
> Without it, duplicate salary values consume multiple offset positions, so `OFFSET 1` might return a duplicate of the top salary instead of the true second-highest distinct value.

**Q3: What's the difference between `WHERE` and `HAVING` in one line?**
> `WHERE` filters rows before aggregation; `HAVING` filters groups after aggregation — and only `HAVING` can reference aggregate functions like `COUNT()`/`SUM()`.

**Q4: Can `GROUP BY` be used without an aggregate function?**
> Yes — it still deduplicates based on the grouped column(s), similar to `DISTINCT`, though using `DISTINCT` is more idiomatic when no aggregation is needed.

**Q5: How is pagination implemented in Spring Data JPA under the hood?**
> `Pageable`/`PageRequest` translates page number and size into `LIMIT`/`OFFSET` SQL clauses automatically when the query executes.

**Q6: What's the danger of `ONLY_FULL_GROUP_BY` mode in MySQL?**
> It rejects queries where a selected column isn't in `GROUP BY` or wrapped in an aggregate function, preventing ambiguous results — a common source of "why did my query break after a MySQL upgrade" issues.

**Q7: How would you find the 3rd highest salary per department using Spring Data JPA?**
> Typically done via a native query with `DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC)` wrapped in a subquery filtered to `rnk = 3`, since JPQL doesn't support window functions directly — native SQL is required.