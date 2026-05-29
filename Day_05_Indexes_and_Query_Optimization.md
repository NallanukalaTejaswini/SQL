# Day 5 - Indexes, Query Optimization, and Execution Plans

---

## Table of Contents

1. What is an Index?
2. B-Tree Index Internals
3. Index Types
4. How the Query Planner Uses Indexes
5. Index Design Principles
6. Reading Execution Plans
7. Query Optimization Techniques
8. Query Anti-Patterns
9. Statistics and the Query Planner
10. Interview Questions and Problems
11. Tricks and Traps

---

## 1. What is an Index?

An index is a separate data structure that maintains a sorted reference to rows in a table, enabling the database engine to locate rows without scanning every row in the table. The analogy is a book's index: rather than reading every page to find "normalization," you look it up in the back and jump directly to the relevant pages.

Without an index, the database performs a sequential scan (also called a full table scan): it reads every row and evaluates the condition for each. For a table with 10 million rows, a sequential scan is expensive. An index on the filtered column reduces the lookup to O(log n) operations for a B-Tree index.

### The Cost of Indexes

Indexes are not free. Each index:
- Consumes disk space.
- Must be updated on every INSERT, UPDATE, and DELETE, adding write overhead.
- Can be ignored by the query planner if statistics are stale or the selectivity is low.

Indexing strategy is a balance between read performance and write overhead.

---

## 2. B-Tree Index Internals

The B-Tree (Balanced Tree) is the default and most common index structure.

### Structure

A B-Tree index consists of:
- **Root node**: The single entry point.
- **Internal nodes**: Contain key ranges that direct traversal.
- **Leaf nodes**: Contain the actual index keys and pointers (row identifiers / heap pointers) to the corresponding table rows.

The tree is always balanced: the path from root to any leaf node has the same depth. This guarantees O(log n) lookups regardless of data distribution.

### What a B-Tree Supports

| Operation | Supported |
|---|---|
| Equality: `WHERE col = value` | Yes |
| Range: `WHERE col BETWEEN a AND b` | Yes |
| Prefix: `WHERE col LIKE 'abc%'` | Yes |
| Suffix: `WHERE col LIKE '%abc'` | No (cannot use B-Tree) |
| Sorting: `ORDER BY col` | Yes (index scan avoids sort) |
| IS NULL | Yes (in PostgreSQL; not all databases) |

---

## 3. Index Types

### Single-Column Index

```sql
CREATE INDEX idx_employees_salary ON employees(salary);
CREATE INDEX idx_employees_last_name ON employees(last_name);
```

### Composite Index (Multi-Column)

```sql
CREATE INDEX idx_employees_dept_salary ON employees(department_id, salary);
```

A composite index on (department_id, salary) can be used for:
- Queries filtering on department_id alone.
- Queries filtering on both department_id AND salary.
- It cannot be efficiently used for queries filtering on salary alone.

This is the **leftmost prefix rule**: a composite index (a, b, c) supports queries on (a), (a, b), or (a, b, c) but not (b), (c), or (b, c) alone.

### Unique Index

```sql
CREATE UNIQUE INDEX idx_employees_email ON employees(email);
-- Enforces uniqueness and speeds up lookups by email
```

### Partial Index

An index covering only a subset of rows. Smaller, faster, and more targeted:

```sql
-- Index only active employees
CREATE INDEX idx_active_employees ON employees(last_name) WHERE status = 'Active';

-- Index only high-salary employees for a common filter
CREATE INDEX idx_high_salary ON employees(salary) WHERE salary > 100000;
```

### Covering Index (Index-Only Scan)

An index that includes all columns needed by a query, eliminating the need to access the table at all:

```sql
-- Query needs employee_id, first_name, salary for department 5
-- A covering index satisfies the entire query from the index alone
CREATE INDEX idx_cover_dept_emp ON employees(department_id, employee_id, first_name, salary);
```

In PostgreSQL, use INCLUDE to add non-key columns to the index leaf pages:

```sql
CREATE INDEX idx_dept_salary_cover
ON employees(department_id, salary)
INCLUDE (first_name, last_name);
```

### Expression Index (Functional Index)

Index on the result of an expression or function:

```sql
-- Without this, WHERE LOWER(email) = '...' cannot use a plain index on email
CREATE INDEX idx_lower_email ON employees(LOWER(email));

-- Now this query uses the index:
SELECT * FROM employees WHERE LOWER(email) = 'john.smith@example.com';
```

### Hash Index

Optimized for equality comparisons only. Faster than B-Tree for pure equality but does not support ranges, sorting, or LIKE. Rarely needed in practice.

```sql
CREATE INDEX idx_hash_employee_id ON employees USING HASH (employee_id);
```

---

