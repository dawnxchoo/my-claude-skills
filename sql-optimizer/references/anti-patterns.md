# SQL Anti-Patterns & Fixes

Cross-dialect catalog of common SQL performance anti-patterns. Each entry includes the problematic pattern, why it's slow, and the fix.

---

## Joins

### Cartesian Join (Missing or Always-True ON Clause)

**Bad:**
```sql
SELECT a.*, b.*
FROM orders a, customers b
```

**Why it's slow:** Produces `rows_a × rows_b` result set. A 10K × 10K join produces 100M rows.

**Fix:**
```sql
SELECT a.*, b.*
FROM orders a
JOIN customers b ON a.customer_id = b.customer_id
```

---

### Joining on Functions/Expressions

**Bad:**
```sql
SELECT *
FROM orders o
JOIN customers c ON LOWER(o.email) = LOWER(c.email)
```

**Why it's slow:** Applying functions to join columns prevents index usage. The engine must evaluate the function for every row before comparing.

**Fix:** Normalize data at write time or create a computed/generated column with an index. If that's not possible, at minimum filter one side first to reduce the join size.

---

### Joining Large Tables Without Pre-Filtering

**Bad:**
```sql
SELECT o.order_id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2024-01-01'
  AND c.country = 'US'
```

**Why it's slow:** The optimizer may join all rows before filtering. While most modern optimizers push predicates down, complex queries or certain dialects may not.

**Fix:**
```sql
SELECT o.order_id, c.name
FROM (SELECT order_id, customer_id FROM orders WHERE order_date > '2024-01-01') o
JOIN (SELECT customer_id, name FROM customers WHERE country = 'US') c
  ON o.customer_id = c.customer_id
```

---

### Implicit Comma-Join Syntax

**Bad:**
```sql
SELECT *
FROM orders o, customers c, products p
WHERE o.customer_id = c.customer_id
  AND o.product_id = p.product_id
```

**Why it's slow:** Not always slower, but error-prone — missing one WHERE condition silently creates a Cartesian product. Explicit JOINs make intent clear and prevent accidental cross joins.

**Fix:**
```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN products p ON o.product_id = p.product_id
```

---

### Wrong Join Type Causing Row Explosion + DISTINCT Band-Aid

**Bad:**
```sql
SELECT DISTINCT o.order_id, o.total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
```

**Why it's slow:** The JOIN multiplies rows (one order → many items), then DISTINCT deduplicates. You're generating rows just to throw them away.

**Fix:** If you only need order-level data, don't join to the detail table. If you need an aggregate from the detail table:
```sql
SELECT o.order_id, o.total, oi.item_count
FROM orders o
JOIN (SELECT order_id, COUNT(*) AS item_count FROM order_items GROUP BY order_id) oi
  ON o.order_id = oi.order_id
```

---

## Subqueries

### Correlated Subquery That Should Be a JOIN or Window Function

**Bad:**
```sql
SELECT e.name, e.salary,
  (SELECT AVG(salary) FROM employees e2 WHERE e2.department_id = e.department_id) AS dept_avg
FROM employees e
```

**Why it's slow:** The subquery executes once per row in the outer query. For 100K employees, that's 100K separate AVG calculations.

**Fix (window function):**
```sql
SELECT e.name, e.salary,
  AVG(e.salary) OVER (PARTITION BY e.department_id) AS dept_avg
FROM employees e
```

**Fix (JOIN):**
```sql
SELECT e.name, e.salary, d.dept_avg
FROM employees e
JOIN (SELECT department_id, AVG(salary) AS dept_avg FROM employees GROUP BY department_id) d
  ON e.department_id = d.department_id
```

---

### Scalar Subquery in SELECT List

**Bad:**
```sql
SELECT
  o.order_id,
  (SELECT name FROM customers WHERE customer_id = o.customer_id) AS customer_name,
  (SELECT name FROM products WHERE product_id = o.product_id) AS product_name
FROM orders o
```

**Why it's slow:** Each scalar subquery executes per row. Two subqueries on 100K orders = 200K extra queries.

**Fix:**
```sql
SELECT o.order_id, c.name AS customer_name, p.name AS product_name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
LEFT JOIN products p ON o.product_id = p.product_id
```

---

### IN (SELECT ...) on Large Result Sets

**Bad:**
```sql
SELECT *
FROM orders
WHERE customer_id IN (SELECT customer_id FROM customers WHERE country = 'US')
```

**Why it's slow:** Some engines materialize the full subquery result into a list, then scan it for every row. With large subquery results, this is expensive.

