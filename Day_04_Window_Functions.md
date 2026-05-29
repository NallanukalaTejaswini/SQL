# Day 4 - Window Functions

---

## Table of Contents

1. What Are Window Functions?
2. The OVER Clause
3. PARTITION BY
4. ORDER BY Inside OVER
5. Ranking Functions: ROW_NUMBER, RANK, DENSE_RANK, NTILE
6. Offset Functions: LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE
7. Aggregate Window Functions
8. Window Frames: ROWS vs RANGE
9. Named Windows
10. Interview Questions and Problems
11. Tricks and Traps

---

## 1. What Are Window Functions?

A window function performs a calculation across a set of rows related to the current row — called the window — without collapsing those rows into a single output row. This is the fundamental distinction from GROUP BY aggregation.

With GROUP BY, 100 employee rows grouped by department might produce 5 rows (one per department). With a window function, you still get 100 rows, but each row carries additional computed information derived from its window (e.g., the department average salary alongside each individual salary).

Window functions are among the most powerful features in SQL. They replace dozens of self-joins and correlated subqueries with clean, readable, and performant code.

### Syntax Template

```sql
function_name(expression) OVER (
    [PARTITION BY partition_expression]
    [ORDER BY sort_expression [ASC|DESC]]
    [ROWS|RANGE frame_specification]
)
```

All three clauses inside OVER are optional depending on the function and use case.

---

## 2. The OVER Clause

The presence of OVER is what makes a function a window function. Without OVER, COUNT, SUM, AVG etc. are aggregate functions. With OVER, they become window functions.

```sql
-- Aggregate function: collapses rows, returns one row
SELECT COUNT(*) FROM employees;

-- Window function: does not collapse rows
SELECT
    employee_id,
    first_name,
    COUNT(*) OVER () AS total_employees
FROM employees;
-- Returns one row per employee with total_employees = 100 (or whatever total)
```

An empty OVER clause `OVER ()` means the window is the entire result set.

---

## 3. PARTITION BY

PARTITION BY divides the result set into partitions (groups). The window function is applied independently within each partition. Think of it as GROUP BY for window functions — except rows are not collapsed.

```sql
SELECT
    employee_id,
    first_name,
    department_id,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary,
    salary - AVG(salary) OVER (PARTITION BY department_id) AS diff_from_dept_avg,
    COUNT(*)    OVER (PARTITION BY department_id) AS dept_headcount,
    SUM(salary) OVER (PARTITION BY department_id) AS dept_total_payroll
FROM employees;
```

Each row retains its individual data while gaining department-level aggregates in additional columns — something that would require a self-join or correlated subquery without window functions.

---

## 4. ORDER BY Inside OVER

ORDER BY inside OVER defines the logical ordering within each partition. This is required for ranking and offset functions. For aggregate window functions, it activates the default frame (RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW), enabling running totals.

```sql
-- Running total of salary ordered by hire date
SELECT
    employee_id,
    first_name,
    hire_date,
    salary,
    SUM(salary) OVER (ORDER BY hire_date) AS running_total
FROM employees;

-- Running total reset per department
SELECT
    employee_id,
    first_name,
    department_id,
    hire_date,
    salary,
    SUM(salary) OVER (
        PARTITION BY department_id
        ORDER BY hire_date
    ) AS dept_running_total
FROM employees;
```

---

## 5. Ranking Functions

### ROW_NUMBER

Assigns a unique sequential integer to each row within the partition. No ties — two rows with equal values still get different numbers. The assignment among ties is arbitrary unless the ORDER BY is fully deterministic.

```sql
SELECT
    employee_id,
    first_name,
    department_id,
    salary,
    ROW_NUMBER() OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
    ) AS salary_rank
FROM employees;
```

### RANK

Assigns the same rank to tied rows. After a tie, the next rank skips the numbers used. (1, 2, 2, 4 — not 1, 2, 2, 3)

```sql
SELECT
    employee_id,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;
-- If two employees share rank 2, the next rank is 4
```

### DENSE_RANK

