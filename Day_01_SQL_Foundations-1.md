# Day 1 - SQL Foundations: Databases, Querying, and Core Concepts

---

## Table of Contents

1. What is a Database?
2. Relational vs Non-Relational Databases
3. The Relational Model
4. SQL Sublanguages
5. Data Types
6. The SELECT Statement
7. Filtering with WHERE
8. Sorting with ORDER BY
9. Limiting Results
10. NULL Handling
11. Interview Questions and Problems
12. Tricks and Traps

---

## 1. What is a Database?

A database is an organized collection of structured data stored electronically and managed by a Database Management System (DBMS). A Relational Database Management System (RDBMS) organizes data into tables composed of rows and columns, enforcing relationships between tables through keys and constraints.

Common RDBMS implementations: PostgreSQL, MySQL, Microsoft SQL Server, Oracle Database, SQLite.

---

## 2. Relational vs Non-Relational Databases

### Relational Databases (SQL)

- Data is stored in tables with predefined schemas.
- Relationships are enforced via foreign keys.
- ACID compliance (Atomicity, Consistency, Isolation, Durability) ensures data integrity.
- Best suited for structured data with complex queries.

### Non-Relational Databases (NoSQL)

- Document stores (MongoDB), key-value stores (Redis), column-family (Cassandra), graph databases (Neo4j).
- Flexible schemas, horizontal scalability.
- Sacrifices some ACID guarantees for performance and flexibility.
- Best suited for unstructured or semi-structured data at scale.

### When to use SQL

Use SQL when data integrity, complex joins, and ad hoc querying matter. Use NoSQL when you need horizontal scaling, flexible schemas, or high write throughput.

---

## 3. The Relational Model

### Tables, Rows, and Columns

- A **table** (relation) is a set of rows with a fixed set of columns.
- A **row** (tuple) is a single record.
- A **column** (attribute) is a single field across all rows.

### Keys

| Key Type | Definition |
|---|---|
| Primary Key | Uniquely identifies each row in a table. Cannot be NULL. |
| Foreign Key | A column that references the primary key of another table. Enforces referential integrity. |
| Candidate Key | Any column or set of columns that could serve as a primary key. |
| Composite Key | A primary key composed of two or more columns. |
| Surrogate Key | An artificially generated key (e.g., auto-incremented integer or UUID). |
| Natural Key | A key derived from real-world data (e.g., SSN, email). |

### Constraints

```sql
CREATE TABLE employees (
    employee_id   INT           PRIMARY KEY,
    email         VARCHAR(255)  UNIQUE NOT NULL,
    department_id INT           REFERENCES departments(department_id),
    salary        DECIMAL(10,2) CHECK (salary > 0),
    hire_date     DATE          DEFAULT CURRENT_DATE
);
```

| Constraint | Purpose |
|---|---|
| NOT NULL | Prevents NULL values in a column. |
| UNIQUE | Ensures all values in a column are distinct. |
| PRIMARY KEY | Combines NOT NULL and UNIQUE; one per table. |
| FOREIGN KEY | Enforces referential integrity between tables. |
| CHECK | Enforces a condition on column values. |
| DEFAULT | Provides a default value when none is supplied. |

---

## 4. SQL Sublanguages

SQL is divided into five sublanguages:

| Sublanguage | Full Name | Commands |
|---|---|---|
| DDL | Data Definition Language | CREATE, ALTER, DROP, TRUNCATE, RENAME |
| DML | Data Manipulation Language | SELECT, INSERT, UPDATE, DELETE |
| DCL | Data Control Language | GRANT, REVOKE |
| TCL | Transaction Control Language | COMMIT, ROLLBACK, SAVEPOINT |
| DQL | Data Query Language | SELECT (sometimes separated from DML) |

---

## 5. Data Types

### Numeric Types

| Type | Description |
|---|---|
| INT / INTEGER | Whole numbers (-2,147,483,648 to 2,147,483,647) |
| BIGINT | Larger whole numbers |
| SMALLINT | Smaller range whole numbers |
| DECIMAL(p,s) / NUMERIC(p,s) | Exact fixed-point numbers (p = precision, s = scale) |
| FLOAT / REAL | Approximate floating-point numbers |

### String Types

