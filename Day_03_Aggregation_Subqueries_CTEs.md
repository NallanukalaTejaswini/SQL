# Day 3 - Aggregation, Grouping, Subqueries, and CTEs

---

## Table of Contents

1. Aggregate Functions
2. GROUP BY
3. HAVING
4. Subqueries
5. Correlated Subqueries
6. Common Table Expressions (CTEs)
7. Recursive CTEs
8. Interview Questions and Problems
9. Tricks and Traps

---

## 1. Aggregate Functions

Aggregate functions compute a single result from a set of rows. They collapse many rows into a single summary value.

### Core Aggregate Functions

```sql
-- COUNT: number of rows
SELECT COUNT(*)          FROM employees;           -- All rows including NULLs
SELECT COUNT(salary)     FROM employees;           -- Rows where salary is NOT NULL
SELECT COUNT(DISTINCT department_id) FROM employees;  -- Distinct non-NULL values

-- SUM: total of numeric values (ignores NULLs)
SELECT SUM(salary) AS total_payroll FROM employees;

-- AVG: arithmetic mean (ignores NULLs)
SELECT AVG(salary) AS average_salary FROM employees;

-- MIN and MAX: smallest and largest values (ignores NULLs)
SELECT MIN(salary) AS lowest_salary, MAX(salary) AS highest_salary FROM employees;
SELECT MIN(hire_date) AS earliest_hire FROM employees;

-- STRING_AGG (PostgreSQL) / GROUP_CONCAT (MySQL): concatenate strings
SELECT
    department_id,
    STRING_AGG(first_name, ', ' ORDER BY first_name) AS employee_list
FROM employees
GROUP BY department_id;
```

### COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)

| Expression | What it counts |
|---|---|
| COUNT(*) | All rows, including NULLs in any column |
| COUNT(column) | Rows where column is NOT NULL |
| COUNT(DISTINCT column) | Distinct non-NULL values in column |

```sql
-- Example revealing the difference
SELECT
    COUNT(*)                     AS total_rows,
    COUNT(salary)                AS rows_with_salary,
    COUNT(DISTINCT department_id) AS unique_departments
FROM employees;
```

---

## 2. GROUP BY

GROUP BY divides rows into groups and applies an aggregate function to each group.

### The Golden Rule of GROUP BY

Every column in the SELECT list must either be:
1. Listed in the GROUP BY clause, OR
2. Used inside an aggregate function.

Violating this rule is a syntax error in standard SQL (and in PostgreSQL, SQL Server, Oracle). MySQL in some modes may allow it but returns unpredictable results.

```sql
-- Headcount and average salary per department
SELECT
    department_id,
    COUNT(*)     AS headcount,
    AVG(salary)  AS avg_salary,
    SUM(salary)  AS total_payroll
FROM employees
GROUP BY department_id;

-- Multiple grouping columns
SELECT
    department_id,
    job_title,
    COUNT(*) AS headcount
FROM employees
GROUP BY department_id, job_title
ORDER BY department_id, headcount DESC;
```

### ROLLUP and CUBE

ROLLUP generates subtotals and a grand total:

```sql
SELECT
    department_id,
    job_title,
    SUM(salary) AS total_salary
FROM employees
GROUP BY ROLLUP(department_id, job_title);
-- Produces: (department_id, job_title) subtotals
--           (department_id) subtotals
--           () grand total
```

CUBE generates subtotals for all possible combinations:

```sql
SELECT
    department_id,
    job_title,
    SUM(salary) AS total_salary
FROM employees
GROUP BY CUBE(department_id, job_title);
-- All combinations: (dept, job), (dept), (job), ()
```

GROUPING SETS gives you explicit control over which groupings to compute:

```sql
SELECT
    department_id,
    job_title,
    SUM(salary) AS total_salary
FROM employees
GROUP BY GROUPING SETS (
    (department_id, job_title),
    (department_id),
    ()
);
```

---

## 3. HAVING

HAVING filters groups after aggregation. It is to GROUP BY what WHERE is to individual rows.