## 4. How the Query Planner Uses Indexes

The query planner (also called query optimizer) generates an execution plan — the set of steps the database will take to execute a query. It estimates the cost of multiple possible plans and chooses the cheapest.

The planner considers:
- Table and index statistics (row counts, value distribution, NULL fraction).
- Available indexes and their types.
- Join methods (nested loop, hash join, merge join).
- Whether an index scan is cheaper than a sequential scan.

### When the Planner Ignores an Index

The planner may choose a sequential scan over an index scan when:
- The query returns a large fraction of rows (low selectivity). Scanning the whole table is faster than random I/O through an index for 80% of rows.
- Table statistics are stale.
- A function is applied to the indexed column in the WHERE clause.
- Implicit type casting prevents index use.

---

## 5. Index Design Principles

### Selectivity

Selectivity is the fraction of rows a condition eliminates. High selectivity (few matching rows) makes an index valuable. Low selectivity (many matching rows) makes an index worthless.

```sql
-- High selectivity: indexes are useful
WHERE employee_id = 12345          -- returns ~1 row
WHERE email = 'user@example.com'  -- returns ~1 row

-- Low selectivity: indexes are often ignored
WHERE status = 'Active'           -- returns 90% of rows
WHERE gender = 'M'                -- returns ~50% of rows
```

### Column Order in Composite Indexes

Place the most selective column first, or the column most commonly used in equality conditions. For a query like `WHERE department_id = 3 AND salary > 50000`, an index on (department_id, salary) is better than (salary, department_id) because department_id is used in an equality condition.

### Index Everything That is Joined On

Foreign key columns used in JOIN conditions are strong index candidates. Without them, joins degrade to nested-loop scans.

### Avoid Over-Indexing

Too many indexes on a write-heavy table cause significant INSERT/UPDATE/DELETE overhead. Each write must update every index. Audit and drop unused indexes regularly.

```sql
-- PostgreSQL: find indexes with zero or very few scans
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan < 10
ORDER BY idx_scan;
```

---

## 6. Reading Execution Plans

### PostgreSQL: EXPLAIN and EXPLAIN ANALYZE

```sql
-- Shows the planned execution steps and estimated costs
EXPLAIN SELECT * FROM employees WHERE department_id = 3;

-- Actually executes the query and shows real vs estimated metrics
EXPLAIN ANALYZE SELECT * FROM employees WHERE department_id = 3;
```

### Key Execution Plan Nodes

| Node | Description |
|---|---|
| Seq Scan | Full table scan: reads every row |
| Index Scan | Uses index to find rows, then fetches from table |
| Index Only Scan | Entire query satisfied from index; no table access |
| Bitmap Heap Scan | Uses index to build a bitmap, then fetches pages in order |
| Nested Loop | For each outer row, scans inner table. Good for small tables. |
| Hash Join | Builds a hash table from inner table, probes with outer. Good for large unsorted tables. |
| Merge Join | Sorts both tables then merges. Good when both inputs are already sorted. |
| Sort | Explicit sort operation (expensive; indicates missing index for ORDER BY) |
| Hash Aggregate | GROUP BY using hash table |
| Limit | Applies LIMIT/OFFSET |

### Understanding Cost Numbers

```
Seq Scan on employees  (cost=0.00..458.00 rows=10000 width=64)
                              ^startup  ^total   ^estimated  ^bytes per row
```

Cost is in arbitrary planner units. Total cost is what matters for comparison between plans. Lower is better.

### SQL Server: Execution Plans

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT * FROM employees WHERE department_id = 3;
```

Use SQL Server Management Studio's "Include Actual Execution Plan" (Ctrl+M) for graphical plans.

---

## 7. Query Optimization Techniques

### Avoid SELECT *

```sql
-- BAD: fetches all columns including unused ones; prevents index-only scans
SELECT * FROM employees WHERE department_id = 3;

-- GOOD: fetch only what you need
SELECT employee_id, first_name, salary FROM employees WHERE department_id = 3;
```

### Avoid Functions on Indexed Columns in WHERE

Applying a function to an indexed column prevents the index from being used unless a functional index exists:

```sql
-- BAD: index on hire_date cannot be used
WHERE YEAR(hire_date) = 2022;
WHERE DATE_TRUNC('year', hire_date) = '2022-01-01';

-- GOOD: express the condition as a range that the index can serve
WHERE hire_date >= '2022-01-01' AND hire_date < '2023-01-01';
```

### SARGable Predicates

A predicate is SARGable (Search ARGument ABLE) if the query engine can use an index to evaluate it. Non-SARGable predicates force full scans.

```sql
-- Non-SARGable (function on column)
WHERE UPPER(last_name) = 'SMITH'
WHERE salary + 1000 > 60000
WHERE SUBSTRING(phone, 1, 3) = '555'

