# MySQL-Specific Optimizations

## Architecture Notes

- **InnoDB (default engine)** — row-based, B-tree indexes, MVCC, supports transactions.
- **Clustered index** — InnoDB stores rows ordered by the primary key. The PK *is* the table structure.
- **Query cache** — deprecated in MySQL 8.0+. Do not rely on it.
- **Optimizer** — cost-based, but historically less sophisticated than PostgreSQL's. Optimizer hints (8.0+) give manual control.

---

## Index Strategy

### Covering Indexes

A covering index includes all columns needed by the query, allowing an **index-only scan** (no row lookup).

**Bad:**
```sql
-- Index on (customer_id), but query also selects order_date and total
SELECT customer_id, order_date, total FROM orders WHERE customer_id = 123
-- MySQL reads the index to find rows, then fetches each row for the other columns
```

**Good:**
```sql
-- Composite index: (customer_id, order_date, total)
-- Now the query is fully covered by the index — no table lookup needed
```

This is why `SELECT *` is especially harmful in MySQL — it almost always prevents covering index usage.

### Leftmost Prefix Rule

Composite indexes in MySQL only work when queries filter on a **leftmost prefix** of the index columns.

Index on `(a, b, c)`:
- `WHERE a = 1` — uses index
- `WHERE a = 1 AND b = 2` — uses index
- `WHERE a = 1 AND b = 2 AND c = 3` — uses index
- `WHERE b = 2` — does NOT use index
- `WHERE a = 1 AND c = 3` — uses index for `a` only, skips to `c`

### Index Hints (Use Sparingly)

When the optimizer chooses a bad plan:
```sql
SELECT * FROM orders USE INDEX (idx_customer_date) WHERE customer_id = 123 AND order_date > '2024-01-01'
SELECT * FROM orders FORCE INDEX (idx_customer_date) WHERE ...  -- stronger hint
```

Use only after verifying with EXPLAIN that the optimizer's choice is suboptimal.

---

## MySQL-Specific Gotchas

### Implicit Type Conversions Silently Kill Index Usage

This is MySQL's most insidious performance trap — no warning is raised.

**Bad:**
```sql
-- user_id is VARCHAR in the table
SELECT * FROM users WHERE user_id = 12345  -- numeric literal, VARCHAR column
-- MySQL silently casts EVERY row's user_id to a number. Full table scan.
```

**Fix:**
```sql
SELECT * FROM users WHERE user_id = '12345'  -- match the column type
```

### UTF8 Collation Mismatches in Joins

If two tables use different collations (e.g., `utf8_general_ci` vs `utf8mb4_unicode_ci`), joins between their string columns cannot use indexes.

**Fix:** Ensure matching collations on joined columns, or use CONVERT/COLLATE explicitly (at a performance cost).

### GROUP BY Implicit Sorting

Before MySQL 8.0, `GROUP BY` implicitly added `ORDER BY`. In 8.0+, this was removed. If you need ordered results, always add an explicit `ORDER BY`.

---

## MySQL-Specific Patterns

### Optimizer Hints (MySQL 8.0+)

```sql
-- Force a specific join order
SELECT /*+ JOIN_ORDER(c, o) */ c.name, o.total
FROM customers c JOIN orders o ON c.id = o.customer_id

-- Force or prevent index usage
SELECT /*+ INDEX(orders idx_customer) */ * FROM orders WHERE customer_id = 123
SELECT /*+ NO_INDEX(orders idx_status) */ * FROM orders WHERE status = 'active'

-- Limit memory for a specific operation
SELECT /*+ SET_VAR(sort_buffer_size = 16M) */ * FROM big_table ORDER BY col
```

### INSERT ... ON DUPLICATE KEY UPDATE for Upserts

```sql
INSERT INTO daily_stats (date, page, views)
VALUES ('2024-06-15', '/home', 100)
ON DUPLICATE KEY UPDATE views = views + VALUES(views)
```

### EXPLAIN FORMAT=TREE (MySQL 8.0+)

The tree format is much more readable than the traditional tabular output:
```sql
EXPLAIN FORMAT=TREE SELECT * FROM orders WHERE customer_id = 123;
-- or for actual execution stats:
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;
```

---

## Things to Avoid

- **SELECT * in any query touching indexed tables** — almost always prevents covering index usage.
- **HAVING for non-aggregate filters** — use WHERE instead (see anti-patterns).
- **Large IN lists** — MySQL handles them reasonably, but beyond ~1000 items, consider a temp table join instead.
- **Subqueries in FROM clause (derived tables) without LIMIT** — older MySQL versions materialize these fully before applying outer filters.
- **ORDER BY RAND()** — scans and generates a random number for every row. For random sampling, use `WHERE id >= (SELECT FLOOR(RAND() * MAX(id)) FROM table) LIMIT n` or application-level sampling.
- **COUNT(*) on InnoDB tables without WHERE** — InnoDB doesn't store exact row counts (unlike MyISAM). A full table scan is required.

---

## Diagnostic Suggestions

```sql
-- Basic query plan
EXPLAIN SELECT ...;

-- Tree format (8.0+, much more readable)
EXPLAIN FORMAT=TREE SELECT ...;

-- Actual execution with timing (8.0.18+)
EXPLAIN ANALYZE SELECT ...;

-- Check index usage on a table
SHOW INDEX FROM table_name;
```