```sql
-- Departments with more than 10 employees
SELECT
    department_id,
    COUNT(*) AS headcount
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 10;

-- Departments where average salary exceeds $60,000
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 60000
ORDER BY avg_salary DESC;
```

### WHERE vs HAVING

```sql
-- WHERE filters rows BEFORE grouping (cannot use aggregate functions)
-- HAVING filters groups AFTER grouping (can use aggregate functions)

-- Find departments where the average salary of full-time employees exceeds $60,000
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
WHERE employment_type = 'Full-Time'   -- Filters rows before grouping
GROUP BY department_id
HAVING AVG(salary) > 60000;           -- Filters groups after aggregation
```

Use WHERE to filter rows when possible. HAVING requires the aggregation to be computed first, which is more expensive.

---

## 4. Subqueries

A subquery is a SELECT statement nested inside another SQL statement. Subqueries can appear in the SELECT, FROM, WHERE, and HAVING clauses.

### Scalar Subquery (Returns One Row, One Column)

```sql
-- Employees who earn more than the company average
SELECT employee_id, first_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Difference from average salary for each employee
SELECT
    employee_id,
    first_name,
    salary,
    salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees;
```

### Row Subquery (Returns One Row, Multiple Columns)

```sql
-- Employees with the same department and salary as employee 101
SELECT employee_id, first_name
FROM employees
WHERE (department_id, salary) = (
    SELECT department_id, salary
    FROM employees
    WHERE employee_id = 101
);
```

### Table Subquery in FROM Clause (Derived Table / Inline View)

```sql
-- Average salary by department, then average of those averages
SELECT AVG(dept_avg) AS avg_of_avgs
FROM (
    SELECT department_id, AVG(salary) AS dept_avg
    FROM employees
    GROUP BY department_id
) AS dept_salaries;
```

### Subquery in SELECT Clause

```sql
-- For each department, show its headcount alongside each employee
SELECT
    e.employee_id,
    e.first_name,
    e.department_id,
    (SELECT COUNT(*) FROM employees e2
     WHERE e2.department_id = e.department_id) AS dept_headcount
FROM employees e;
```

Note: Scalar subqueries in SELECT are evaluated once per row. For large datasets, window functions are more efficient.

### EXISTS and NOT EXISTS

EXISTS tests whether a subquery returns any rows. It short-circuits as soon as the first row is found.

```sql
-- Departments that have at least one employee
SELECT department_id, department_name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM employees e
    WHERE e.department_id = d.department_id
);

-- Departments with no employees
SELECT department_id, department_name
FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e
    WHERE e.department_id = d.department_id
);
```

EXISTS is generally faster than IN for large subquery results because it short-circuits. The value selected inside EXISTS (here `1`) is irrelevant since only existence is tested.

---

## 5. Correlated Subqueries

A correlated subquery references a column from the outer query. It is evaluated once per row of the outer query.

```sql
-- Employees who earn more than the average salary in their own department
SELECT e.employee_id, e.first_name, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id  -- Correlated: references outer e
);

-- The second highest salary in each department (without window functions)
SELECT e.employee_id, e.first_name, e.salary, e.department_id
FROM employees e
WHERE 1 = (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
    AND e2.salary > e.salary
);
-- Rows where exactly one distinct salary is greater (making this the second highest)
```

Correlated subqueries can be slow at scale because they execute for every outer row. Window functions are usually the better alternative.

---

## 6. Common Table Expressions (CTEs)

A CTE is a named temporary result set defined before the main query using the WITH keyword. CTEs improve readability and allow you to reference the same subquery multiple times.

### Basic CTE Syntax

```sql
WITH dept_stats AS (
    SELECT
        department_id,
        COUNT(*)    AS headcount,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT
    d.department_name,
    ds.headcount,
    ds.avg_salary
FROM departments d
JOIN dept_stats ds ON d.department_id = ds.department_id
WHERE ds.headcount > 5
ORDER BY ds.avg_salary DESC;
```

### Multiple CTEs