Like RANK but without gaps. Tied rows get the same rank and the next rank is always the previous rank plus one. (1, 2, 2, 3 — not 1, 2, 2, 4)

```sql
SELECT
    employee_id,
    salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_dense_rank
FROM employees;
```

### Comparison Table

```
Salaries: 90000, 80000, 80000, 70000

ROW_NUMBER:  1, 2, 3, 4
RANK:        1, 2, 2, 4
DENSE_RANK:  1, 2, 2, 3
```

### NTILE

Divides the ordered partition into n approximately equal buckets and assigns a bucket number to each row.

```sql
-- Divide employees into 4 salary quartiles
SELECT
    employee_id,
    first_name,
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS salary_quartile
FROM employees;
-- Quartile 1 = top 25%, Quartile 4 = bottom 25%
```

### Top-N Per Group Pattern

One of the most common interview problems and real-world tasks:

```sql
-- Top 3 earners per department
WITH ranked AS (
    SELECT
        employee_id,
        first_name,
        department_id,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS rn
    FROM employees
)
SELECT employee_id, first_name, department_id, salary
FROM ranked
WHERE rn <= 3;
```

Use ROW_NUMBER for strict top-N (no ties in the cut). Use RANK or DENSE_RANK if you want all tied values to qualify.

---

## 6. Offset Functions

Offset functions access a row at a specified offset from the current row within the window.

### LAG

Returns a value from a previous row within the partition.

```sql
-- Compare each month's sales to the previous month
SELECT
    sale_month,
    total_sales,
    LAG(total_sales, 1, 0) OVER (ORDER BY sale_month) AS prev_month_sales,
    total_sales - LAG(total_sales, 1, 0) OVER (ORDER BY sale_month) AS month_over_month
FROM monthly_sales;

-- Syntax: LAG(expression, offset, default_if_null)
-- offset defaults to 1; default_if_null replaces NULL when no prior row exists
```

### LEAD

Returns a value from a subsequent row within the partition.

```sql
-- Show next month's sales target alongside current
SELECT
    sale_month,
    total_sales,
    LEAD(total_sales, 1) OVER (ORDER BY sale_month) AS next_month_sales
FROM monthly_sales;
```

### FIRST_VALUE and LAST_VALUE

```sql
-- Salary difference from the highest earner in the same department
SELECT
    employee_id,
    first_name,
    department_id,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
    ) AS dept_max_salary,
    salary - FIRST_VALUE(salary) OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
    ) AS gap_from_top
FROM employees;

-- LAST_VALUE requires an explicit frame to behave as expected (see Section 8)
SELECT
    employee_id,
    salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS dept_min_salary
FROM employees;
```

### NTH_VALUE

Returns the value of the Nth row in the window:

```sql
-- Salary of the second highest earner in each department
SELECT
    employee_id,
    first_name,
    salary,
    NTH_VALUE(salary, 2) OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_highest_in_dept
FROM employees;
```

---

## 7. Aggregate Window Functions

All standard aggregate functions (SUM, COUNT, AVG, MIN, MAX) can be used as window functions with OVER.

```sql
SELECT
    employee_id,
    first_name,
    department_id,
    salary,
    -- Department-level aggregates alongside each row
    SUM(salary)  OVER (PARTITION BY department_id) AS dept_total,
    AVG(salary)  OVER (PARTITION BY department_id) AS dept_avg,
    MIN(salary)  OVER (PARTITION BY department_id) AS dept_min,
    MAX(salary)  OVER (PARTITION BY department_id) AS dept_max,
    COUNT(*)     OVER (PARTITION BY department_id) AS dept_count,
    -- Company-wide aggregates alongside each row
    SUM(salary)  OVER ()                            AS company_total,
    -- Percentage of company payroll
    ROUND(salary / SUM(salary) OVER () * 100, 2)   AS pct_of_payroll
FROM employees;
```

### Running Totals and Moving Averages

```sql
-- Cumulative sum (running total)
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY sale_date) AS cumulative_revenue
FROM daily_sales;

-- 7-day moving average
SELECT
    sale_date,
    daily_revenue,
    AVG(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_sales;
```

---