-- SARGable equivalents
WHERE last_name = 'Smith'       -- Or use a functional index on UPPER(last_name)
WHERE salary > 59000
WHERE phone LIKE '555%'
```

### Use EXISTS Instead of COUNT for Existence Checks

```sql
-- BAD: counts all matching rows before checking
IF (SELECT COUNT(*) FROM orders WHERE customer_id = 101) > 0 ...

-- GOOD: stops at first match
IF EXISTS (SELECT 1 FROM orders WHERE customer_id = 101) ...
```

### Avoid Leading Wildcards in LIKE

```sql
-- BAD: cannot use index; full scan required
WHERE last_name LIKE '%son'

-- GOOD: index usable for prefix searches
WHERE last_name LIKE 'son%'
```

For full-text or suffix searches, use a full-text search index (PostgreSQL `tsvector`, MySQL FULLTEXT, or a dedicated search engine).

### Use UNION ALL Instead of UNION

UNION performs deduplication. UNION ALL does not. Unless duplicates must be removed, always use UNION ALL.

### Push Filters Down, Not Up

Apply the most selective filters as early as possible — in WHERE, not in outer queries or application code. Let the database do the filtering.

### Avoid N+1 Query Patterns

The N+1 problem occurs when an application executes 1 query to get N records, then N additional queries to get related data for each. Solve it with a single JOIN or a well-structured IN clause.

---

## 8. Query Anti-Patterns

### Implicit Type Conversion

```sql
-- BAD: employee_id is INT; the string '123' forces a type cast on every row
WHERE employee_id = '123'

-- GOOD
WHERE employee_id = 123
```

### NOT IN With a Subquery That Can Return NULLs

Covered in Day 1 but critical enough to repeat: if the subquery returns any NULL, NOT IN returns no rows.

```sql
-- DANGEROUS
WHERE department_id NOT IN (SELECT department_id FROM departments WHERE ...)
-- Safe only if department_id is NOT NULL in the subquery source

-- SAFE ALTERNATIVE
WHERE NOT EXISTS (
    SELECT 1 FROM departments d
    WHERE d.department_id = e.department_id AND ...
)
```

### OR Conditions That Block Index Use

```sql
-- This may not use indexes well on all databases
WHERE department_id = 3 OR salary > 100000

-- Sometimes faster when rewritten as UNION ALL (allows separate index scans per condition)
SELECT * FROM employees WHERE department_id = 3
UNION ALL
SELECT * FROM employees WHERE salary > 100000
AND department_id != 3;  -- Avoid duplicates
```

### Unbounded Pagination With OFFSET

```sql
-- SLOW for large offsets: the database must read and discard OFFSET rows
SELECT * FROM employees ORDER BY employee_id LIMIT 20 OFFSET 100000;

-- FAST: keyset pagination (seek method)
SELECT * FROM employees
WHERE employee_id > 100020  -- Last seen employee_id from previous page
ORDER BY employee_id
LIMIT 20;
```

---

## 9. Statistics and the Query Planner

The query planner relies on statistics about table data: row counts, column value distributions (histograms), number of distinct values, NULL fraction. If statistics are stale, the planner makes poor decisions.

```sql
-- PostgreSQL: manually update statistics
ANALYZE employees;
ANALYZE;  -- All tables

-- PostgreSQL: view column statistics
SELECT * FROM pg_stats WHERE tablename = 'employees';

-- SQL Server: update statistics
UPDATE STATISTICS employees;
UPDATE STATISTICS employees WITH FULLSCAN;  -- Full scan, most accurate