| Type | Description |
|---|---|
| CHAR(n) | Fixed-length string, always n characters (padded with spaces) |
| VARCHAR(n) | Variable-length string, up to n characters |
| TEXT | Variable-length string of unlimited length |

### Date and Time Types

| Type | Description |
|---|---|
| DATE | Calendar date (YYYY-MM-DD) |
| TIME | Time of day (HH:MM:SS) |
| TIMESTAMP | Date and time combined |
| INTERVAL | A span of time |

### CHAR vs VARCHAR

`CHAR(10)` storing the value `'SQL'` occupies exactly 10 bytes (padded). `VARCHAR(10)` storing `'SQL'` occupies 3 bytes plus overhead. Use `CHAR` for fixed-length values (e.g., country codes). Use `VARCHAR` for variable-length values.

---

## 6. The SELECT Statement

The SELECT statement is the primary means of querying data.

### Syntax and Logical Processing Order

SQL is written in a specific order but executed in a different logical order:

**Written order:**
```
SELECT -> FROM -> WHERE -> GROUP BY -> HAVING -> ORDER BY -> LIMIT
```

**Logical execution order:**
```
FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT
```

This distinction is critical. Column aliases defined in SELECT are not available in WHERE because WHERE is evaluated before SELECT.

### Basic SELECT

```sql
-- Select all columns
SELECT * FROM employees;

-- Select specific columns
SELECT employee_id, first_name, last_name, salary
FROM employees;

-- Column aliases
SELECT
    first_name || ' ' || last_name AS full_name,
    salary * 12                    AS annual_salary
FROM employees;

-- Selecting literal values and expressions
SELECT
    1 + 1          AS result,
    'hello'        AS greeting,
    CURRENT_DATE   AS today;

-- DISTINCT: removes duplicate rows from result
SELECT DISTINCT department_id FROM employees;
```

### SELECT Without FROM

In PostgreSQL and SQL Server you can select without a table:
```sql
SELECT 2 + 2 AS result;
SELECT CURRENT_TIMESTAMP AS now;
```

---

## 7. Filtering with WHERE

WHERE filters rows before any aggregation occurs.

### Comparison Operators

```sql
SELECT * FROM employees WHERE salary > 50000;
SELECT * FROM employees WHERE salary >= 50000;
SELECT * FROM employees WHERE department_id = 3;
SELECT * FROM employees WHERE department_id != 3;  -- or <>
SELECT * FROM employees WHERE last_name = 'Smith';
```

### Logical Operators

```sql
-- AND: both conditions must be true
SELECT * FROM employees
WHERE salary > 50000 AND department_id = 3;

-- OR: at least one condition must be true
SELECT * FROM employees
WHERE department_id = 3 OR department_id = 5;

-- NOT: negates a condition
SELECT * FROM employees
WHERE NOT department_id = 3;
```

### BETWEEN

```sql
-- Inclusive on both ends
SELECT * FROM employees
WHERE salary BETWEEN 40000 AND 80000;

-- Equivalent to:
SELECT * FROM employees
WHERE salary >= 40000 AND salary <= 80000;
```

### IN

```sql
SELECT * FROM employees
WHERE department_id IN (1, 3, 5);

-- NOT IN: be careful with NULLs (see Section 10)
SELECT * FROM employees
WHERE department_id NOT IN (1, 3, 5);
```

### LIKE and Pattern Matching

```sql
-- % matches zero or more characters
SELECT * FROM employees WHERE last_name LIKE 'S%';    -- starts with S
SELECT * FROM employees WHERE last_name LIKE '%son';  -- ends with son
SELECT * FROM employees WHERE last_name LIKE '%ar%';  -- contains ar

-- _ matches exactly one character
SELECT * FROM employees WHERE last_name LIKE '_mith'; -- 5-letter names ending in mith

-- ILIKE (PostgreSQL): case-insensitive LIKE
SELECT * FROM employees WHERE last_name ILIKE 'smith';
```

---

## 8. Sorting with ORDER BY

```sql
-- Ascending (default)
SELECT * FROM employees ORDER BY salary;
SELECT * FROM employees ORDER BY salary ASC;

-- Descending
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple columns
SELECT * FROM employees
ORDER BY department_id ASC, salary DESC;

-- Order by column position (valid but avoid in production code)
SELECT first_name, last_name, salary
FROM employees
ORDER BY 3 DESC;
```