**Fix:**
```sql
SELECT o.*
FROM orders o
WHERE EXISTS (SELECT 1 FROM customers c WHERE c.customer_id = o.customer_id AND c.country = 'US')
```

Or use a JOIN if you need columns from the inner table.

---

### Unnecessary Nesting Depth

**Bad:**
```sql
SELECT * FROM (
  SELECT * FROM (
    SELECT * FROM (
      SELECT id, amount FROM orders WHERE status = 'active'
    ) t1 WHERE amount > 100
  ) t2 WHERE id > 1000
) t3
```

**Why it's slow:** Deep nesting obscures intent and can confuse the optimizer. Each layer may prevent predicate pushdown.

**Fix:**
```sql
SELECT id, amount
FROM orders
WHERE status = 'active'
  AND amount > 100
  AND id > 1000
```

---

## Filtering

### Functions on Filtered Columns

**Bad:**
```sql
SELECT * FROM orders WHERE YEAR(created_at) = 2024
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15'
SELECT * FROM users WHERE UPPER(email) = 'USER@EXAMPLE.COM'
```

**Why it's slow:** Wrapping a column in a function prevents index or partition pruning. The engine must evaluate the function on every row.

**Fix:**
```sql
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
SELECT * FROM orders WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16'
-- For case-insensitive matching: use COLLATE, ILIKE, or a computed column
```

---

### LIKE with Leading Wildcard

**Bad:**
```sql
SELECT * FROM products WHERE name LIKE '%widget%'
```

**Why it's slow:** Leading wildcard prevents index usage — the engine must scan every row.

**Fix:** If possible, use full-text search (dialect-dependent). If not, consider whether the query can be restructured to use a prefix match (`LIKE 'widget%'`). If a leading wildcard is truly needed, accept the scan but limit it with additional filters.

---

### OR Conditions Preventing Index Usage

**Bad:**
```sql
SELECT * FROM orders WHERE customer_id = 123 OR product_id = 456
```

**Why it's slow:** Many engines can't use indexes on OR across different columns — falls back to a full scan.

**Fix:**
```sql
SELECT * FROM orders WHERE customer_id = 123
UNION ALL
SELECT * FROM orders WHERE product_id = 456 AND customer_id != 123
```

---

### NOT IN with NULLs

**Bad:**
```sql
SELECT * FROM orders WHERE customer_id NOT IN (SELECT customer_id FROM blacklist)
```

**Why it's slow/wrong:** If `blacklist.customer_id` contains any NULL, the entire NOT IN returns no rows (SQL three-valued logic). This is a correctness bug, not just a performance issue.

**Fix:**
```sql
SELECT o.*
FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.customer_id = o.customer_id)
```

---

### Filtering After Aggregation When It Could Be Done Before

**Bad:**
```sql
SELECT department_id, SUM(salary) AS total_salary
FROM employees
GROUP BY department_id
HAVING department_id IN (1, 2, 3)
```

**Why it's slow:** Aggregates all departments, then discards most. The filter doesn't depend on the aggregate.

**Fix:**
```sql
SELECT department_id, SUM(salary) AS total_salary
FROM employees
WHERE department_id IN (1, 2, 3)
GROUP BY department_id
```

Use HAVING only for conditions that reference aggregate values (e.g., `HAVING SUM(salary) > 100000`).

---

## Aggregations

### GROUP BY on Unnecessary Columns

**Bad:**
```sql
SELECT c.customer_id, c.name, c.email, c.phone, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.email, c.phone
```

**Why it's slow:** Grouping on many columns increases the work the engine does to hash/sort groups.

**Fix:** Group by the primary key only (if the dialect supports it — PostgreSQL and MySQL do):
```sql
SELECT c.customer_id, c.name, c.email, c.phone, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id  -- PK implies the other columns
```

Or aggregate first, then join:
```sql
SELECT c.customer_id, c.name, c.email, c.phone, oc.order_count
FROM customers c
JOIN (SELECT customer_id, COUNT(*) AS order_count FROM orders GROUP BY customer_id) oc
  ON c.customer_id = oc.customer_id
```

---

### HAVING for Conditions That Belong in WHERE

