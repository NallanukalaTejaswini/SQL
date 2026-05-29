# Day 7 - Advanced SQL and Interview Master Class

---

## Table of Contents

1. Pivoting and Conditional Aggregation
2. String Functions
3. Date and Time Functions
4. JSON in SQL
5. Stored Procedures and Functions
6. Triggers
7. Dynamic SQL
8. Gap and Island Problems
9. Advanced Interview Problems: All Difficulty Levels
10. Behavioral and System Design SQL Questions
11. Final Tricks and Reference Cheat Sheet

---

## 1. Pivoting and Conditional Aggregation

### Conditional Aggregation (CASE inside Aggregate)

The most portable and flexible way to pivot data. No vendor-specific syntax required.

```sql
-- Source: monthly_sales (year, month, product, revenue)
-- Goal: one row per year, one column per month

SELECT
    year,
    SUM(CASE WHEN month = 1  THEN revenue ELSE 0 END) AS jan,
    SUM(CASE WHEN month = 2  THEN revenue ELSE 0 END) AS feb,
    SUM(CASE WHEN month = 3  THEN revenue ELSE 0 END) AS mar,
    SUM(CASE WHEN month = 4  THEN revenue ELSE 0 END) AS apr,
    SUM(CASE WHEN month = 5  THEN revenue ELSE 0 END) AS may,
    SUM(CASE WHEN month = 6  THEN revenue ELSE 0 END) AS jun,
    SUM(CASE WHEN month = 7  THEN revenue ELSE 0 END) AS jul,
    SUM(CASE WHEN month = 8  THEN revenue ELSE 0 END) AS aug,
    SUM(CASE WHEN month = 9  THEN revenue ELSE 0 END) AS sep,
    SUM(CASE WHEN month = 10 THEN revenue ELSE 0 END) AS oct,
    SUM(CASE WHEN month = 11 THEN revenue ELSE 0 END) AS nov,
    SUM(CASE WHEN month = 12 THEN revenue ELSE 0 END) AS dec,
    SUM(revenue) AS annual_total
FROM monthly_sales
GROUP BY year
ORDER BY year;
```

### COUNT with CASE (Counting Subsets)

```sql
-- Count employees by status in a single query row
SELECT
    department_id,
    COUNT(*)                                            AS total,
    COUNT(CASE WHEN status = 'Active'   THEN 1 END)    AS active_count,
    COUNT(CASE WHEN status = 'Inactive' THEN 1 END)    AS inactive_count,
    COUNT(CASE WHEN salary > 80000      THEN 1 END)    AS high_earners,
    ROUND(AVG(CASE WHEN status = 'Active' THEN salary END), 2) AS avg_active_salary
FROM employees
GROUP BY department_id;
```

### SQL Server PIVOT Operator

```sql
SELECT year, [1] AS jan, [2] AS feb, [3] AS mar
FROM (
    SELECT year, month, revenue FROM monthly_sales
) AS src
PIVOT (
    SUM(revenue)
    FOR month IN ([1], [2], [3])
) AS pvt;
```

### UNPIVOT: Columns to Rows

```sql
-- Turn monthly columns back into rows (SQL Server)
SELECT year, month_name, revenue
FROM quarterly_summary
UNPIVOT (
    revenue FOR month_name IN (jan, feb, mar, apr)
) AS unpvt;

-- PostgreSQL equivalent using UNION ALL
SELECT year, 'jan' AS month, jan AS revenue FROM quarterly_summary
UNION ALL
SELECT year, 'feb', feb FROM quarterly_summary
UNION ALL
SELECT year, 'mar', mar FROM quarterly_summary;
```

---

## 2. String Functions

### Core String Functions