### NULL Ordering

By default, NULL values sort last in ascending order in most databases. You can control this:

```sql
-- PostgreSQL
SELECT * FROM employees ORDER BY salary ASC NULLS FIRST;
SELECT * FROM employees ORDER BY salary ASC NULLS LAST;
```

---

## 9. Limiting Results

```sql
-- PostgreSQL, MySQL, SQLite
SELECT * FROM employees ORDER BY salary DESC LIMIT 10;

-- SQL Server
SELECT TOP 10 * FROM employees ORDER BY salary DESC;

-- Oracle (pre-12c)
SELECT * FROM employees WHERE ROWNUM <= 10 ORDER BY salary DESC;

-- SQL Standard (ANSI, also PostgreSQL, SQL Server 2012+)
SELECT * FROM employees ORDER BY salary DESC FETCH FIRST 10 ROWS ONLY;

-- Pagination with OFFSET
SELECT * FROM employees
ORDER BY employee_id
LIMIT 10 OFFSET 20;  -- Returns rows 21 through 30
```

---

## 10. NULL Handling

NULL represents an unknown or missing value. NULL is not a value; it is the absence of a value.

### NULL Arithmetic

Any arithmetic operation involving NULL produces NULL:
```sql
SELECT 5 + NULL;     -- NULL
SELECT NULL * 100;   -- NULL
SELECT NULL = NULL;  -- NULL (not TRUE)
SELECT NULL != NULL; -- NULL (not TRUE)
```

### Testing for NULL

```sql
-- Correct
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE manager_id IS NOT NULL;

-- Wrong: these always return no rows
SELECT * FROM employees WHERE manager_id = NULL;
SELECT * FROM employees WHERE manager_id != NULL;
```

### COALESCE

Returns the first non-NULL value in the list:
```sql
SELECT
    employee_id,
    COALESCE(commission_pct, 0) AS commission
FROM employees;

SELECT
    COALESCE(preferred_name, first_name, 'Unknown') AS display_name
FROM employees;
```

### NULLIF

Returns NULL if two expressions are equal, otherwise returns the first expression:
```sql
-- Prevent division by zero
SELECT total_sales / NULLIF(num_transactions, 0) AS avg_sale
FROM sales_summary;
```

### CASE with NULL

```sql
SELECT
    employee_id,
    CASE
        WHEN commission_pct IS NULL THEN 'No Commission'
        WHEN commission_pct < 0.1   THEN 'Low Commission'
        ELSE 'Standard Commission'
    END AS commission_tier
FROM employees;
```

---

## 11. Interview Questions and Problems

### Theory Questions

**Q: What is the difference between DELETE, TRUNCATE, and DROP?**

DELETE is a DML statement that removes rows one at a time, can be filtered with WHERE, fires row-level triggers, and can be rolled back within a transaction. TRUNCATE is a DDL statement that removes all rows by deallocating data pages, is faster, cannot be filtered, does not fire row-level triggers, and in most databases cannot be rolled back. DROP is a DDL statement that removes the entire table structure and all its data permanently.

**Q: What is the difference between CHAR and VARCHAR?**

CHAR is fixed-length and always occupies n bytes regardless of the actual stored value. VARCHAR is variable-length and occupies only as many bytes as the actual value (plus a small length overhead). CHAR is slightly faster for fixed-length data (e.g., ISO country codes, fixed codes). VARCHAR saves storage for variable data.

**Q: Explain the logical order of SQL query execution.**

FROM, then JOIN and ON, then WHERE, then GROUP BY, then aggregate functions, then HAVING, then SELECT (including expressions and aliases), then DISTINCT, then ORDER BY, then LIMIT/OFFSET.

**Q: Why can't you reference a column alias from SELECT in the WHERE clause?**

Because WHERE is evaluated before SELECT in the logical processing order. The alias does not yet exist when WHERE is processed. You can use a subquery or CTE to work around this.

**Q: What is the difference between UNIQUE and PRIMARY KEY?**

A PRIMARY KEY is always NOT NULL and there can be only one per table. A UNIQUE constraint allows NULL values (multiple NULLs are typically permitted since NULL != NULL) and a table can have multiple UNIQUE constraints.