See [Filtering After Aggregation](#filtering-after-aggregation-when-it-could-be-done-before) above.

---

### COUNT(DISTINCT) on High-Cardinality Columns

**Bad:**
```sql
SELECT COUNT(DISTINCT user_id) FROM events  -- 1B rows, 50M distinct users
```

**Why it's slow:** Exact distinct counts require tracking every unique value — memory-intensive and slow at scale.

**Fix:** If an approximate count is acceptable (~2% error):
- BigQuery: `APPROX_COUNT_DISTINCT(user_id)`
- PostgreSQL: `HyperLogLog` extension
- Snowflake: `APPROX_COUNT_DISTINCT(user_id)`
- MySQL / SQL Server: No built-in approximate; consider application-level HyperLogLog or accept the cost

---

## Data Types

### Implicit Type Conversions in Joins or Filters

**Bad:**
```sql
-- user_id is INT in orders, VARCHAR in users
SELECT * FROM orders o JOIN users u ON o.user_id = u.user_id
-- status is INT, compared with string
SELECT * FROM orders WHERE status = '1'
```

**Why it's slow:** Implicit casts prevent index usage and can cause full table scans. In MySQL, this is especially insidious — no warning is raised.

**Fix:** Ensure matching types on both sides. Cast explicitly if needed, but prefer fixing the schema.

---

### Timestamp Comparisons Using String Casting

**Bad:**
```sql
SELECT * FROM events WHERE CAST(event_time AS VARCHAR) LIKE '2024-01%'
```

**Why it's slow:** Casts every row's timestamp to a string, then does a string comparison. Cannot use indexes or partition pruning.

**Fix:**
```sql
SELECT * FROM events WHERE event_time >= '2024-01-01' AND event_time < '2024-02-01'
```

---

## SELECT & Projection

### SELECT * in Production Queries

**Bad:**
```sql
SELECT * FROM orders WHERE order_date > '2024-01-01'
```

**Why it's slow:**
- Reads all columns — especially expensive in columnar stores (BigQuery, Snowflake) where cost = bytes scanned
- Prevents covering index usage in row-based engines (PostgreSQL, MySQL, SQL Server)
- Breaks downstream code when schema changes

**Fix:** Select only the columns you need:
```sql
SELECT order_id, customer_id, total FROM orders WHERE order_date > '2024-01-01'
```

---

### Unnecessary DISTINCT

**Bad:**
```sql
SELECT DISTINCT customer_id, order_date FROM orders
```

**Why it's slow:** DISTINCT forces a sort or hash to deduplicate. If the data is already unique (e.g., one row per order), DISTINCT is wasted work.

**Fix:** Determine why duplicates exist. If they shouldn't, fix the join or source. If DISTINCT is truly needed, keep it but be aware of the cost.

---

## Sorting & Ordering

### ORDER BY in Subqueries or CTEs

**Bad:**
```sql
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
  FROM employees
  ORDER BY name  -- this ORDER BY is pointless here
)
SELECT * FROM ranked WHERE rn = 1
```

**Why it's slow:** ORDER BY inside a CTE or subquery is generally ignored by the optimizer (Snowflake, BigQuery) or adds unnecessary sorting (PostgreSQL, MySQL). Only the outermost ORDER BY determines result order.

**Fix:** Remove ORDER BY from CTEs and subqueries. Add it only to the final SELECT.

---

### Sorting on Non-Indexed Columns (Large Result Sets)

**Bad:**
```sql
SELECT * FROM events ORDER BY event_data->>'category'  -- 100M rows, no index
```

**Why it's slow:** Requires a full sort of the entire result set in memory/disk.

**Fix:** Add an index on the sort column, filter to reduce rows before sorting, or paginate with `LIMIT`/`OFFSET` (or keyset pagination for better performance).

---

## Set Operations

### UNION Instead of UNION ALL

**Bad:**
```sql
SELECT customer_id FROM orders_2023
UNION
SELECT customer_id FROM orders_2024
```

**Why it's slow:** UNION deduplicates the combined result (sort + compare). If the sets are already disjoint or you don't care about duplicates, this is wasted work.

**Fix:**
```sql
SELECT customer_id FROM orders_2023
UNION ALL
SELECT customer_id FROM orders_2024
```

---

### Multiple Sequential UNIONs That Could Be a Single Query

**Bad:**
```sql
SELECT 'active' AS status, COUNT(*) FROM orders WHERE status = 'active'
UNION ALL
SELECT 'pending', COUNT(*) FROM orders WHERE status = 'pending'
UNION ALL
SELECT 'cancelled', COUNT(*) FROM orders WHERE status = 'cancelled'
```

**Why it's slow:** Scans the table three times.

**Fix:**
```sql
SELECT status, COUNT(*)
FROM orders
WHERE status IN ('active', 'pending', 'cancelled')
GROUP BY status
```
