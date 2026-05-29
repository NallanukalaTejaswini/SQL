# Day 6 - Transactions, Concurrency, Normalization, and Schema Design

---

## Table of Contents

1. Transactions and ACID Properties
2. Transaction Control: COMMIT, ROLLBACK, SAVEPOINT
3. Isolation Levels and Concurrency Phenomena
4. Locking
5. Normalization: 1NF through BCNF
6. Denormalization
7. Schema Design Patterns
8. Views
9. Materialized Views
10. Interview Questions and Problems
11. Tricks and Traps

---

## 1. Transactions and ACID Properties

A transaction is a sequence of SQL operations executed as a single logical unit of work. Either all operations succeed and are committed, or all are rolled back and the database returns to its prior state. There is no partial success.

### ACID Properties

| Property | Definition |
|---|---|
| Atomicity | A transaction is all-or-nothing. Either all operations within it succeed, or none are applied. |
| Consistency | A transaction brings the database from one valid state to another valid state. Constraints, rules, and invariants are maintained. |
| Isolation | Concurrent transactions execute as if they were serial. Intermediate states of a transaction are not visible to other transactions (to varying degrees depending on isolation level). |
| Durability | Once a transaction is committed, the changes are permanent and survive system failures (power loss, crash). Data is written to durable storage. |

### Why ACID Matters

Without ACID, a bank transfer (debit account A, credit account B) could partially succeed: money leaves account A but never arrives in account B due to a crash between the two operations. Atomicity prevents this.

---

## 2. Transaction Control

### Explicit Transactions

```sql
-- PostgreSQL / SQL Server / Oracle
BEGIN;  -- or START TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;

COMMIT;  -- Makes both changes permanent
```

```sql
-- Rolling back on error
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;

-- Something goes wrong
ROLLBACK;  -- Neither change is applied
```

### SAVEPOINT

A savepoint marks a point within a transaction to which you can roll back without abandoning the entire transaction:

```sql
BEGIN;

INSERT INTO orders (customer_id, total) VALUES (101, 250.00);
SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, qty) VALUES (1001, 55, 2);
-- If this fails, roll back to the savepoint, not the beginning
ROLLBACK TO SAVEPOINT after_order;

-- The order insert is still pending
COMMIT;  -- Only the order insert is committed
```

### Autocommit

Most database clients operate in autocommit mode by default: every single SQL statement is automatically committed. To run a multi-statement transaction, you must explicitly begin one.

```sql
-- Disable autocommit (PostgreSQL via psql)
\set AUTOCOMMIT off

-- Or explicitly wrap in BEGIN/COMMIT
```

---

## 3. Isolation Levels and Concurrency Phenomena

When multiple transactions run concurrently, they can interfere with each other. The SQL standard defines four isolation levels to control the degree of interference, trading consistency for performance.

### Concurrency Phenomena

| Phenomenon | Description |
|---|---|
| Dirty Read | Transaction A reads uncommitted changes made by Transaction B. If B rolls back, A has read data that never existed. |
| Non-Repeatable Read | Transaction A reads the same row twice and gets different values because Transaction B committed an update between the two reads. |
| Phantom Read | Transaction A executes the same range query twice and gets different sets of rows because Transaction B inserted or deleted rows between the two reads. |
| Lost Update | Two transactions read the same value and both update it. The second commit overwrites the first without seeing it. |

### Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented |

Higher isolation levels prevent more phenomena but reduce concurrency and throughput.

```sql
-- Setting isolation level (PostgreSQL)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Setting isolation level (SQL Server)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### PostgreSQL and MVCC

PostgreSQL uses Multi-Version Concurrency Control (MVCC). Instead of locking rows when reading, PostgreSQL maintains multiple versions of each row. Readers never block writers and writers never block readers. Each transaction sees a consistent snapshot of the database as of its start time (in REPEATABLE READ) or statement start time (in READ COMMITTED).

---

## 4. Locking

### Row-Level Locks

```sql
-- SELECT FOR UPDATE: locks selected rows for update within the current transaction
-- Other transactions attempting to update these rows will wait
BEGIN;
SELECT * FROM accounts WHERE account_id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
COMMIT;