```sql
WITH
high_earners AS (
    SELECT employee_id, first_name, salary, department_id
    FROM employees
    WHERE salary > 80000
),
dept_totals AS (
    SELECT department_id, SUM(salary) AS total_payroll
    FROM employees
    GROUP BY department_id
)
SELECT
    he.first_name,
    he.salary,
    dt.total_payroll,
    ROUND(he.salary / dt.total_payroll * 100, 2) AS pct_of_dept_payroll
FROM high_earners he
JOIN dept_totals dt ON he.department_id = dt.department_id;
```

### CTE vs Derived Table vs Subquery

| Feature | Subquery / Derived Table | CTE |
|---|---|---|
| Readability | Low for complex queries | High |
| Reuse in same query | Must repeat or use self-join | Reference by name |
| Recursion support | No | Yes |
| Materialization | Depends on optimizer | Depends on optimizer (hint available) |

---

## 7. Recursive CTEs

A recursive CTE references itself. It consists of an anchor member and a recursive member joined by UNION ALL. Essential for hierarchical and graph traversal.

### Anatomy

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor member: the base case (non-recursive)
    SELECT ...

    UNION ALL

    -- Recursive member: references the CTE itself
    SELECT ... FROM cte_name WHERE <termination_condition>
)
SELECT * FROM cte_name;
```

### Traversing an Employee Hierarchy

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: start with the top-level employee (no manager)
    SELECT
        employee_id,
        first_name,
        manager_id,
        0 AS level,
        first_name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: find direct reports of each employee in the CTE
    SELECT
        e.employee_id,
        e.first_name,
        e.manager_id,
        oc.level + 1,
        oc.path || ' > ' || e.first_name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT
    employee_id,
    REPEAT('  ', level) || first_name AS indented_name,
    level,
    path
FROM org_chart
ORDER BY path;
```

### Generating a Number Series

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 100
)
SELECT n FROM nums;
```

### Termination Warning

A recursive CTE without a proper termination condition will loop infinitely. Always include a WHERE condition that ensures the recursion eventually stops. Most databases impose a maximum recursion depth (default 100 in SQL Server).

---

## 8. Interview Questions and Problems

### Theory Questions

**Q: What is the difference between WHERE and HAVING?**

WHERE filters individual rows before grouping occurs. HAVING filters groups after GROUP BY and aggregation. WHERE cannot use aggregate functions; HAVING can. WHERE is evaluated earlier in query processing and is therefore more efficient when possible.

**Q: What does COUNT(*) vs COUNT(column) return differently?**

COUNT(*) counts all rows regardless of NULL values. COUNT(column) counts only rows where that column is NOT NULL. If a column has 5 NULL values in a 100-row table, COUNT(*) returns 100 and COUNT(column) returns 95.

**Q: What is a correlated subquery and what are its performance implications?**

A correlated subquery references a column from the outer query, causing it to be re-evaluated once for every row the outer query processes. This results in O(n) executions of the inner query. For large tables this is extremely slow. Window functions and joins almost always produce a more efficient execution plan.

**Q: What is the difference between a CTE and a derived table?**

Both are temporary named result sets. A derived table is defined inline in the FROM clause and can only be referenced once. A CTE is defined before the main query using WITH and can be referenced multiple times within the same query. CTEs also support recursion. In most databases, both are optimized similarly; neither is automatically materialized.

**Q: What is a recursive CTE? Give a real use case.**

A recursive CTE calls itself until a termination condition is met. Real use cases include traversing organizational hierarchies, graph paths (find all paths between two nodes), generating date series (every day between two dates), bill of materials explosion (a product made of parts made of sub-parts), and folder/directory tree traversal.

### Coding Problems

**Problem 1:** Find the department with the highest average salary.

```sql
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
ORDER BY avg_salary DESC
LIMIT 1;
```

**Problem 2:** Find employees who earn more than their department's average salary.

```sql
-- Method 1: Correlated subquery
SELECT e.employee_id, e.first_name, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary) FROM employees
    WHERE department_id = e.department_id
);

