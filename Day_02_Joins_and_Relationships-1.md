# Day 2 - Joins, Relationships, and Set Operations

---

## Table of Contents

1. Understanding Joins
2. INNER JOIN
3. LEFT JOIN (LEFT OUTER JOIN)
4. RIGHT JOIN (RIGHT OUTER JOIN)
5. FULL OUTER JOIN
6. CROSS JOIN
7. SELF JOIN
8. Multiple Joins
9. Set Operations: UNION, INTERSECT, EXCEPT
10. Interview Questions and Problems
11. Tricks and Traps

---

## 1. Understanding Joins

A JOIN combines rows from two or more tables based on a related column. Understanding joins deeply is non-negotiable for SQL proficiency. Every join type answers a different business question.

### Sample Schema for This Section

```sql
CREATE TABLE departments (
    department_id   INT PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL,
    location        VARCHAR(100)
);

CREATE TABLE employees (
    employee_id   INT PRIMARY KEY,
    first_name    VARCHAR(50),
    last_name     VARCHAR(50),
    department_id INT REFERENCES departments(department_id),
    manager_id    INT REFERENCES employees(employee_id),
    salary        DECIMAL(10,2),
    hire_date     DATE
);

CREATE TABLE projects (
    project_id   INT PRIMARY KEY,
    project_name VARCHAR(100),
    department_id INT
);

CREATE TABLE employee_projects (
    employee_id INT REFERENCES employees(employee_id),
    project_id  INT REFERENCES projects(project_id),
    PRIMARY KEY (employee_id, project_id)
);
```

### Join Venn Diagram Mental Model

Visualize two sets: Table A and Table B.

- INNER JOIN: Only the intersection (rows that match in both).
- LEFT JOIN: All of A plus matching rows from B.
- RIGHT JOIN: All of B plus matching rows from A.
- FULL OUTER JOIN: All rows from both A and B.
- CROSS JOIN: Every row in A paired with every row in B (Cartesian product).

---

## 2. INNER JOIN

Returns only rows where the join condition is satisfied in both tables. Non-matching rows from either table are excluded.

```sql
-- Explicit INNER JOIN syntax
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;

-- INNER keyword is optional; JOIN alone implies INNER JOIN
SELECT
    e.employee_id,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;
```

Employees without a `department_id` (NULL) and departments with no employees are both excluded from the result.

---

## 3. LEFT JOIN (LEFT OUTER JOIN)

Returns all rows from the left table and matching rows from the right table. Where there is no match, columns from the right table appear as NULL.

```sql
-- All employees, including those not assigned to a department
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    d.department_name  -- NULL for employees with no department
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

### Finding Rows With No Match

One of the most common and important patterns in SQL:

```sql
-- Employees who are NOT assigned to any department
SELECT
    e.employee_id,
    e.first_name,
    e.last_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
WHERE d.department_id IS NULL;
```

This is functionally equivalent to `NOT IN` and `NOT EXISTS` but the LEFT JOIN / IS NULL approach can be faster depending on the query planner.

---

## 4. RIGHT JOIN (RIGHT OUTER JOIN)

Returns all rows from the right table and matching rows from the left table. Rarely used in practice because any RIGHT JOIN can be rewritten as a LEFT JOIN by swapping the table order.

```sql
-- All departments, including those with no employees
SELECT
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;

-- Equivalent using LEFT JOIN (preferred style)
SELECT
    e.first_name,
    e.last_name,
    d.department_name
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id;
```

---

## 5. FULL OUTER JOIN

Returns all rows from both tables. Where there is no match, NULLs fill the missing side.

```sql
SELECT
    e.employee_id,
    e.first_name,
    d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.department_id;
```

This returns:
- Employees with a matching department.
- Employees with no department (department columns are NULL).
- Departments with no employees (employee columns are NULL).

MySQL does not support FULL OUTER JOIN natively. Simulate it with UNION:

```sql
SELECT e.employee_id, e.first_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
UNION
SELECT e.employee_id, e.first_name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

---

## 6. CROSS JOIN

Returns the Cartesian product of two tables. Every row in the left table is paired with every row in the right table. No ON clause is used.

```sql
-- If employees has 100 rows and departments has 10 rows, result has 1000 rows
SELECT e.first_name, d.department_name
FROM employees e
CROSS JOIN departments d;
```

### Practical Uses of CROSS JOIN

```sql
-- Generate a calendar of all combinations of years and months
SELECT y.year_val, m.month_num
FROM (SELECT 2023 AS year_val UNION SELECT 2024 UNION SELECT 2025) y
CROSS JOIN (SELECT generate_series(1,12) AS month_num) m;

-- Generate test data or pairing tables
-- Find all possible employee-project assignments before filtering
```

---

## 7. SELF JOIN

A self join joins a table to itself. Used for hierarchical or comparative data within the same table.