-- SELECT FOR SHARE: locks rows for shared read; allows others to read but not update
SELECT * FROM products WHERE product_id = 10 FOR SHARE;
```

### Table-Level Locks

```sql
-- Explicit table lock (use sparingly)
LOCK TABLE employees IN EXCLUSIVE MODE;
```

### Deadlocks

A deadlock occurs when two transactions each hold a lock that the other needs:
- Transaction A locks row 1, then tries to lock row 2.
- Transaction B locks row 2, then tries to lock row 1.
- Both wait forever.

Databases detect deadlocks and automatically kill one transaction (the victim) and return an error. Prevention: always acquire locks in the same order across all transactions.

---

## 5. Normalization

Normalization is the process of organizing a relational database schema to reduce data redundancy and improve data integrity. Each normal form (NF) builds on the previous.

### First Normal Form (1NF)

Requirements:
- Each column contains atomic (indivisible) values.
- Each column contains values of a single type.
- Each row is uniquely identifiable (has a primary key).
- No repeating groups or arrays in columns.

```sql
-- VIOLATES 1NF: phone_numbers column stores multiple values
CREATE TABLE contacts_bad (
    contact_id  INT PRIMARY KEY,
    name        VARCHAR(100),
    phone_numbers VARCHAR(255)  -- e.g., '555-1234, 555-5678, 555-9999'
);

-- SATISFIES 1NF: separate table for phone numbers
CREATE TABLE contacts (
    contact_id INT PRIMARY KEY,
    name       VARCHAR(100)
);

CREATE TABLE contact_phones (
    phone_id   INT PRIMARY KEY,
    contact_id INT REFERENCES contacts(contact_id),
    phone      VARCHAR(20),
    type       VARCHAR(10)  -- 'mobile', 'home', 'work'
);
```

### Second Normal Form (2NF)

Requirements:
- Must be in 1NF.
- Every non-key attribute must be fully functionally dependent on the entire primary key (not just part of it).
- Applies only to tables with composite primary keys.

```sql
-- VIOLATES 2NF: (order_id, product_id) is PK, but product_name depends only on product_id
CREATE TABLE order_items_bad (
    order_id     INT,
    product_id   INT,
    product_name VARCHAR(100),  -- Partial dependency on product_id alone
    quantity     INT,
    unit_price   DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)
);

-- SATISFIES 2NF: product_name moved to products table
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id   INT,
    product_id INT REFERENCES products(product_id),
    quantity   INT,
    unit_price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)
);
```

### Third Normal Form (3NF)

Requirements:
- Must be in 2NF.
- No non-key attribute depends on another non-key attribute (no transitive dependencies).

```sql
-- VIOLATES 3NF: zip_code -> city -> state (transitive dependency)
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    name        VARCHAR(100),
    zip_code    CHAR(5),
    city        VARCHAR(100),  -- Depends on zip_code, not employee_id
    state       CHAR(2)        -- Depends on zip_code, not employee_id
);

