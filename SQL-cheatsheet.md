# SQL Cheat Sheet

A comprehensive and concise reference guide covering essential SQL syntax, advanced features, and best practices—complete with examples for querying, joins, window functions, pagination, performance tuning, and security enhancements.

---

## Table of Contents

1. [Data Querying Basics](#data-querying-basics)
2. [Filtering & Grouping](#filtering--grouping)
3. [Joining Tables](#joining-tables)
4. [Subqueries & CTEs](#subqueries--ctes)
5. [Window Functions](#window-functions)
6. [Aggregate Functions](#aggregate-functions)
7. [String Functions](#string-functions)
8. [Date & Time Functions](#date--time-functions)
9. [Numeric Functions](#numeric-functions)
10. [Conversion Functions](#conversion-functions)
11. [Conditional Expressions](#conditional-expressions)
12. [Set Operations](#set-operations)
13. [Data Modification (DML)](#data-modification-dml)
14. [Schema Definition (DDL)](#schema-definition-ddl)
15. [Transactions & Control](#transactions--control)
16. [Indexing & Performance](#indexing--performance)
17. [Security & Permissions](#security--permissions)
18. [Gaps & Enhancement Suggestions](#gaps--enhancement-suggestions)

## Data Querying Basics

**SELECT**

Definition: Retrieves columns from one or more tables.

Use: The core of any query; pick which fields you want.

```sql
SELECT col1, col2 FROM table;
```

**FROM**

Definition: Specifies source table(s) for the query.

Use: Indicates where to pull data from.

**WHERE**

Definition: Filters rows based on boolean conditions.

Use: Return only rows meeting criteria.

```sql
SELECT *
  FROM users
 WHERE age >= 18;
```

**DISTINCT**

Definition: Eliminates duplicate rows in the result set.

Use: Find unique values.

```sql
SELECT DISTINCT country
  FROM customers;
```

**ORDER BY**

Definition: Sorts result rows by one or more columns.

Use: Order ascending (ASC) or descending (DESC).

```sql
SELECT *
  FROM sales
 ORDER BY sale_date DESC;
```

**LIMIT / FETCH / TOP**

Definition: Restricts number of rows returned.

Use: Paging or getting top N rows.

```sql
-- MySQL, PostgreSQL
SELECT * FROM orders LIMIT 10;

-- SQL Server
SELECT TOP 10 * FROM orders;
```

## Filtering & Grouping

**GROUP BY**

Definition: Aggregates rows sharing values in specified column(s).

Use: Compute sums, counts, etc., per group.

```sql
SELECT department, COUNT(*) AS num_emp
  FROM employees
 GROUP BY department;
```

**HAVING**

Definition: Filters groups (post-aggregation).

Use: E.g., departments with > 5 employees.

```sql
SELECT department, COUNT(*) AS num_emp
  FROM employees
 GROUP BY department
 HAVING COUNT(*) > 5;
```

## Joining Tables

**INNER JOIN**

Definition: Returns rows with matching keys in both tables.

Use: Combine related data where both sides exist.

```sql
SELECT *
  FROM orders o
  JOIN customers c ON o.cust_id = c.id;
```

**LEFT (OUTER) JOIN**

Definition: All rows from left table + matched from right, else NULL.

Use: Keep all left-table rows even if no match.

**RIGHT (OUTER) JOIN**

Definition: Mirror of LEFT JOIN for the right table.

Use: Less common; keep all right-table rows.

**FULL (OUTER) JOIN**

Definition: All rows from both tables, NULL when no match.

Use: Union of left and right with matching where possible.

**CROSS JOIN**

Definition: Cartesian product of two tables.

Use: Generate all combinations (use sparingly).

## Subqueries & CTEs

**Subquery (Nested Query)**

Definition: A SELECT inside another query.

Use: Filter or derive values without extra tables.

```sql
SELECT name
  FROM products
 WHERE price > (
   SELECT AVG(price) FROM products
 );
```

**Common Table Expression (CTE)**

Definition: Named temporary result set for use within a query.

Use: Break complex queries into readable parts.

```sql
WITH RecentSales AS (
  SELECT *
    FROM sales
   WHERE sale_date >= '2025-01-01'
)
SELECT region, SUM(amount)
  FROM RecentSales
 GROUP BY region;
```

## Window Functions

Definition: Perform calculations across a set of rows related to the current row.

Use: Running totals, ranking, moving averages.

| Function                                    | Definition & Use                          |
| ------------------------------------------- | ----------------------------------------- |
| `ROW_NUMBER()`                              | Assigns sequential integer to rows        |
| `RANK()`                                    | Like ROW\_NUMBER but ties share same rank |
| `DENSE_RANK()`                              | Like RANK but without gaps in ranking     |
| `LEAD(col, n)`                              | Access value from following row(s)        |
| `LAG(col, n)`                               | Access value from preceding row(s)        |
| `SUM(col) OVER (PARTITION BY … ORDER BY …)` | Running or partitioned sum                |

**Example:**

```sql
SELECT
  employee,
  salary,
  SUM(salary) OVER (
    ORDER BY hire_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_pay
FROM payroll;
```

## Aggregate Functions

| Function    | Definition                |
| ----------- | ------------------------- |
| `COUNT(*)`  | Number of rows            |
| `SUM(expr)` | Sum of numeric expression |
| `AVG(expr)` | Average of expression     |
| `MIN(expr)` | Minimum value             |
| `MAX(expr)` | Maximum value             |

## String Functions

| Function                   | Definition & Use               |
| -------------------------- | ------------------------------ |
| `CONCAT(str1, str2, …)`    | Join two or more strings       |
| `SUBSTRING(str, pos, len)` | Extract substring              |
| `LENGTH(str)`              | Number of characters           |
| `TRIM(str)`                | Remove leading/trailing spaces |
| `UPPER(str)`, `LOWER(str)` | Change case                    |
| `REPLACE(str, from, to)`   | Replace occurrences            |
| `POSITION(sub IN str)`     | Find substring position        |

## Date & Time Functions

| Function                                | Definition & Use                            |
| --------------------------------------- | ------------------------------------------- |
| `CURRENT_DATE`, `CURRENT_TIME`, `NOW()` | Get today’s date, current time or timestamp |
| `DATEADD(unit, n, date)` (T-SQL)        | Add interval to date                        |
| `DATEDIFF(unit, start, end)` (T-SQL)    | Difference between dates                    |
| `EXTRACT(field FROM date)`              | Pull part (year, month, day, hour…)         |
| `DATE_TRUNC(unit, timestamp)`           | Truncate to precision (e.g., month)         |
| `AGE(timestamp1, timestamp2)`           | Interval between timestamps                 |

## Numeric Functions

| Function                    | Definition & Use              |
| --------------------------- | ----------------------------- |
| `ROUND(expr, decimals)`     | Round to precision            |
| `FLOOR(expr)`, `CEIL(expr)` | Round down/up                 |
| `ABS(expr)`                 | Absolute value                |
| `POWER(x, y)`, `SQRT(x)`    | Exponentiation or square root |
| `MOD(x, y)`                 | Remainder of division         |

## Conversion Functions

**Syntax:** `CAST(expr AS type)` / `CONVERT(type, expr)`

Use: Change data type (e.g., string → date, int → varchar).

```sql
SELECT CAST(order_date AS DATE)
  FROM orders;
```

## Conditional Expressions

**`CASE WHEN … THEN … [ELSE …] END`**

Inline if-else logic.

```sql
SELECT
  amount,
  CASE
    WHEN amount < 100 THEN 'low'
    WHEN amount < 500 THEN 'medium'
    ELSE 'high'
  END AS category
FROM transactions;
```

**`COALESCE(val1, val2, …)`**

First non-NULL value.

**`NULLIF(expr1, expr2)`**

Returns NULL if `expr1 = expr2`, otherwise `expr1`.

## Set Operations

**`UNION [ALL]`**

Combine results of two queries (`ALL` preserves duplicates).

**`INTERSECT`**

Rows common to both queries.

**`EXCEPT` / `MINUS`**

Rows in first query not in second.

## Data Modification (DML)

**`INSERT INTO table (cols…) VALUES (…)`**

Add new row(s).

**`UPDATE table SET col = expr [, …] [WHERE …]`**

Change existing rows.

**`DELETE FROM table [WHERE …]`**

Remove rows.

## Schema Definition (DDL)

**`CREATE TABLE table (... columns…)`**

Define new table.

**`ALTER TABLE table ADD | DROP | MODIFY column …`**

Change schema.

**`DROP TABLE table`**

Remove table and data.

## Transactions & Control

**Commands:** `BEGIN` / `START TRANSACTION`, `COMMIT`, `ROLLBACK`

Group statements; ensure atomicity.

**Isolation Levels:** READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE

## Indexing & Performance

**`CREATE INDEX idx_name ON table(col[, …])`**

Speed lookups on columns.

**Maintenance:** `ANALYZE` / `VACUUM` (Postgres)

## Security & Permissions

**`GRANT privilege ON object TO role`**

Give `SELECT`, `INSERT`, `UPDATE`, `DELETE`, etc.

**`REVOKE privilege …`**

Remove access.

*Tip:* Different RDBMS (PostgreSQL, MySQL, SQL Server, Oracle) may have slight syntax/function variations—always check vendor docs.

## Gaps & Enhancement Suggestions

**Pagination**

* **OFFSET:** Commonly used in tandem with `LIMIT` (e.g., `LIMIT 10 OFFSET 20`).

* **ANSI FETCH:** Standard SQL’s `FETCH FIRST 10 ROWS ONLY` (optionally with `OFFSET`) for portability.

```sql
SELECT *
  FROM orders
 ORDER BY order_date
 OFFSET 20 ROWS FETCH FIRST 10 ROWS ONLY;
```

**Filtering Extensions**

* **IN / BETWEEN / LIKE:** Useful operators alongside `WHERE`:

```sql
SELECT *
  FROM products
 WHERE category IN ('A','B');

SELECT *
  FROM orders
 WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31';

SELECT *
  FROM users
 WHERE name LIKE 'J%';
```

* **Regular Expressions:** PostgreSQL’s `~` operator for pattern matching:

```sql
SELECT *
  FROM logs
 WHERE message ~ 'error [0-9]+';
```

**CTE Advanced Features**

* **Recursive CTEs:** Hierarchical queries, e.g., organizational charts:

```sql
WITH RECURSIVE Org AS (
  SELECT id, manager_id, name
    FROM employees
   WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.manager_id, e.name
    FROM employees e
    JOIN Org o ON e.manager_id = o.id
)
SELECT * FROM Org;
```

**Window Function Enhancements**

* **PARTITION BY** examples beyond running totals, like percentiles (`NTILE()`), moving averages:

```sql
SELECT region, sales,
  NTILE(4) OVER (PARTITION BY region ORDER BY sales) AS sales_quartile
FROM sales_data;
```

* **Frame clauses:** Precise windows, e.g., `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`.

**Advanced Aggregations**

* **GROUPING SETS / ROLLUP / CUBE:** Multidimensional summaries:

```sql
SELECT region, product,
  SUM(amount)
  FROM sales
 GROUP BY ROLLUP(region, product);
```

**Schema Constraints**

* **PRIMARY KEY** / **FOREIGN KEY** / **UNIQUE** / **NOT NULL** / **CHECK:** Fundamental table-level constraints.

**Data Modeling & Integrity**

* **Views:** Virtual tables via `CREATE VIEW`.

* **Triggers & Stored Procedures:** Automate and encapsulate logic at the database level.

```sql
-- Example Trigger
CREATE TRIGGER update_modified
  BEFORE UPDATE ON employees
  FOR EACH ROW
  EXECUTE PROCEDURE update_modified_column();

-- Example Stored Procedure
CREATE PROCEDURE add_employee(
  IN p_name TEXT,
  IN p_dept TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
  INSERT INTO employees (name, department)
  VALUES (p_name, p_dept);
END;
$$;
```

**Performance Tuning**

* **EXPLAIN / EXPLAIN ANALYZE:** Inspect query plans and costs.

```sql
EXPLAIN ANALYZE
SELECT *
  FROM employees
 WHERE salary > 50000;
```

Sample output:

```text
                                       QUERY PLAN
------------------------------------------------------------------------------------------------
Seq Scan on employees  (cost=0.00..12.34 rows=56 width=100)
  Filter: (salary > 50000)
Planning Time: 0.123 ms
Execution Time: 0.456 ms
```

* **Partitioning:** Large table management (range, list, hash partitioning).

**Security Advanced**

* **Roles / Row-Level Security:** Fine-grained access control.

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Allow users to see only their own orders
CREATE POLICY user_orders_policy
  ON orders
  FOR SELECT
  USING (user_id = current_user);
```

* **Encryption-at-Rest & In-Transit:** Best practices for data protection.

**Vendor-Specific Highlights**

* Syntactic differences, e.g., Oracle’s `NVL` vs. `COALESCE`, Oracle’s `ROWNUM`, SQL Server’s `TOP`, etc.