**Q: Can a table have multiple NULL values in a UNIQUE column?**

Yes in most databases (PostgreSQL, SQL Server, MySQL). A NULL value is considered distinct from all other values including other NULLs, so multiple NULLs can exist in a UNIQUE column. Oracle treats NULLs differently and allows multiple NULLs in a UNIQUE column as well.

**Q: What is a composite key and when would you use one?**

A composite key is a primary key made up of two or more columns. You use it when no single column uniquely identifies a row but a combination of columns does. For example, an `order_items` table might use `(order_id, product_id)` as a composite key.

### Coding Problems

**Problem 1:** Write a query to retrieve all employees with a salary greater than $70,000, ordered by salary descending, showing only the top 5.

```sql
SELECT employee_id, first_name, last_name, salary
FROM employees
WHERE salary > 70000
ORDER BY salary DESC
LIMIT 5;
```

**Problem 2:** Write a query to find all employees whose last name starts with 'A' or 'B'.

```sql
SELECT employee_id, first_name, last_name
FROM employees
WHERE last_name LIKE 'A%' OR last_name LIKE 'B%';

-- Alternative using BETWEEN on characters
SELECT employee_id, first_name, last_name
FROM employees
WHERE last_name >= 'A' AND last_name < 'C';
```

**Problem 3:** Write a query to display employee names and their salaries. Replace any NULL salary with 0.

```sql
SELECT
    first_name,
    last_name,
    COALESCE(salary, 0) AS salary
FROM employees;
```

**Problem 4:** Write a query to find all employees who do not have a manager.

```sql
SELECT employee_id, first_name, last_name
FROM employees
WHERE manager_id IS NULL;
```

**Problem 5 (Trap):** What does the following query return?

```sql
SELECT * FROM employees WHERE department_id NOT IN (SELECT department_id FROM departments WHERE location = 'Remote');
```

If any row in the subquery returns NULL for `department_id`, the entire `NOT IN` returns no rows. This is a critical NULL trap. The safe alternative is:

```sql
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d
    WHERE d.location = 'Remote'
    AND d.department_id = e.department_id
);
```

---

## 12. Tricks and Traps

### The NULL in NOT IN Trap

```sql
-- If values contains a NULL, this returns zero rows even when you expect matches
SELECT * FROM employees WHERE department_id NOT IN (1, 2, NULL);
-- Internally: department_id != 1 AND department_id != 2 AND department_id != NULL
-- The last condition evaluates to NULL (unknown), making the whole expression NULL
```

### DISTINCT on Multiple Columns

```sql
-- DISTINCT applies to the entire row, not just one column
SELECT DISTINCT department_id, job_id FROM employees;
-- Returns unique (department_id, job_id) pairs, not just unique department_ids
```

### ORDER BY Without LIMIT is Not Guaranteed

The order of rows in a SELECT result without ORDER BY is undefined. Do not rely on insertion order. Even with ORDER BY, if the sort key is not unique, tied rows may appear in any order between executions.

### Implicit Type Casting Pitfalls

```sql
-- This may work due to implicit casting but is fragile and slow (index unusable)
SELECT * FROM employees WHERE employee_id = '123';  -- employee_id is INT
-- Always match the literal type to the column type
SELECT * FROM employees WHERE employee_id = 123;
```

### String Comparison is Case-Sensitive in Most Databases

```sql
-- In PostgreSQL and MySQL (default), 'Smith' != 'smith' in LIKE
-- Use ILIKE (PostgreSQL) or LOWER() for case-insensitive matching
SELECT * FROM employees WHERE LOWER(last_name) = 'smith';
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Primary Key | Uniquely identifies rows; cannot be NULL |
| NULL | Unknown value; use IS NULL / IS NOT NULL to test |
| Logical execution order | FROM > WHERE > GROUP BY > HAVING > SELECT > ORDER BY |
| COALESCE | Returns first non-NULL argument |
| NOT IN with NULL | Always returns empty set; use NOT EXISTS instead |
| CHAR vs VARCHAR | Fixed-length vs variable-length strings |
| DISTINCT | Applies to all selected columns combined |

---

*Day 1 of 7 | Next: Joins and Relationships*