-- SATISFIES 3NF
CREATE TABLE zip_codes (
    zip_code CHAR(5) PRIMARY KEY,
    city     VARCHAR(100),
    state    CHAR(2)
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name        VARCHAR(100),
    zip_code    CHAR(5) REFERENCES zip_codes(zip_code)
);
```

### Boyce-Codd Normal Form (BCNF)

Requirements:
- Must be in 3NF.
- For every functional dependency X -> Y, X must be a superkey (a key or superset of a key).

BCNF is a stricter form of 3NF. A table can be in 3NF but not BCNF when there are multiple overlapping candidate keys.

### Fourth Normal Form (4NF)

Eliminates multi-valued dependencies. If a table contains two or more independent multi-valued facts about an entity, decompose it.

### Practical Normalization Target

Most production schemas target 3NF. BCNF and 4NF are applied when anomalies arise. Over-normalization leads to excessive joins and reduced query performance.

### Functional Dependencies

A functional dependency A -> B means: knowing A uniquely determines B. For example, `employee_id -> salary` (each employee_id corresponds to exactly one salary). Understanding functional dependencies is the foundation of normalization.

---

## 6. Denormalization

Denormalization intentionally introduces redundancy to improve query performance. Common in analytical (OLAP) systems and data warehouses.

### When to Denormalize

- Read-heavy workloads where joins are expensive.
- Data warehouse or reporting schemas where write latency is acceptable.
- Aggregate values that are expensive to recompute on every query.

### Common Denormalization Techniques

```sql
-- Store a pre-computed column to avoid expensive joins
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);
UPDATE orders o
SET customer_name = c.name
FROM customers c WHERE c.customer_id = o.customer_id;
-- Trade-off: must be kept in sync on every customer name change

-- Store aggregate values
ALTER TABLE departments ADD COLUMN employee_count INT DEFAULT 0;
-- Update via trigger or application logic on insert/delete of employees
```

---

## 7. Schema Design Patterns

### Star Schema (Data Warehouse)

A central fact table surrounded by dimension tables. Optimized for analytical queries.

```sql
-- Fact table: stores measurable events
CREATE TABLE fact_sales (
    sale_id       BIGINT PRIMARY KEY,
    date_key      INT REFERENCES dim_date(date_key),
    product_key   INT REFERENCES dim_product(product_key),
    customer_key  INT REFERENCES dim_customer(customer_key),
    quantity      INT,
    revenue       DECIMAL(12,2),
    discount      DECIMAL(10,2)
);

-- Dimension tables: descriptive attributes
CREATE TABLE dim_product (
    product_key  INT PRIMARY KEY,
    product_name VARCHAR(100),
    category     VARCHAR(50),
    brand        VARCHAR(50)
);

CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,  -- e.g., 20240615
    full_date DATE,
    year      INT,
    quarter   INT,
    month     INT,
    week      INT,
    day_name  VARCHAR(10)
);
```

### Surrogate Keys vs Natural Keys

| Approach | Surrogate Key | Natural Key |
|---|---|---|
| Definition | System-generated (INT, UUID) | Real-world attribute (SSN, email) |
| Stability | Stable; never changes | May change (email changes, SSN reassigned) |
| Join performance | INT joins are faster than VARCHAR joins | Depends on key type |
| Business meaning | None | Carries meaning; may leak PII |
| Recommendation | Preferred for most tables | Acceptable for truly immutable identifiers |

### Temporal Data Patterns

```sql
-- Slowly Changing Dimension Type 2: full history of changes
CREATE TABLE employee_history (
    history_id   INT PRIMARY KEY,
    employee_id  INT,
    department_id INT,
    salary       DECIMAL(10,2),
    valid_from   DATE NOT NULL,
    valid_to     DATE,  -- NULL means currently active
    is_current   BOOLEAN DEFAULT TRUE
);

-- Query current record
SELECT * FROM employee_history WHERE employee_id = 101 AND is_current = TRUE;

-- Query historical state
SELECT * FROM employee_history
WHERE employee_id = 101
AND '2022-06-01' BETWEEN valid_from AND COALESCE(valid_to, '9999-12-31');
```

---

## 8. Views

A view is a named, stored SELECT statement. It does not store data (unless materialized). Each time the view is queried, the underlying SELECT is executed.

```sql
-- Create a view
CREATE VIEW v_employee_details AS
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    d.department_name,
    e.salary,
    e.hire_date
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- Query the view like a table
SELECT * FROM v_employee_details WHERE department_name = 'Engineering';

-- Modify a view
CREATE OR REPLACE VIEW v_employee_details AS
SELECT ... ;  -- New definition

