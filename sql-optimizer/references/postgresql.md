# PostgreSQL-Specific Optimizations

## Architecture Notes

- **Row-based storage** — entire rows are stored together. Good for OLTP, but wide-table scans read unused columns.
- **B-tree indexes by default** — also supports GIN, GiST, BRIN, and hash indexes.
- **Statistics-driven planner** — `ANALYZE` keeps table statistics fresh. Stale stats lead to bad plans.
- **MVCC** — multi-version concurrency control means dead tuples accumulate. `VACUUM` is essential.

---

## Index Types & When to Suggest Them

| Index Type | Best For |
|---|---|
| **B-tree** | Equality and range queries (`=`, `<`, `>`, `BETWEEN`). Default choice. |
| **GIN** | Arrays, JSONB, full-text search. Use when querying `@>`, `?`, `@@` operators. |
| **GiST** | Geometric data, range types, full-text search. Use for spatial queries or range overlaps. |
| **BRIN** | Very large tables where data is physically sorted (e.g., append-only time-series). Tiny index, big savings. |

When a query filters or joins on a column without an index, suggest creating one — but note this is a schema change, not a query rewrite.

### Partial Indexes

For queries that always include a filter, a partial index avoids indexing irrelevant rows:
```sql
CREATE INDEX idx_active_orders ON orders (customer_id) WHERE status = 'active';
```

### Covering Indexes (INCLUDE)

PostgreSQL 11+ supports covering indexes to enable index-only scans:
```sql
CREATE INDEX idx_orders_covering ON orders (customer_id) INCLUDE (order_date, total);
```

---

## CTE Materialization

### Pre-v12: CTEs Are Optimization Fences

Before PostgreSQL 12, CTEs were always materialized — the optimizer could not push predicates into them.

**Problem in pre-v12:**
```sql
WITH big_cte AS (
  SELECT * FROM large_table  -- materialized: scans entire table
)
SELECT * FROM big_cte WHERE id = 42  -- filter applied AFTER materialization
```

### v12+: CTEs Are Inlined by Default

PostgreSQL 12+ inlines CTEs unless they're referenced multiple times or you explicitly materialize them.

**Control materialization explicitly:**
```sql
-- Force materialization (useful when CTE is referenced multiple times)
WITH expensive_calc AS MATERIALIZED (
  SELECT department_id, AVG(salary) AS avg_salary FROM employees GROUP BY department_id
)
SELECT * FROM expensive_calc ...

-- Force inlining (useful when you know the outer filter is selective)
WITH big_cte AS NOT MATERIALIZED (
  SELECT * FROM large_table
)
SELECT * FROM big_cte WHERE id = 42
```

---

## PostgreSQL-Specific Patterns

### EXISTS Over IN for Subqueries

PostgreSQL's planner handles EXISTS more efficiently than IN for large subquery results:
```sql
-- Prefer
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM customers c WHERE c.id = o.customer_id AND c.country = 'US')

-- Over
SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customers WHERE country = 'US')
```

### LATERAL Joins for Dependent Subqueries

Replace correlated subqueries with LATERAL for better optimizer visibility:
```sql
-- Bad: correlated subquery
SELECT c.name,
  (SELECT MAX(o.total) FROM orders o WHERE o.customer_id = c.id) AS max_order
FROM customers c

-- Good: LATERAL join
SELECT c.name, o.max_order
FROM customers c
LEFT JOIN LATERAL (
  SELECT MAX(total) AS max_order FROM orders WHERE customer_id = c.id
) o ON true
```

### FILTER Clause Over CASE in Aggregates

```sql
-- Verbose
SELECT
  COUNT(CASE WHEN status = 'active' THEN 1 END) AS active_count,
  COUNT(CASE WHEN status = 'pending' THEN 1 END) AS pending_count
FROM orders

-- Clean (PostgreSQL-specific)
SELECT
  COUNT(*) FILTER (WHERE status = 'active') AS active_count,
  COUNT(*) FILTER (WHERE status = 'pending') AS pending_count
FROM orders
```

### Use generate_series for Gap-Filling

Instead of self-joins or application logic to fill date gaps:
```sql
SELECT d.date, COALESCE(o.order_count, 0) AS order_count
FROM generate_series('2024-01-01'::date, '2024-12-31'::date, '1 day') AS d(date)
LEFT JOIN (
  SELECT order_date, COUNT(*) AS order_count FROM orders GROUP BY order_date
) o ON o.order_date = d.date
```

---

## Things to Avoid

- **Scalar UDFs (PL/pgSQL) in WHERE clauses** — prevents index usage and blocks parallelism.
- **Implicit type casts** — `WHERE id = '123'` when `id` is integer forces a cast on every row. Use explicit `::` casting.
- **SELECT DISTINCT on unindexed columns** — forces a full sort. Suggest adding an index or restructuring.
- **Large OFFSET values for pagination** — `OFFSET 100000` still scans and discards 100K rows. Suggest keyset pagination instead.

---

## Diagnostic Suggestions

When optimization depends on data distribution, suggest:
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) <query>;
```

This shows actual execution times, row estimates vs actuals, and buffer usage — essential for identifying where the planner's estimates diverge from reality.
