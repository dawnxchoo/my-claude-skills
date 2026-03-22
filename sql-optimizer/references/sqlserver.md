# SQL Server-Specific Optimizations

## Architecture Notes

- **Row-based + columnstore** — traditional row store (B-tree) plus columnstore indexes for analytics workloads.
- **Cost-based optimizer** — sophisticated planner with plan caching. Cached plans can be a double-edged sword (parameter sniffing).
- **Clustered index** — like MySQL/InnoDB, the clustered index defines physical row order. Tables without one are heaps (generally avoid).
- **T-SQL** — SQL Server's dialect. Square bracket identifiers `[table]`, `TOP` instead of `LIMIT`, `CROSS APPLY` for lateral joins.

---

## Index Types

| Index Type | Best For |
|---|---|
| **Clustered** | One per table. Defines physical order. Usually the PK. Choose a narrow, ever-increasing column (e.g., INT IDENTITY). |
| **Non-clustered** | Secondary lookups. Multiple per table. Include additional columns with INCLUDE to enable covering. |
| **Columnstore** | Analytics/reporting on large tables. Massive compression and batch-mode execution. |
| **Filtered** | Indexes on a subset of rows. Like PostgreSQL's partial indexes. |

### Included Columns (Covering Indexes)

```sql
-- Instead of indexing all queried columns (wide index, expensive to maintain)
CREATE NONCLUSTERED INDEX idx_orders_customer
ON orders (customer_id)
INCLUDE (order_date, total);
-- Covers: SELECT order_date, total FROM orders WHERE customer_id = 123
```

### Filtered Indexes

```sql
CREATE NONCLUSTERED INDEX idx_active_orders
ON orders (customer_id, order_date)
WHERE status = 'active';
-- Only indexes active orders — smaller, faster, and cheaper to maintain
```

### Columnstore Indexes for Analytics

For large tables queried primarily with aggregations and scans:
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX idx_orders_analytics
ON orders (order_date, customer_id, product_id, total);
```

Columnstore provides 10-100x compression and enables batch-mode execution for aggregation queries.

---

## Parameter Sniffing

SQL Server caches query plans based on the first execution's parameter values. If data distribution is skewed, the cached plan may be terrible for subsequent calls.

**Symptoms:** A query is fast with one parameter value, slow with another.

**Fixes:**
```sql
-- Option 1: Force recompile for specific queries
SELECT * FROM orders WHERE customer_id = @id OPTION (RECOMPILE);

-- Option 2: Optimize for unknown (average plan)
SELECT * FROM orders WHERE customer_id = @id OPTION (OPTIMIZE FOR UNKNOWN);

-- Option 3: Optimize for a specific typical value
SELECT * FROM orders WHERE customer_id = @id OPTION (OPTIMIZE FOR (@id = 12345));
```

---

## SQL Server-Specific Patterns

### CROSS APPLY Instead of Correlated Subqueries

`CROSS APPLY` is SQL Server's equivalent of PostgreSQL's `LATERAL JOIN`:

**Bad:**
```sql
SELECT c.name,
  (SELECT TOP 1 o.total FROM orders o WHERE o.customer_id = c.id ORDER BY o.order_date DESC) AS latest_total
FROM customers c
```

**Good:**
```sql
SELECT c.name, o.latest_total
FROM customers c
CROSS APPLY (
  SELECT TOP 1 total AS latest_total
  FROM orders
  WHERE customer_id = c.id
  ORDER BY order_date DESC
) o
```

Use `OUTER APPLY` (equivalent of `LEFT JOIN LATERAL`) when the subquery may return no rows.

### Temp Tables vs Table Variables

**Temp tables (`#temp`):**
- Have statistics — optimizer makes better choices
- Can be indexed
- Use for larger result sets (thousands+ rows)

**Table variables (`@table`):**
- No statistics — optimizer assumes 1 row
- Cannot be indexed (before SQL Server 2019)
- Use only for small result sets (dozens of rows)

**Rule of thumb:** Default to temp tables. Table variables are only appropriate for very small, simple intermediary results.

### SET NOCOUNT ON

In stored procedures and scripts, always include:
```sql
SET NOCOUNT ON;
```
Suppresses "N rows affected" messages, reducing network overhead.

### TOP with ORDER BY

SQL Server uses `TOP` instead of `LIMIT`:
```sql
-- Get top 10 customers by order total
SELECT TOP 10 customer_id, SUM(total) AS total_spent
FROM orders
GROUP BY customer_id
ORDER BY total_spent DESC
```

For pagination, use `OFFSET...FETCH` (SQL Server 2012+):
```sql
SELECT order_id, total
FROM orders
ORDER BY order_date DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY
```

---

## Things to Avoid

- **Scalar UDFs in WHERE clauses** — SQL Server executes scalar UDFs row-by-row, preventing parallelism. Rewrite as inline table-valued functions or native SQL.
- **Cursors** — row-by-row processing. Almost always replaceable with set-based operations.
- **NOLOCK hints everywhere** — `WITH (NOLOCK)` reads uncommitted data (dirty reads). Use only when you genuinely accept stale/phantom data.
- **SELECT * with columnstore indexes** — columnstore's benefit comes from column pruning. `SELECT *` forces reading all columns.
- **Functions on indexed columns in WHERE** — same as all dialects, but especially impactful here because SQL Server's plan cache amplifies the effect.
- **Large transaction batches without batching** — wrap large UPDATE/DELETE operations in smaller batches to avoid lock escalation and log bloat.

---

## Diagnostic Suggestions

```sql
-- Actual execution plan with runtime stats
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT ...;

-- Graphical execution plan (in SSMS, Ctrl+M before running)

-- Estimated plan without executing
SET SHOWPLAN_XML ON;
GO
SELECT ...;
GO
SET SHOWPLAN_XML OFF;

-- Find missing indexes (SQL Server suggests them in execution plans)
SELECT * FROM sys.dm_db_missing_index_details;
```