-- Drop a view
DROP VIEW v_employee_details;
```

### Updatable Views

A view is updatable (supports INSERT, UPDATE, DELETE) only if it meets strict conditions: based on a single table, no DISTINCT, no aggregate functions, no GROUP BY, no HAVING, no UNION. Most complex views are read-only.

### Benefits of Views

- Abstraction: hide schema complexity from application developers.
- Security: expose only specific columns/rows (column-level and row-level security).
- Simplification: encapsulate frequently used complex queries.
- Backward compatibility: when refactoring schemas, views can maintain the old interface.

---

## 9. Materialized Views

A materialized view stores the result of a query physically on disk. It is precomputed and must be refreshed when the underlying data changes.

```sql
-- PostgreSQL
CREATE MATERIALIZED VIEW mv_dept_stats AS
SELECT
    department_id,
    COUNT(*)    AS headcount,
    AVG(salary) AS avg_salary,
    SUM(salary) AS total_payroll
FROM employees
GROUP BY department_id;

-- Create an index on the materialized view for fast queries
CREATE INDEX idx_mv_dept_stats ON mv_dept_stats(department_id);

-- Refresh the materialized view (blocking)
REFRESH MATERIALIZED VIEW mv_dept_stats;

-- Refresh without locking reads (PostgreSQL 9.4+)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_dept_stats;
-- Requires a UNIQUE index on the materialized view
```

### View vs Materialized View

| Feature | View | Materialized View |
|---|---|---|
| Data stored | No: executed on query | Yes: precomputed result |
| Freshness | Always current | Stale until refreshed |
| Query performance | Depends on underlying query | Fast: pre-aggregated |
| Storage | None | Disk space required |
| Best use case | Simplification, security | Expensive aggregations queried frequently |

---

## 10. Interview Questions and Problems

### Theory Questions

**Q: Explain ACID properties with a real-world example.**

Consider a bank transfer of $500 from Account A to Account B. Atomicity ensures both the debit and credit happen together or not at all. Consistency ensures account balances never go negative (if that constraint exists) and total money is conserved. Isolation ensures another transaction reading Account A's balance mid-transfer sees either the original amount or the final amount, not an intermediate state. Durability ensures that after the transfer commits, the new balances survive even if the server crashes immediately after.

**Q: What is the difference between READ COMMITTED and REPEATABLE READ isolation levels?**

In READ COMMITTED, a transaction always reads the latest committed data. If another transaction commits a change between two reads of the same row, the second read returns the new value (non-repeatable read is possible). In REPEATABLE READ, a transaction sees a consistent snapshot from its start time. The same row read twice within the same transaction returns the same value regardless of concurrent commits.

**Q: What is a deadlock and how do you prevent it?**

A deadlock occurs when two or more transactions each hold a lock that another needs, forming a cycle. Prevention strategies include: always acquiring locks in the same consistent order across all transactions; keeping transactions short to minimize lock hold time; using SELECT FOR UPDATE only when necessary; designing application logic to avoid holding locks during user interaction.

**Q: Explain the difference between 2NF and 3NF.**

2NF eliminates partial dependencies: every non-key attribute must depend on the whole primary key, not just part of it (applies to composite keys). 3NF eliminates transitive dependencies: no non-key attribute should depend on another non-key attribute. Both reduce update anomalies. 2NF is about removing partial key dependencies; 3NF is about removing indirect (transitive) dependencies through non-key columns.

**Q: What is the difference between a view and a materialized view?**

A view is a stored query that is executed each time it is referenced. It always returns current data but has no precomputed storage. A materialized view stores the query result physically and must be explicitly refreshed. Materialized views are faster to query (especially for expensive aggregations) but can return stale data. Views are always fresh but can be slow if the underlying query is complex.

**Q: What is MVCC?**

Multi-Version Concurrency Control is a concurrency strategy where the database maintains multiple versions of each row. Readers see a consistent snapshot from the start of their transaction without locking rows. Writers create new row versions rather than overwriting in place. This means reads never block writes and writes never block reads, enabling high concurrency. PostgreSQL uses MVCC natively; Oracle uses a similar mechanism (undo segments).

### Coding Problems

**Problem 1:** Write a transaction to transfer funds between accounts safely, handling the case where the source account has insufficient funds.

```sql
BEGIN;