-- View index usage in PostgreSQL
SELECT * FROM pg_stat_user_indexes WHERE relname = 'employees';
```

### The Planner Can Be Wrong

The planner uses estimated row counts. When estimates are far off (due to data skew, correlated columns, or stale statistics), plans degrade. In PostgreSQL you can force index use with:

```sql
-- Temporarily disable sequential scans for testing (never do this in production permanently)
SET enable_seqscan = OFF;
EXPLAIN SELECT * FROM employees WHERE department_id = 3;
SET enable_seqscan = ON;
```

---

## 10. Interview Questions and Problems

### Theory Questions

**Q: What is an index and how does a B-Tree index work?**

An index is a separate data structure that stores sorted references to table rows, enabling fast lookups without full table scans. A B-Tree index is a self-balancing tree where all leaf nodes are at the same depth. Searches traverse from root to leaf in O(log n) time, following keys that narrow down the range until the matching leaf node is reached.

**Q: What is the leftmost prefix rule for composite indexes?**

A composite index (a, b, c) can only be used by the query planner if the query's WHERE clause includes the leftmost columns in sequence. A query filtering on (a), (a, b), or (a, b, c) can use the index. A query filtering on (b) or (c) alone cannot. A query on (a, c) can use the index for column a but not for c (since b is skipped).

**Q: What is a covering index?**

An index that contains all the columns required to satisfy a query (filter columns, join columns, and selected columns). When a covering index is used, the database engine never needs to access the actual table rows — it reads only the index, which is an index-only scan. This is significantly faster because it avoids random heap access.

**Q: When would a query planner choose a full table scan over an index scan?**

When the estimated number of matching rows is large (low selectivity), the cost of random I/O through an index exceeds sequential I/O through the table. A sequential scan reads pages in order with high memory throughput. An index scan causes random I/O, which is expensive for a large number of rows.

**Q: What is a partial index and when would you use one?**

A partial index is built on a subset of rows defined by a WHERE condition. It is smaller than a full index, faster to scan, and faster to update. Use it when queries consistently filter on a specific condition (e.g., only active records, only unprocessed queue items, only rows for a specific status).

**Q: What is a SARGable predicate?**

A predicate is SARGable (Search ARGument ABLE) if the database engine can use an index to evaluate it efficiently. Applying a function or transformation to the indexed column makes the predicate non-SARGable. Rewriting `WHERE YEAR(col) = 2022` as `WHERE col >= '2022-01-01' AND col < '2023-01-01'` restores SARGability.

### Coding / Design Problems

**Problem 1:** A query `SELECT * FROM orders WHERE customer_id = 101 AND order_date > '2023-01-01'` is slow. What indexes would you consider?

```sql
-- Single column indexes
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_order_date  ON orders(order_date);

-- Composite index (better): customer_id equality first, then date range
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Covering index if SELECT columns are known and fixed
CREATE INDEX idx_orders_cover
ON orders(customer_id, order_date)
INCLUDE (order_id, total_amount, status);
```

**Problem 2:** Rewrite this slow query to be SARGable.

```sql
-- SLOW (non-SARGable)
SELECT * FROM transactions WHERE CAST(created_at AS DATE) = '2024-06-15';

-- FAST (SARGable)
SELECT * FROM transactions
WHERE created_at >= '2024-06-15 00:00:00'
  AND created_at  < '2024-06-16 00:00:00';
```

**Problem 3:** Implement keyset pagination for a high-volume table.

```sql
-- Table: events (event_id BIGINT PK, event_time TIMESTAMP, user_id INT, payload TEXT)
-- Index: CREATE INDEX idx_events_time_id ON events(event_time DESC, event_id DESC);

-- First page
SELECT event_id, event_time, user_id
FROM events
ORDER BY event_time DESC, event_id DESC
LIMIT 50;

-- Subsequent pages (pass last seen event_time and event_id from previous page)
SELECT event_id, event_time, user_id
FROM events
WHERE (event_time, event_id) < ('2024-06-15 10:23:00', 98765)
ORDER BY event_time DESC, event_id DESC
LIMIT 50;
```

---

## 11. Tricks and Traps

### Indexes on Low-Cardinality Columns Are Rarely Useful

A column with only 2-3 distinct values (boolean flags, status codes with few values) has very low selectivity. The planner usually opts for a sequential scan. Exception: partial indexes or composite indexes where cardinality is improved by combination.

### Indexes Slow Down Writes

A table with 10 indexes will have noticeably slower INSERT, UPDATE, and DELETE operations than the same table with 2 indexes. For bulk load operations, consider dropping indexes, loading data, then rebuilding them.

### Index Bloat

After many UPDATE and DELETE operations, index pages can become fragmented and contain dead tuples. Periodically rebuild or reorganize indexes:

```sql
-- PostgreSQL: rebuild index without locking table
REINDEX INDEX CONCURRENTLY idx_employees_salary;

-- SQL Server
ALTER INDEX idx_employees_salary ON employees REBUILD;
ALTER INDEX idx_employees_salary ON employees REORGANIZE;  -- Less locking
```

### NULL and Indexes

Most databases include NULL values in B-Tree indexes (PostgreSQL does; Oracle does not by default). Be aware when filtering with IS NULL or IS NOT NULL whether the index will be used.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| B-Tree | Default index; supports equality, range, prefix, sort |
| Composite index | Leftmost prefix rule; order columns by selectivity and query pattern |
| Covering index | Eliminates table access entirely for qualifying queries |
| Partial index | Index a subset of rows for targeted, smaller, faster indexes |
| SARGable | Keep indexed columns free of functions in WHERE |
| EXPLAIN ANALYZE | Always verify the actual execution plan, not just the estimate |
| Statistics | Keep stats fresh; the planner is only as good as its information |

---

*Day 5 of 7 | Previous: Window Functions | Next: Transactions, Normalization, and Schema Design*
