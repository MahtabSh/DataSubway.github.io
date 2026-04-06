---
title: "SQL Window Functions: A Practical Guide"
date: 2026-04-06
categories: ["SQL"]
tags: ["SQL", "Window Functions", "Analytics", "Query Optimization"]
series: ["Engineer's Notebook"]
draft: false
---

Window functions are one of the most powerful and underused features in SQL. Unlike `GROUP BY`, they let you perform calculations **across a set of rows related to the current row — without collapsing the result set**.

---

## The Core Syntax

```sql
function_name() OVER (
    PARTITION BY column1
    ORDER BY column2
    ROWS/RANGE BETWEEN ... AND ...
)
```

| Clause | Purpose | Required? |
| :--- | :--- | :--- |
| `PARTITION BY` | Divides rows into groups (like GROUP BY, but rows are kept) | No |
| `ORDER BY` | Defines the order within each partition | Depends on function |
| Frame (`ROWS BETWEEN`) | Defines which rows to include relative to the current row | No |

---

## 1. Ranking Functions

### `ROW_NUMBER()`
Assigns a unique sequential integer to each row within a partition. No ties.

```sql
SELECT
    employee_id,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_in_dept
FROM employees;
```

**Use when:** You need exactly one row per group (e.g., the top earner per department).

---

### `RANK()` and `DENSE_RANK()`
Both handle ties, but differently:
- `RANK()` — gaps after ties (1, 2, 2, **4**)
- `DENSE_RANK()` — no gaps (1, 2, 2, **3**)

```sql
SELECT
    employee_id,
    salary,
    RANK()       OVER (ORDER BY salary DESC) AS rank_with_gap,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS rank_no_gap
FROM employees;
```

**Use when:** You want to find "top N" records and need to handle ties consistently.

---

## 2. Aggregate Window Functions

Any standard aggregate (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) can be used as a window function by adding `OVER()`.

### Running Total

```sql
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

### Percentage of Total

```sql
SELECT
    department,
    salary,
    ROUND(salary / SUM(salary) OVER () * 100, 2) AS pct_of_total_payroll
FROM employees;
```

### Running Average per Group

```sql
SELECT
    department,
    order_date,
    revenue,
    AVG(revenue) OVER (
        PARTITION BY department
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM sales;
```

---

## 3. Offset Functions

### `LAG()` and `LEAD()`
Access a value from a **previous** or **next** row without a self-join.

```sql
SELECT
    order_date,
    revenue,
    LAG(revenue, 1)  OVER (ORDER BY order_date) AS prev_day_revenue,
    LEAD(revenue, 1) OVER (ORDER BY order_date) AS next_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_sales;
```

**Use when:** Calculating period-over-period changes (MoM, YoY, day-over-day).

---

### `FIRST_VALUE()` and `LAST_VALUE()`
Return the first or last value in the window frame.

```sql
SELECT
    employee_id,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS highest_salary_in_dept
FROM employees;
```

> **Watch out:** `LAST_VALUE()` requires an explicit frame (`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) because the default frame stops at the current row.

---

## 4. Frame Clauses

The frame defines which rows are included in the window relative to the current row.

| Frame | Meaning |
| :--- | :--- |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | All rows from the start of the partition to now (running total) |
| `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` | Current row + 2 rows before (3-row rolling window) |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | All rows in the partition |
| `RANGE BETWEEN INTERVAL '7' DAY PRECEDING AND CURRENT ROW` | Rows within the last 7 days by date value |

---

## 5. Common Patterns

### Deduplicate: Keep the latest record per ID

```sql
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY updated_at DESC) AS rn
    FROM users
)
SELECT * FROM ranked WHERE rn = 1;
```

### Find records where a value changed

```sql
WITH changes AS (
    SELECT *,
        LAG(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS prev_status
    FROM order_history
)
SELECT * FROM changes
WHERE status <> prev_status OR prev_status IS NULL;
```

### Top N per group

```sql
WITH ranked AS (
    SELECT *,
        DENSE_RANK() OVER (PARTITION BY category ORDER BY sales DESC) AS dr
    FROM products
)
SELECT * FROM ranked WHERE dr <= 3;
```

---

## When to Use Window Functions vs. GROUP BY

| Situation | Use |
| :--- | :--- |
| You need aggregated values **and** the original rows | Window function |
| You only need one row per group | `GROUP BY` |
| You need to compare a row to its neighbors | Window function (`LAG`/`LEAD`) |
| You need a running total or rolling average | Window function |
| You need a simple count or sum per group | `GROUP BY` (simpler, often faster) |

---

## Performance Notes

- Window functions run **after** `WHERE` and `GROUP BY`, but **before** `ORDER BY` and `LIMIT`.
- Avoid using window functions in `WHERE` clauses — wrap in a CTE or subquery first.
- `PARTITION BY` on an indexed column significantly improves performance.
- In columnar warehouses (BigQuery, Snowflake, Redshift), window functions are well-optimized — don't be afraid to use them.