```sql
-- Length
SELECT LENGTH('Hello World');                -- 11 (PostgreSQL, MySQL)
SELECT LEN('Hello World');                   -- 11 (SQL Server)

-- Case conversion
SELECT UPPER('hello'), LOWER('HELLO');

-- Trimming
SELECT TRIM('  hello  ');                    -- 'hello'
SELECT LTRIM('  hello  ');                   -- 'hello  '
SELECT RTRIM('  hello  ');                   -- '  hello'
SELECT TRIM(BOTH 'x' FROM 'xxxhelloxxx');    -- 'hello' (PostgreSQL)

-- Substring extraction
SELECT SUBSTRING('Hello World', 1, 5);       -- 'Hello' (1-indexed)
SELECT LEFT('Hello World', 5);               -- 'Hello'
SELECT RIGHT('Hello World', 5);              -- 'World'

-- Search within string
SELECT POSITION('World' IN 'Hello World');   -- 7
SELECT STRPOS('Hello World', 'World');        -- 7 (PostgreSQL)
SELECT CHARINDEX('World', 'Hello World');    -- 7 (SQL Server)

-- Concatenation
SELECT CONCAT('Hello', ' ', 'World');        -- 'Hello World'
SELECT 'Hello' || ' ' || 'World';           -- PostgreSQL operator

-- Replace
SELECT REPLACE('Hello World', 'World', 'SQL');  -- 'Hello SQL'

-- Repeat
SELECT REPEAT('ab', 3);                      -- 'ababab'

-- Padding
SELECT LPAD('42', 5, '0');                   -- '00042'
SELECT RPAD('42', 5, '0');                   -- '42000'

-- Splitting and parsing
SELECT SPLIT_PART('a,b,c', ',', 2);          -- 'b' (PostgreSQL)
SELECT STRING_TO_ARRAY('a,b,c', ',');         -- {a,b,c} (PostgreSQL)
```

### Regular Expressions (PostgreSQL)

```sql
-- Match: ~ (case-sensitive), ~* (case-insensitive)
SELECT * FROM employees WHERE email ~ '^[a-z]+\.[a-z]+@company\.com$';

-- Replace
SELECT REGEXP_REPLACE('Phone: 555-1234', '[0-9]', 'X', 'g');  -- 'Phone: XXX-XXXX'

-- Extract
SELECT REGEXP_MATCHES('2024-06-15', '(\d{4})-(\d{2})-(\d{2})');
-- Returns: {2024,06,15}
```

---

## 3. Date and Time Functions

### Current Date and Time

```sql
SELECT CURRENT_DATE;        -- Date only: 2024-06-15
SELECT CURRENT_TIME;        -- Time only: 14:32:00
SELECT CURRENT_TIMESTAMP;   -- Date + time: 2024-06-15 14:32:00
SELECT NOW();               -- PostgreSQL: same as CURRENT_TIMESTAMP
SELECT GETDATE();           -- SQL Server equivalent
```

### Date Arithmetic

```sql
-- PostgreSQL
SELECT CURRENT_DATE + INTERVAL '30 days';
SELECT CURRENT_DATE - INTERVAL '1 year';
SELECT AGE(CURRENT_DATE, '1990-05-15');         -- Precise age: 34 years 1 mon 0 days
SELECT EXTRACT(YEAR FROM AGE(CURRENT_DATE, birth_date)) AS age_years FROM employees;

-- Difference between dates
SELECT CURRENT_DATE - hire_date AS days_employed FROM employees;

-- MySQL
SELECT DATE_ADD(CURRENT_DATE, INTERVAL 30 DAY);
SELECT DATEDIFF(CURRENT_DATE, hire_date) AS days_employed FROM employees;
```

### Extracting Parts

```sql
SELECT EXTRACT(YEAR    FROM hire_date) AS year   FROM employees;
SELECT EXTRACT(MONTH   FROM hire_date) AS month  FROM employees;
SELECT EXTRACT(DOW     FROM hire_date) AS weekday FROM employees;  -- 0=Sun, 6=Sat
SELECT EXTRACT(QUARTER FROM hire_date) AS quarter FROM employees;
SELECT EXTRACT(EPOCH   FROM hire_date) AS unix_ts FROM employees;  -- PostgreSQL
```

### Truncating Dates

```sql
-- Truncate to beginning of period
SELECT DATE_TRUNC('month',   CURRENT_TIMESTAMP);  -- 2024-06-01 00:00:00
SELECT DATE_TRUNC('year',    CURRENT_TIMESTAMP);  -- 2024-01-01 00:00:00
SELECT DATE_TRUNC('week',    CURRENT_TIMESTAMP);  -- Most recent Monday
SELECT DATE_TRUNC('quarter', CURRENT_TIMESTAMP);  -- 2024-04-01 00:00:00
```