-- Lock both rows to prevent concurrent modifications
SELECT account_id, balance
FROM accounts
WHERE account_id IN (1, 2)
FOR UPDATE;

-- Check sufficient funds (application would check the result here)
-- If balance < 500, ROLLBACK and return error to application

UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;

-- Log the transaction
INSERT INTO transaction_log (from_account, to_account, amount, executed_at)
VALUES (1, 2, 500, CURRENT_TIMESTAMP);

COMMIT;
```

**Problem 2:** Normalize the following table to 3NF.

```
order_details (order_id, customer_id, customer_name, customer_email,
               product_id, product_name, product_category, quantity, unit_price)
```

```sql
-- Normalized to 3NF:

CREATE TABLE customers (
    customer_id    INT PRIMARY KEY,
    customer_name  VARCHAR(100),
    customer_email VARCHAR(255) UNIQUE
);

CREATE TABLE products (
    product_id       INT PRIMARY KEY,
    product_name     VARCHAR(100),
    product_category VARCHAR(50)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_date  DATE
);

CREATE TABLE order_items (
    order_id   INT REFERENCES orders(order_id),
    product_id INT REFERENCES products(product_id),
    quantity   INT,
    unit_price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)
);
```

**Problem 3:** Create a view that shows each employee, their department, and whether their salary is above or below the department average.

```sql
CREATE VIEW v_salary_vs_dept_avg AS
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    d.department_name,
    e.salary,
    ROUND(AVG(e.salary) OVER (PARTITION BY e.department_id), 2) AS dept_avg,
    CASE
        WHEN e.salary > AVG(e.salary) OVER (PARTITION BY e.department_id)
        THEN 'Above Average'
        WHEN e.salary < AVG(e.salary) OVER (PARTITION BY e.department_id)
        THEN 'Below Average'
        ELSE 'At Average'
    END AS salary_position
FROM employees e
JOIN departments d ON e.department_id = d.department_id;
```

---

## 11. Tricks and Traps

### Long Transactions Hold Locks

A transaction that remains open for a long time holds its locks for that duration. In PostgreSQL with MVCC, long transactions also prevent VACUUM from cleaning dead row versions, leading to table bloat. Always commit or rollback transactions promptly.

### Phantom Reads in REPEATABLE READ

In PostgreSQL's REPEATABLE READ, phantom reads are actually prevented due to the snapshot mechanism. However, in MySQL's REPEATABLE READ, phantoms can still occur in some edge cases. Know the specific behavior of your database.

### Normalization is Not Always Correct

Highly normalized schemas require many joins. For OLTP (transactional) systems, 3NF is usually appropriate. For OLAP (analytical) systems, star schemas (intentionally denormalized) are preferred for query performance. There is no universally correct answer; it depends on the workload.

### View Performance Is Not Automatic

A view does not inherently improve performance. If the view contains complex aggregations or joins and is queried frequently, it may actually hurt performance by hiding the complexity. Use materialized views for performance optimization.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| ACID | Atomicity, Consistency, Isolation, Durability — guarantees of reliable transactions |
| Isolation levels | Higher isolation = fewer concurrency anomalies = lower throughput |
| MVCC | Readers don't block writers; each transaction sees a consistent snapshot |
| Deadlocks | Acquire locks in consistent order; keep transactions short |
| 1NF | Atomic values, no repeating groups |
| 2NF | No partial dependencies on composite key |
| 3NF | No transitive dependencies through non-key columns |
| View | Stored query; always current; no storage |
| Materialized View | Stored result; fast; must be refreshed |

---

*Day 6 of 7 | Previous: Indexes and Optimization | Next: Advanced SQL and Interview Master Class*