## 8. Window Frames: ROWS vs RANGE

A window frame defines which rows within the partition are included in the window for the current row. Frames are only meaningful when ORDER BY is specified inside OVER.

### Frame Syntax

```sql
ROWS  BETWEEN <start_bound> AND <end_bound>
RANGE BETWEEN <start_bound> AND <end_bound>
```

### Boundary Options

| Boundary | Meaning |
|---|---|
| UNBOUNDED PRECEDING | First row of the partition |
| N PRECEDING | N rows before the current row |
| CURRENT ROW | The current row |
| N FOLLOWING | N rows after the current row |
| UNBOUNDED FOLLOWING | Last row of the partition |

### ROWS vs RANGE

ROWS operates on physical row positions. RANGE operates on logical value ranges (treating tied rows as a group).

```sql
-- Default frame when ORDER BY is present:
-- RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- This means: all rows from start of partition to all rows with the same ORDER BY value

-- Explicit ROWS frame: exactly N physical rows
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_sales;

-- RANGE with ties: all rows sharing the same sale_date value are included
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY sale_date
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total_range
FROM daily_sales;
-- If multiple rows share the same sale_date, RANGE includes all of them
-- before advancing; ROWS treats them as distinct physical positions
```

### The LAST_VALUE Frame Trap

Without an explicit frame, LAST_VALUE returns the value of the current row (due to the default frame ending at CURRENT ROW), not the last row in the partition:

```sql
-- WRONG: returns current row's salary due to default frame
SELECT
    LAST_VALUE(salary) OVER (ORDER BY salary) AS last_salary
FROM employees;

-- CORRECT: extend frame to end of partition
SELECT
    LAST_VALUE(salary) OVER (
        ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_salary
FROM employees;
```

---

## 9. Named Windows

If you use the same window specification multiple times, define it once with a WINDOW clause:

```sql
SELECT
    employee_id,
    department_id,
    salary,
    ROW_NUMBER() OVER dept_window   AS row_num,
    RANK()       OVER dept_window   AS rnk,
    SUM(salary)  OVER dept_window   AS dept_total
FROM employees
WINDOW dept_window AS (
    PARTITION BY department_id
    ORDER BY salary DESC
);
-- Supported in PostgreSQL, MySQL 8+; not in SQL Server
```

---

## 10. Interview Questions and Problems

### Theory Questions

**Q: What is the difference between a window function and GROUP BY?**

GROUP BY collapses multiple rows into a single summary row per group, eliminating access to individual row data. A window function computes values across a set of related rows but preserves every individual row in the output. You can have both the individual salary and the department average in the same row with a window function; with GROUP BY you must choose one or use a join.

**Q: What is the difference between RANK and DENSE_RANK?**

Both assign the same rank to tied values. RANK leaves gaps after a tie: if two rows tie for rank 2, the next rank is 4. DENSE_RANK never leaves gaps: if two rows tie for rank 2, the next rank is 3. DENSE_RANK is preferred when you need to find the Nth distinct value.

**Q: What is the default window frame when ORDER BY is specified?**

RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW. This means the window includes all rows from the start of the partition through all rows that share the same ORDER BY value as the current row. This is why LAST_VALUE appears to not work correctly unless you override the frame.

**Q: When would you use ROW_NUMBER vs RANK vs DENSE_RANK?**

ROW_NUMBER: when you need a unique identifier per row (pagination, deduplication, top-N with no ties). RANK: when ties should share a rank and the gap matters (competition rankings where tied participants skip the next position). DENSE_RANK: when ties share a rank and you need the Nth distinct rank value (finding the 3rd highest salary).

**Q: What does PARTITION BY do in a window function?**

PARTITION BY divides the result set into independent groups (partitions). The window function resets and is computed independently within each partition, similar to how GROUP BY defines groups for aggregation.

### Coding Problems

**Problem 1:** Find the second highest salary in each department.

```sql
WITH ranked AS (
    SELECT
        employee_id,
        first_name,
        department_id,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS dr
    FROM employees
)
SELECT employee_id, first_name, department_id, salary
FROM ranked
WHERE dr = 2;
```