```sql
-- Find each employee and their manager's name
SELECT
    e.employee_id,
    e.first_name     AS employee_name,
    m.first_name     AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

Why LEFT JOIN and not INNER JOIN? Because the top-level manager has no manager themselves (`manager_id IS NULL`). An INNER JOIN would exclude them from the result.

```sql
-- Find pairs of employees in the same department (avoid duplicates)
SELECT
    a.first_name AS employee_1,
    b.first_name AS employee_2,
    a.department_id
FROM employees a
JOIN employees b
    ON a.department_id = b.department_id
    AND a.employee_id < b.employee_id;
-- The < condition prevents (A,B) and (B,A) from both appearing
```

---

## 8. Multiple Joins

You can chain as many joins as needed. Each JOIN adds a new table to the query.

```sql
-- Employees, their departments, and their projects
SELECT
    e.first_name,
    e.last_name,
    d.department_name,
    p.project_name
FROM employees e
JOIN departments d        ON e.department_id = d.department_id
JOIN employee_projects ep ON e.employee_id   = ep.employee_id
JOIN projects p           ON ep.project_id   = p.project_id;
```

### Non-Equi Joins

A join condition does not have to use equality. Any boolean expression is valid:

```sql
-- Find employees whose salary is above the average salary in their department
-- (This is better done with window functions, but illustrates non-equi joins)
SELECT a.employee_id, a.salary, b.salary AS compared_salary
FROM employees a
JOIN employees b
    ON a.department_id = b.department_id
    AND a.salary > b.salary;
```

---

## 9. Set Operations: UNION, INTERSECT, EXCEPT

Set operations combine the results of two SELECT statements.

### Rules for All Set Operations

- Both queries must return the same number of columns.
- Corresponding columns must have compatible data types.
- Column names in the result come from the first query.

### UNION vs UNION ALL

```sql
-- UNION: removes duplicates (like SELECT DISTINCT on the combined result)
SELECT employee_id FROM employees_usa
UNION
SELECT employee_id FROM employees_uk;

-- UNION ALL: keeps all rows including duplicates (faster, no deduplication)
SELECT employee_id FROM employees_usa
UNION ALL
SELECT employee_id FROM employees_uk;
```

Prefer UNION ALL unless you specifically need deduplication. UNION performs an implicit sort to find duplicates, which is expensive.

### INTERSECT

Returns only rows that appear in both result sets:

```sql
-- Employees who worked in both 2022 AND 2023
SELECT employee_id FROM project_assignments WHERE project_year = 2022
INTERSECT
SELECT employee_id FROM project_assignments WHERE project_year = 2023;
```

### EXCEPT (MINUS in Oracle)

Returns rows from the first result set that do not appear in the second:

```sql
-- Employees in USA but not in UK
SELECT employee_id FROM employees_usa
EXCEPT
SELECT employee_id FROM employees_uk;
```

### Ordering Set Operation Results

ORDER BY must appear at the very end and applies to the final combined result:

```sql
SELECT employee_id, first_name FROM employees_usa
UNION ALL
SELECT employee_id, first_name FROM employees_uk
ORDER BY first_name;
```

---

## 10. Interview Questions and Problems

### Theory Questions

**Q: What is the difference between INNER JOIN and LEFT JOIN?**

INNER JOIN returns only rows where the join condition is satisfied in both tables. LEFT JOIN returns all rows from the left table regardless of whether a match exists in the right table; non-matching right-table columns appear as NULL.

**Q: When would you use a FULL OUTER JOIN?**

When you need to reconcile two data sources and see which records exist in one but not the other, as well as records that match. Common in data migration validation, master data reconciliation, and finding orphaned records in both directions simultaneously.

**Q: What is a Cartesian product and when is it dangerous?**

A Cartesian product is the result of joining two tables without a join condition (or an accidental cross join). If table A has 10,000 rows and table B has 10,000 rows, a Cartesian product returns 100,000,000 rows, which can crash a database session or cause significant performance degradation.

**Q: Can a JOIN condition use inequality?**

Yes. Any boolean expression is valid: `ON a.start_date <= b.end_date AND a.end_date >= b.start_date` for date range overlaps is a classic example of a non-equi join.

**Q: What is the difference between UNION and UNION ALL?**

UNION eliminates duplicate rows by performing an implicit deduplication step (similar to DISTINCT). UNION ALL retains all rows including duplicates and is faster because it does no deduplication. Always use UNION ALL if you know duplicates are impossible or are acceptable.

**Q: Explain the difference between INTERSECT and INNER JOIN.**

Both can return overlapping data but they operate differently. INTERSECT compares entire rows across two result sets. INNER JOIN matches rows based on specified column conditions and can return additional columns from both tables. INTERSECT is a set operation; INNER JOIN is a relational operation.

### Coding Problems

**Problem 1:** Find all departments that have no employees.

```sql
SELECT d.department_id, d.department_name
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
WHERE e.employee_id IS NULL;
```

**Problem 2:** For each employee, show their name and their manager's name. Include employees who have no manager.

```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name  AS employee_name,
    m.first_name || ' ' || m.last_name  AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