-- Method 2: CTE (more readable, avoids repeated subquery evaluation)
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT e.employee_id, e.first_name, e.salary, e.department_id
FROM employees e
JOIN dept_avg da ON e.department_id = da.department_id
WHERE e.salary > da.avg_salary;
```

**Problem 3:** For each department, list the top 2 highest-paid employees.

```sql
-- Method using correlated subquery (pre-window function approach)
SELECT e.department_id, e.employee_id, e.first_name, e.salary
FROM employees e
WHERE (
    SELECT COUNT(*)
    FROM employees e2
    WHERE e2.department_id = e.department_id
    AND e2.salary > e.salary
) < 2
ORDER BY department_id, salary DESC;
```

**Problem 4:** Find departments where all employees earn more than $50,000.

```sql
-- Method 1: Using HAVING with MIN
SELECT department_id
FROM employees
GROUP BY department_id
HAVING MIN(salary) > 50000;

-- Method 2: Using NOT EXISTS
SELECT DISTINCT department_id
FROM employees d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e
    WHERE e.department_id = d.department_id
    AND e.salary <= 50000
);
```

**Problem 5 (Hard):** Find the Nth highest salary from the employees table without using window functions.

```sql
-- Find the 3rd highest salary
SELECT DISTINCT salary
FROM employees e1
WHERE 2 = (
    SELECT COUNT(DISTINCT salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
)
ORDER BY salary DESC
LIMIT 1;
-- Generalize: replace 2 with N-1
```

**Problem 6:** Generate a date series for every day in 2024 using a recursive CTE.

```sql
WITH RECURSIVE date_series AS (
    SELECT DATE '2024-01-01' AS dt
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < DATE '2024-12-31'
)
SELECT dt FROM date_series;
```

---

## 9. Tricks and Traps

### Aggregating Without GROUP BY

If you use an aggregate function without GROUP BY, the entire table is treated as one group:

```sql
-- Returns a single row: max salary of all employees
SELECT MAX(salary) FROM employees;

-- This is an error in strict SQL: you cannot mix aggregate and non-aggregate without GROUP BY
SELECT first_name, MAX(salary) FROM employees;  -- ERROR in PostgreSQL/SQL Server
```

### HAVING Without GROUP BY

HAVING without GROUP BY treats the entire result as a single group:

```sql
-- Returns the table if it has at least 100 rows, otherwise returns nothing
SELECT * FROM employees HAVING COUNT(*) > 100;  -- Valid in some databases
```

### COUNT and Empty Tables

```sql
-- On an empty table, COUNT(*) returns 0 (not NULL)
-- Other aggregates on empty tables return NULL
SELECT COUNT(*), SUM(salary), AVG(salary) FROM employees WHERE 1=0;
-- Result: 0, NULL, NULL
```

### Subquery vs JOIN Performance

Scalar subqueries in the SELECT clause execute once per output row. For thousands of rows, this can be catastrophically slow. Always consider replacing scalar subqueries with joins or window functions:

```sql
-- SLOW: Scalar subquery executes once per employee row
SELECT e.*, (SELECT department_name FROM departments d WHERE d.department_id = e.department_id) AS dept
FROM employees e;

-- FAST: Single join
SELECT e.*, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

### Aggregating NULL Values

```sql
-- AVG ignores NULLs, which can be misleading
-- If 50 out of 100 employees have NULL salary, AVG is computed over 50 rows
SELECT AVG(salary) FROM employees;
-- This is the average of employees WITH a salary, not of all employees

-- To include NULLs as zero in the average:
SELECT AVG(COALESCE(salary, 0)) FROM employees;
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| GROUP BY golden rule | Every non-aggregate SELECT column must appear in GROUP BY |
| WHERE vs HAVING | WHERE filters rows; HAVING filters aggregated groups |
| COUNT(*) vs COUNT(col) | COUNT(*) includes NULLs; COUNT(col) excludes them |
| EXISTS | Short-circuits; the selected value inside doesn't matter |
| Correlated subquery | Executes per outer row; slow at scale; prefer window functions |
| CTE | Named temporary result set; supports recursion; reusable in same query |

---

*Day 3 of 7 | Previous: Joins | Next: Window Functions*