### Formatting

```sql
-- PostgreSQL
SELECT TO_CHAR(hire_date, 'DD Mon YYYY')    FROM employees;  -- '15 Jun 2024'
SELECT TO_CHAR(salary, 'FM$999,999.00')     FROM employees;  -- '$75,000.00'

-- SQL Server
SELECT FORMAT(hire_date, 'dd MMM yyyy')     FROM employees;
SELECT CONVERT(VARCHAR, hire_date, 103);    -- DD/MM/YYYY style
```

---

## 4. JSON in SQL

### PostgreSQL JSON

```sql
-- Create table with JSON column
CREATE TABLE events (
    event_id   BIGINT PRIMARY KEY,
    event_type VARCHAR(50),
    payload    JSONB  -- JSONB: binary JSON, indexable, preferred over JSON
);

-- Insert JSON data
INSERT INTO events VALUES (1, 'purchase', '{"user_id": 101, "amount": 49.99, "items": ["book", "pen"]}');

-- Extract field: -> returns JSON, ->> returns text
SELECT payload -> 'user_id'    AS user_id_json  FROM events;  -- Returns JSON: 101
SELECT payload ->> 'user_id'   AS user_id_text  FROM events;  -- Returns text: '101'
SELECT payload ->> 'amount'    AS amount        FROM events;  -- '49.99'

-- Nested access
SELECT payload -> 'address' ->> 'city' AS city FROM events;

-- Array element
SELECT payload -> 'items' ->> 0 AS first_item FROM events;  -- 'book'

-- Filter on JSON value
SELECT * FROM events WHERE payload ->> 'user_id' = '101';
SELECT * FROM events WHERE (payload ->> 'amount')::DECIMAL > 40;

-- Check key existence
SELECT * FROM events WHERE payload ? 'user_id';          -- Has key 'user_id'
SELECT * FROM events WHERE payload ?| ARRAY['a','b'];    -- Has any of these keys
SELECT * FROM events WHERE payload ?& ARRAY['a','b'];    -- Has all of these keys

-- Update JSON field
UPDATE events
SET payload = jsonb_set(payload, '{amount}', '59.99')
WHERE event_id = 1;

-- Aggregate to JSON
SELECT json_agg(employee_id) AS ids FROM employees;
SELECT json_object_agg(department_id, department_name) FROM departments;

-- Expand JSON array to rows
SELECT event_id, jsonb_array_elements_text(payload->'items') AS item
FROM events;

-- Index JSONB
CREATE INDEX idx_events_user_id ON events USING GIN (payload);
CREATE INDEX idx_events_user_id_path ON events ((payload->>'user_id'));
```

---

## 5. Stored Procedures and Functions

### User-Defined Functions (PostgreSQL)

```sql
-- Scalar function: returns a single value
CREATE OR REPLACE FUNCTION calculate_annual_salary(
    monthly_salary DECIMAL,
    bonus_pct      DECIMAL DEFAULT 0.10
)
RETURNS DECIMAL
LANGUAGE SQL
IMMUTABLE  -- Same inputs always produce same output (enables caching and optimization)
AS $$
    SELECT monthly_salary * 12 * (1 + bonus_pct);
$$;

-- Usage
SELECT employee_id, calculate_annual_salary(salary, 0.15) AS annual_compensation
FROM employees;

-- Table-returning function
CREATE OR REPLACE FUNCTION get_dept_employees(p_dept_id INT)
RETURNS TABLE(employee_id INT, full_name TEXT, salary DECIMAL)
LANGUAGE SQL
AS $$
    SELECT employee_id, first_name || ' ' || last_name, salary
    FROM employees
    WHERE department_id = p_dept_id;
$$;

-- Usage
SELECT * FROM get_dept_employees(3);
```

### Stored Procedures (PostgreSQL 11+)

Stored procedures differ from functions in that they can manage transactions:

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    p_from_account INT,
    p_to_account   INT,
    p_amount       DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF (SELECT balance FROM accounts WHERE account_id = p_from_account) < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds in account %', p_from_account;
    END IF;

    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_from_account;
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_to_account;

    INSERT INTO transfer_log(from_acc, to_acc, amount, transferred_at)
    VALUES (p_from_account, p_to_account, p_amount, CURRENT_TIMESTAMP);

    COMMIT;