**Problem 3:** Find employees who are assigned to at least one project.

```sql
SELECT DISTINCT e.employee_id, e.first_name, e.last_name
FROM employees e
JOIN employee_projects ep ON e.employee_id = ep.employee_id;
```

**Problem 4:** Find employees who are not assigned to any project.

```sql
-- Method 1: LEFT JOIN / IS NULL
SELECT e.employee_id, e.first_name, e.last_name
FROM employees e
LEFT JOIN employee_projects ep ON e.employee_id = ep.employee_id
WHERE ep.employee_id IS NULL;

-- Method 2: NOT EXISTS (preferred for clarity)
SELECT e.employee_id, e.first_name, e.last_name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM employee_projects ep
    WHERE ep.employee_id = e.employee_id
);

-- Method 3: NOT IN (dangerous if subquery can return NULLs)
SELECT employee_id, first_name, last_name
FROM employees
WHERE employee_id NOT IN (SELECT employee_id FROM employee_projects);
```

**Problem 5 (Hard):** Find pairs of employees who share the same manager and are in the same department.

```sql
SELECT
    a.employee_id AS employee_1_id,
    a.first_name  AS employee_1_name,
    b.employee_id AS employee_2_id,
    b.first_name  AS employee_2_name,
    a.manager_id,
    a.department_id
FROM employees a
JOIN employees b
    ON a.manager_id    = b.manager_id
    AND a.department_id = b.department_id
    AND a.employee_id  < b.employee_id  -- Avoid duplicates and self-pairing
WHERE a.manager_id IS NOT NULL;
```

**Problem 6:** List all employees and all departments in a single result set. Include a column indicating whether the row is an employee or a department.

```sql
SELECT first_name AS name, 'Employee'   AS record_type FROM employees
UNION ALL
SELECT department_name, 'Department' AS record_type FROM departments
ORDER BY record_type, name;
```

---

## 11. Tricks and Traps

### The Accidental Cross Join

```sql
-- WRONG: Missing ON clause creates a Cartesian product
SELECT * FROM employees, departments;
-- With 1000 employees and 50 departments: 50,000 rows

-- CORRECT
SELECT * FROM employees e JOIN departments d ON e.department_id = d.department_id;
```

### Filtering After a LEFT JOIN

```sql
-- WRONG: This turns the LEFT JOIN into an INNER JOIN
-- Because WHERE filters out NULLs from the right side
SELECT e.*, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
WHERE d.location = 'New York';  -- Employees with no department are excluded

-- CORRECT: Move the filter into the ON clause to preserve unmatched rows
SELECT e.*, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
                       AND d.location = 'New York';
```

This is one of the most common and dangerous SQL mistakes in professional settings.

### Duplicate Rows from Joins on Non-Unique Columns

```sql
-- If employee_projects has multiple rows per employee, this returns duplicates
SELECT e.employee_id, e.first_name
FROM employees e
JOIN employee_projects ep ON e.employee_id = ep.employee_id;
-- Employee appears once per project assignment

-- Use DISTINCT only if you need one row per employee
SELECT DISTINCT e.employee_id, e.first_name
FROM employees e
JOIN employee_projects ep ON e.employee_id = ep.employee_id;
```

### UNION Column Naming

```sql
-- Column names come from the first SELECT statement
SELECT employee_id AS id, first_name AS name FROM employees
UNION ALL
SELECT department_id, department_name FROM departments;
-- Result columns are named: id, name (from the first SELECT)
```

### INTERSECT and EXCEPT on NULL Values

NULL equals NULL in set operations (unlike regular comparisons):
```sql
-- Two NULLs in the same column position are treated as equal by INTERSECT/EXCEPT
SELECT NULL INTERSECT SELECT NULL;  -- Returns one NULL row
```

---

## Summary

| Join Type | Returns |
|---|---|
| INNER JOIN | Only matching rows from both tables |
| LEFT JOIN | All left rows; NULLs for non-matching right rows |
| RIGHT JOIN | All right rows; NULLs for non-matching left rows |
| FULL OUTER JOIN | All rows from both; NULLs where no match |
| CROSS JOIN | Cartesian product of both tables |
| SELF JOIN | A table joined to itself |

| Set Operation | Returns |
|---|---|
| UNION | Combined rows, duplicates removed |
| UNION ALL | Combined rows including duplicates |
| INTERSECT | Only rows present in both results |
| EXCEPT | Rows in first result but not in second |

---

*Day 2 of 7 | Previous: Foundations | Next: Aggregation and Grouping*
