# Snowflake-Specific Optimizations

## Architecture Notes

- **Columnar micro-partition storage** — data is organized into micro-partitions (50-500 MB compressed). Each partition stores column min/max metadata for pruning.
- **No traditional indexes** — performance depends entirely on partition pruning and clustering.
- **Automatic clustering** — Snowflake automatically maintains clustering over time for active tables, but manual clustering keys help for large tables with specific access patterns.
- **Aggressive caching** — result cache (24hr), local disk cache (warehouse), and metadata cache. Identical queries return instantly from cache.
- **Warehouse sizing** — compute is separate from storage. Larger warehouse = more parallel execution = faster queries (but more credits).

---

## Clustering Keys

### When to Use Clustering Keys

Clustering keys are Snowflake's primary performance lever for large tables (typically > 1 TB or > 1B rows).

**Good clustering key candidates:**
- Columns frequently used in WHERE clauses
- Columns used in JOIN conditions
- Date/time columns for time-series data
- Low-to-medium cardinality columns (e.g., region, status)

```sql
ALTER TABLE events CLUSTER BY (event_date, event_type);
```

### Filter on Clustering Columns

**Bad:**
```sql
SELECT * FROM events WHERE user_id = 12345
-- If clustered by (event_date, event_type), this scans most partitions
```

**Good:**
```sql
SELECT * FROM events
WHERE event_date = '2024-06-15' AND event_type = 'purchase' AND user_id = 12345
-- Prunes to relevant partitions, then filters within them
```

---

## Snowflake-Specific Functions & Patterns

### QUALIFY for Window Function Filtering

**Bad:**
```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_time DESC) AS rn
  FROM events
) WHERE rn = 1
```

**Good:**
```sql
SELECT *
FROM events
QUALIFY ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_time DESC) = 1
```

### FLATTEN for Semi-Structured Data

When working with VARIANT/OBJECT/ARRAY columns:

**Bad:**
```sql
-- Extracting array elements with a lateral join pattern
SELECT e.event_id, t.value::STRING AS tag
FROM events e,
LATERAL FLATTEN(input => e.tags) t
```

This is actually the correct pattern — but be aware that FLATTEN on large arrays produces many rows. Pre-filter before flattening:

```sql
SELECT e.event_id, t.value::STRING AS tag
FROM events e,
LATERAL FLATTEN(input => e.tags) t
WHERE e.event_date = '2024-06-15'  -- filter first, flatten the smaller set
```

### TRY_CAST and TRY_TO_* for Safe Conversions

```sql
-- Fails on bad data
SELECT CAST(raw_value AS INTEGER) FROM raw_events

-- Returns NULL on bad data
SELECT TRY_CAST(raw_value AS INTEGER) FROM raw_events
-- or
SELECT TRY_TO_NUMBER(raw_value) FROM raw_events
```

### APPROX_COUNT_DISTINCT

```sql
-- Exact: expensive at scale
SELECT COUNT(DISTINCT user_id) FROM events

-- Approximate (~2% error): much faster
SELECT APPROX_COUNT_DISTINCT(user_id) FROM events
```

### TABLESAMPLE for Development and Testing

Avoid scanning full tables during development:
```sql
-- Sample ~10% of rows (block-level sampling)
SELECT * FROM large_table TABLESAMPLE (10);

-- Or fixed number of rows
SELECT * FROM large_table SAMPLE (1000 ROWS);
```

---

## Query Structure Optimizations

### Column Pruning

Like BigQuery, Snowflake is columnar — fewer columns = less I/O:
```sql
-- Bad: reads all columns from all micro-partitions
SELECT * FROM wide_table

-- Good: reads only needed columns
SELECT user_id, event_type, event_time FROM wide_table
```

### Avoid ORDER BY in CTEs and Views

Snowflake ignores ORDER BY in CTEs and views. Including it misleads readers and wastes parse time:
```sql
-- Bad: ORDER BY here does nothing
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (ORDER BY created_at) AS rn
  FROM events
  ORDER BY created_at  -- ignored by Snowflake
)
SELECT * FROM ranked

-- Good: ORDER BY only at the outermost query
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (ORDER BY created_at) AS rn
  FROM events
)
SELECT * FROM ranked ORDER BY created_at
```

### CTEs Are Inlined (Not Materialized)

Snowflake inlines CTEs — no optimization fence. However, if a CTE is referenced multiple times, it's re-evaluated each time. For expensive computations referenced multiple times, consider a temp table:

```sql
-- If expensive_calc is referenced 3+ times, consider:
CREATE TEMPORARY TABLE temp_calc AS
SELECT department_id, AVG(salary) AS avg_salary FROM employees GROUP BY department_id;
```

---

## Things to Avoid

- **ORDER BY in views or CTEs** — silently ignored (see above).
- **Very wide SELECT on VARIANT columns** — `SELECT raw_data:*` or extracting dozens of nested paths. Extract only what's needed.
- **Small warehouse for large scan queries** — if a query scans TBs of data, a larger warehouse (more nodes) can be significantly faster. This is a compute sizing decision, not a SQL change.
- **UNION instead of UNION ALL** — forces dedup across the full result.

---

## Warehouse Sizing Notes

When a query involves large scans or complex joins:
- Suggest the user consider warehouse sizing as a tuning lever
- XS warehouses are fine for simple queries on small data
- For analytical queries scanning > 100 GB, a Medium or Large warehouse may complete in seconds vs minutes
- Auto-suspend and auto-resume minimize cost — larger warehouses aren't always more expensive overall

---

## Diagnostic Suggestions

```sql
-- Query profile (in Snowflake UI: History → Query Profile)
-- Shows operator tree, data scanned, partition pruning stats

-- Check clustering quality
SELECT SYSTEM$CLUSTERING_INFORMATION('events', '(event_date, event_type)');

-- Check bytes scanned (after running a query)
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_ID = LAST_QUERY_ID()
```