END;
$$;

-- Execute
CALL transfer_funds(1, 2, 500.00);
```

---

## 6. Triggers

A trigger is a procedure that automatically executes in response to specified database events (INSERT, UPDATE, DELETE) on a table.

```sql
-- Audit trigger: log every salary change
CREATE TABLE salary_audit (
    audit_id    SERIAL PRIMARY KEY,
    employee_id INT,
    old_salary  DECIMAL,
    new_salary  DECIMAL,
    changed_by  VARCHAR(100) DEFAULT CURRENT_USER,
    changed_at  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF OLD.salary IS DISTINCT FROM NEW.salary THEN
        INSERT INTO salary_audit(employee_id, old_salary, new_salary)
        VALUES (NEW.employee_id, OLD.salary, NEW.salary);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_salary_audit
AFTER UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION log_salary_change();
```

### Trigger Types

| Type | When it fires |
|---|---|
| BEFORE | Before the row is changed; can modify the row or cancel the operation |
| AFTER | After the row is changed; typically used for auditing and cascading actions |
| INSTEAD OF | Fires instead of the operation (used on views to make them updatable) |

---

## 7. Dynamic SQL

Dynamic SQL constructs and executes SQL statements at runtime. Use with extreme caution — always parameterize to prevent SQL injection.

```sql
-- PostgreSQL: EXECUTE with parameterized input
CREATE OR REPLACE FUNCTION get_table_count(p_table_name TEXT)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_count BIGINT;
    v_sql   TEXT;
BEGIN
    -- Validate table name to prevent injection (use quote_ident)
    v_sql := 'SELECT COUNT(*) FROM ' || quote_ident(p_table_name);
    EXECUTE v_sql INTO v_count;
    RETURN v_count;
END;
$$;

-- SQL Server
DECLARE @table_name NVARCHAR(100) = 'employees';
DECLARE @sql NVARCHAR(MAX) = N'SELECT COUNT(*) FROM ' + QUOTENAME(@table_name);
EXEC sp_executesql @sql;
```

---

## 8. Gap and Island Problems

Gap and island problems are a class of problems involving consecutive sequences with breaks (gaps) or consecutive groups (islands). They appear frequently in interviews and real analytics.

### Detecting Gaps in a Sequence

```sql
-- Find missing employee IDs in a sequence
SELECT
    a.employee_id + 1                AS gap_start,
    MIN(b.employee_id) - 1           AS gap_end
FROM employees a
JOIN employees b ON b.employee_id > a.employee_id
GROUP BY a.employee_id
HAVING MIN(b.employee_id) > a.employee_id + 1
ORDER BY gap_start;
```

### Finding Islands (Consecutive Groups)

```sql
-- Sessions data: find consecutive active days per user
-- user_id, activity_date

WITH numbered AS (
    SELECT
        user_id,
        activity_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date) AS rn
    FROM user_activity
),
grouped AS (
    SELECT
        user_id,
        activity_date,
        -- If dates are consecutive, date - rn is constant within an island
        activity_date - CAST(rn AS INT) * INTERVAL '1 day' AS group_key
    FROM numbered
)
SELECT
    user_id,
    MIN(activity_date) AS island_start,
    MAX(activity_date) AS island_end,
    COUNT(*)           AS consecutive_days
FROM grouped
GROUP BY user_id, group_key
ORDER BY user_id, island_start;
```

---

## 9. Advanced Interview Problems: All Difficulty Levels

### Easy Level

**E1: Write a query to find duplicate emails in an employees table.**

```sql
SELECT email, COUNT(*) AS occurrences
FROM employees
GROUP BY email
HAVING COUNT(*) > 1;
```

**E2: Find the Nth highest salary using DENSE_RANK.**

```sql
-- Replace 3 with any N
WITH ranked AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM employees
)
SELECT DISTINCT salary FROM ranked WHERE dr = 3;
```

**E3: Find employees hired in the last 90 days.**

```sql
SELECT * FROM employees
WHERE hire_date >= CURRENT_DATE - INTERVAL '90 days';
```

### Medium Level

**M1: Find customers who placed orders in every month of 2023.**

```sql
SELECT customer_id
FROM orders
WHERE order_date >= '2023-01-01' AND order_date < '2024-01-01'
GROUP BY customer_id
HAVING COUNT(DISTINCT EXTRACT(MONTH FROM order_date)) = 12;
```

**M2: Find departments where the highest salary is less than the average salary of the entire company.**

```sql
SELECT department_id, MAX(salary) AS dept_max
FROM employees
GROUP BY department_id
HAVING MAX(salary) < (SELECT AVG(salary) FROM employees);
```

**M3: Write a query to calculate the 30-day rolling average of daily sign-ups.**

```sql
SELECT
    signup_date,
    daily_signups,
    ROUND(AVG(daily_signups) OVER (
        ORDER BY signup_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ), 2) AS rolling_30d_avg