**Problem 2:** For each employee, calculate the salary as a percentage of the total department salary and total company salary.

```sql
SELECT
    employee_id,
    first_name,
    department_id,
    salary,
    ROUND(salary * 100.0 / SUM(salary) OVER (PARTITION BY department_id), 2) AS pct_of_dept,
    ROUND(salary * 100.0 / SUM(salary) OVER (),                            2) AS pct_of_company
FROM employees;
```

**Problem 3:** Identify employees whose salary is greater than the salary of the employee hired just before them (by hire date) within the same department.

```sql
WITH lagged AS (
    SELECT
        employee_id,
        first_name,
        department_id,
        hire_date,
        salary,
        LAG(salary) OVER (
            PARTITION BY department_id
            ORDER BY hire_date
        ) AS prev_hire_salary
    FROM employees
)
SELECT employee_id, first_name, department_id, salary, prev_hire_salary
FROM lagged
WHERE salary > prev_hire_salary;
```

**Problem 4:** For each row, calculate a 3-row moving average of daily_revenue.

```sql
SELECT
    sale_date,
    daily_revenue,
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_3d
FROM daily_sales;
```

**Problem 5 (Hard):** Deduplicate a table by keeping only the most recently hired employee per email address.

```sql
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY email
            ORDER BY hire_date DESC
        ) AS rn
    FROM employees
)
DELETE FROM employees
WHERE employee_id IN (
    SELECT employee_id FROM ranked WHERE rn > 1
);

-- Or to SELECT the deduplicated result:
SELECT * FROM ranked WHERE rn = 1;
```

**Problem 6 (Hard):** Calculate the cumulative percentage of total salary, ordered from highest to lowest earner.

```sql
SELECT
    employee_id,
    first_name,
    salary,
    ROUND(
        SUM(salary) OVER (ORDER BY salary DESC
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        * 100.0
        / SUM(salary) OVER (),
    2) AS cumulative_pct
FROM employees
ORDER BY salary DESC;
```

---

## 11. Tricks and Traps

### Window Functions Cannot Be Used in WHERE

Window functions are evaluated after WHERE in the logical processing order. To filter on a window function result, wrap it in a CTE or subquery:

```sql
-- ERROR: window functions not allowed in WHERE
SELECT * FROM employees WHERE ROW_NUMBER() OVER (ORDER BY salary) = 1;

-- CORRECT
WITH numbered AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM numbered WHERE rn = 1;
```

### PARTITION BY vs GROUP BY Do Not Interact

A window function PARTITION BY is completely independent of any GROUP BY in the same query. The GROUP BY collapses rows first; the window function then operates on the collapsed result.

### ORDER BY in OVER vs Query-Level ORDER BY

These are independent. ORDER BY inside OVER defines the window ordering for the function. ORDER BY at the query level defines the output sort order. They can be different.

```sql
SELECT
    employee_id,
    salary,
    SUM(salary) OVER (ORDER BY hire_date) AS running_total  -- window order: hire_date
FROM employees
ORDER BY salary DESC;  -- output order: salary
```

### ROWS BETWEEN and NULL in ORDER BY

If ORDER BY in OVER contains NULLs, they are included in CURRENT ROW's range for RANGE frames. This can cause unexpected results. Use ROWS instead of RANGE for physical row-based frames to avoid ambiguity.

---

## Summary

| Function | Purpose |
|---|---|
| ROW_NUMBER() | Unique sequential number per partition; no ties |
| RANK() | Rank with gaps after ties |
| DENSE_RANK() | Rank without gaps after ties |
| NTILE(n) | Divide into n equal buckets |
| LAG(col, n) | Value n rows before current row |
| LEAD(col, n) | Value n rows after current row |
| FIRST_VALUE(col) | First value in the window frame |
| LAST_VALUE(col) | Last value in the window frame (requires explicit frame) |
| SUM/AVG/COUNT OVER | Aggregate over window without collapsing rows |

---

*Day 4 of 7 | Previous: Aggregation and Subqueries | Next: Indexes and Query Optimization*