FROM daily_signups
ORDER BY signup_date;
```

**M4: Find employees who have never been assigned to a project, using three different methods.**

```sql
-- Method 1: LEFT JOIN / IS NULL
SELECT e.* FROM employees e
LEFT JOIN employee_projects ep ON e.employee_id = ep.employee_id
WHERE ep.employee_id IS NULL;

-- Method 2: NOT EXISTS
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM employee_projects ep WHERE ep.employee_id = e.employee_id
);

-- Method 3: NOT IN (safe only when subquery column is NOT NULL)
SELECT * FROM employees
WHERE employee_id NOT IN (SELECT DISTINCT employee_id FROM employee_projects);
```

**M5: Rank products by revenue within each category, and label the top 3 as "Top Performer."**

```sql
WITH ranked AS (
    SELECT
        product_id,
        product_name,
        category,
        total_revenue,
        RANK() OVER (PARTITION BY category ORDER BY total_revenue DESC) AS revenue_rank
    FROM product_revenue
)
SELECT
    product_id,
    product_name,
    category,
    total_revenue,
    revenue_rank,
    CASE WHEN revenue_rank <= 3 THEN 'Top Performer' ELSE 'Standard' END AS performance_label
FROM ranked
ORDER BY category, revenue_rank;
```

### Hard Level

**H1: Write a query to find users who logged in on at least 3 consecutive days.**

```sql
WITH daily_logins AS (
    SELECT DISTINCT user_id, login_date FROM user_logins
),
numbered AS (
    SELECT
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::INT * INTERVAL '1 day' AS grp
    FROM daily_logins
),
streaks AS (
    SELECT user_id, grp, COUNT(*) AS streak_length
    FROM numbered
    GROUP BY user_id, grp
)
SELECT DISTINCT user_id
FROM streaks
WHERE streak_length >= 3;
```

**H2: Write a query to find the median salary.**

```sql
-- PostgreSQL: built-in percentile function
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;

-- Without percentile_cont (any database):
WITH ordered AS (
    SELECT
        salary,
        ROW_NUMBER() OVER (ORDER BY salary) AS rn,
        COUNT(*) OVER ()                    AS total
    FROM employees
)
SELECT AVG(salary) AS median_salary
FROM ordered
WHERE rn IN (FLOOR((total + 1) / 2.0), CEIL((total + 1) / 2.0));
```

**H3: Find all manager-employee chains of depth 3 (employee -> manager -> manager's manager).**

```sql
WITH RECURSIVE org AS (
    SELECT employee_id, manager_id, first_name, 0 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.manager_id, e.first_name, org.depth + 1
    FROM employees e
    JOIN org ON e.manager_id = org.employee_id
    WHERE org.depth < 3
)
SELECT * FROM org WHERE depth = 3;
```

**H4: Find the longest streak of days any user visited the site consecutively.**

```sql
WITH daily AS (
    SELECT DISTINCT user_id, visit_date FROM site_visits
),
numbered AS (
    SELECT
        user_id,
        visit_date,
        visit_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY visit_date) || ' days')::INTERVAL AS grp
    FROM daily
),
streaks AS (
    SELECT user_id, grp, COUNT(*) AS streak_len
    FROM numbered
    GROUP BY user_id, grp
)
SELECT user_id, MAX(streak_len) AS longest_streak
FROM streaks
GROUP BY user_id
ORDER BY longest_streak DESC;
```

**H5: Write a query to detect and flag rows that were inserted more than once (exact duplicates across all non-key columns), keeping only the first occurrence.**

```sql
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY first_name, last_name, email, hire_date, salary
            ORDER BY employee_id
        ) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;

-- Or to delete duplicates (keep lowest employee_id):
DELETE FROM employees
WHERE employee_id IN (
    SELECT employee_id FROM (
        SELECT employee_id,
               ROW_NUMBER() OVER (
                   PARTITION BY first_name, last_name, email, hire_date, salary
                   ORDER BY employee_id
               ) AS rn
        FROM employees
    ) t WHERE rn > 1
);
```

**H6: For each month, find the product with the highest revenue. Show ties.**

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) AS sale_month,
        product_id,
        SUM(revenue) AS monthly_revenue
    FROM orders
    GROUP BY 1, 2
),
ranked AS (
    SELECT
        sale_month,
        product_id,
        monthly_revenue,
        RANK() OVER (PARTITION BY sale_month ORDER BY monthly_revenue DESC) AS rnk
    FROM monthly_revenue
)
SELECT sale_month, product_id, monthly_revenue
FROM ranked
WHERE rnk = 1
ORDER BY sale_month;
```

---

## 10. Behavioral and System Design SQL Questions

### System Design Questions

**Q: Design a schema for a URL shortener.**

```sql
CREATE TABLE short_urls (
    short_code   CHAR(8)      PRIMARY KEY,
    original_url TEXT         NOT NULL,
    created_by   INT          REFERENCES users(user_id),
    created_at   TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    expires_at   TIMESTAMP,
    is_active    BOOLEAN      DEFAULT TRUE
);

CREATE TABLE url_clicks (
    click_id    BIGSERIAL    PRIMARY KEY,
    short_code  CHAR(8)      REFERENCES short_urls(short_code),
    clicked_at  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    ip_address  INET,
    user_agent  TEXT,
    referrer    TEXT
);

CREATE INDEX idx_url_clicks_short_code ON url_clicks(short_code, clicked_at DESC);
```

**Q: Design a schema for a social media feed (users, posts, follows, likes).**

```sql
CREATE TABLE users (
    user_id    BIGSERIAL PRIMARY KEY,
    username   VARCHAR(50) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE follows (
    follower_id  BIGINT REFERENCES users(user_id),
    following_id BIGINT REFERENCES users(user_id),
    followed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    CHECK (follower_id != following_id)
);

CREATE TABLE posts (
    post_id    BIGSERIAL PRIMARY KEY,
    user_id    BIGINT REFERENCES users(user_id),
    content    TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE likes (
    user_id    BIGINT REFERENCES users(user_id),
    post_id    BIGINT REFERENCES posts(post_id),
    liked_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
);

-- Feed query: posts from users I follow, most recent first
SELECT p.post_id, p.user_id, p.content, p.created_at
FROM posts p
JOIN follows f ON f.following_id = p.user_id
WHERE f.follower_id = :current_user_id
ORDER BY p.created_at DESC
LIMIT 20;
```

**Q: How would you write a query for a leaderboard with millions of rows efficiently?**

```sql
-- Pre-aggregate scores in a materialized view or summary table
CREATE MATERIALIZED VIEW mv_leaderboard AS
SELECT
    user_id,
    SUM(score)  AS total_score,
    COUNT(*)    AS games_played,
    MAX(score)  AS best_score
FROM game_results
GROUP BY user_id;

CREATE UNIQUE INDEX idx_mv_leaderboard_user ON mv_leaderboard(user_id);
CREATE INDEX idx_mv_leaderboard_score ON mv_leaderboard(total_score DESC);

-- Paginated leaderboard using keyset pagination
SELECT user_id, total_score, RANK() OVER (ORDER BY total_score DESC) AS rank
FROM mv_leaderboard
ORDER BY total_score DESC
LIMIT 50;
```

---

## 11. Final Tricks and Reference Cheat Sheet

### Generate Series Without Recursive CTE (PostgreSQL)

```sql
-- Numbers
SELECT generate_series(1, 100) AS n;

-- Dates
SELECT generate_series('2024-01-01'::DATE, '2024-12-31'::DATE, '1 day'::INTERVAL) AS dt;
```

### Safe Division (Prevent Division by Zero)

```sql
SELECT numerator / NULLIF(denominator, 0) AS safe_division FROM calculations;
```

### String Aggregation

```sql
-- PostgreSQL
SELECT department_id, STRING_AGG(first_name, ', ' ORDER BY first_name) FROM employees GROUP BY department_id;

-- MySQL
SELECT department_id, GROUP_CONCAT(first_name ORDER BY first_name SEPARATOR ', ') FROM employees GROUP BY department_id;

-- SQL Server
SELECT department_id, STRING_AGG(first_name, ', ') WITHIN GROUP (ORDER BY first_name) FROM employees GROUP BY department_id;
```

### UPSERT (Insert or Update on Conflict)

```sql
-- PostgreSQL
INSERT INTO employees (employee_id, email, salary)
VALUES (101, 'new@example.com', 75000)
ON CONFLICT (employee_id)
DO UPDATE SET
    email  = EXCLUDED.email,
    salary = EXCLUDED.salary;

-- MySQL
INSERT INTO employees (employee_id, email, salary)
VALUES (101, 'new@example.com', 75000)
ON DUPLICATE KEY UPDATE
    email  = VALUES(email),
    salary = VALUES(salary);
```

### CASE Expression vs CASE Statement

A CASE expression returns a value and can appear anywhere a value is expected (SELECT, WHERE, ORDER BY, GROUP BY). A CASE statement (in procedural code like PL/pgSQL) controls program flow. In plain SQL, only CASE expressions exist.

```sql
-- CASE expression in ORDER BY: custom sort order
SELECT * FROM orders
ORDER BY
    CASE status
        WHEN 'Urgent'   THEN 1
        WHEN 'Normal'   THEN 2
        WHEN 'Low'      THEN 3
        ELSE 4
    END;
```

### Interview Performance Tips

Clarify the schema and expected output before writing anything. State your approach out loud before coding. Write readable SQL with consistent formatting. Consider edge cases: NULL values, empty tables, duplicate values, ties. Discuss trade-offs between approaches (readability, performance, portability). Know at least two ways to solve every problem.

### Complete SQL Logical Execution Order Reference

```
1.  FROM          -- Identify source tables
2.  ON            -- Join conditions
3.  JOIN          -- Combine tables
4.  WHERE         -- Filter rows (before aggregation, cannot use aliases or aggregates)
5.  GROUP BY      -- Group rows for aggregation
6.  HAVING        -- Filter groups (after aggregation, can use aggregate functions)
7.  SELECT        -- Compute output columns (aliases defined here)
8.  DISTINCT      -- Remove duplicate rows
9.  ORDER BY      -- Sort results (can use aliases defined in SELECT)
10. LIMIT/OFFSET  -- Restrict result count
```

### Common SQL Interview Topics Checklist

| Topic | Covered Day |
|---|---|
| Logical execution order | Day 1 |
| NULL behavior and traps | Day 1 |
| All JOIN types | Day 2 |
| LEFT JOIN filter trap | Day 2 |
| GROUP BY golden rule | Day 3 |
| HAVING vs WHERE | Day 3 |
| Correlated subqueries | Day 3 |
| CTEs and recursive CTEs | Day 3 |
| ROW_NUMBER vs RANK vs DENSE_RANK | Day 4 |
| LAG/LEAD for time-series analysis | Day 4 |
| Window frames | Day 4 |
| Top-N per group | Day 4 |
| Index internals and B-Tree | Day 5 |
| Composite index prefix rule | Day 5 |
| SARGable predicates | Day 5 |
| Execution plans | Day 5 |
| ACID properties | Day 6 |
| Isolation levels | Day 6 |
| Normalization 1NF-3NF | Day 6 |
| Views vs materialized views | Day 6 |
| Conditional aggregation / pivot | Day 7 |
| Gap and island problems | Day 7 |
| Deduplication with ROW_NUMBER | Day 7 |
| Median calculation | Day 7 |
| Schema design | Day 7 |

---

*Day 7 of 7 | End of the 7-Day SQL Mastery Program*

*Consistent practice, not passive reading, is what builds SQL mastery. Take every problem in this series and solve it from scratch without looking at the solution. Repeat until fluent.*
