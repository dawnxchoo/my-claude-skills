# BigQuery-Specific Optimizations

## Architecture Notes

- **Columnar storage** — data is stored by column, not by row. Selecting fewer columns = less data scanned = lower cost.
- **Slot-based execution** — queries are distributed across compute slots. No traditional indexes.
- **Cost model** — charged by bytes scanned (on-demand) or slot-time (flat-rate). Column pruning directly reduces cost.
- **Performance levers** — partitioning and clustering are the primary tools. There are no indexes to create.

---

## Partitioning

### Always Filter on the Partition Column

BigQuery tables are commonly partitioned by a date/timestamp column. Queries without a partition filter scan the entire table.

**Bad:**
```sql
SELECT * FROM `project.dataset.events`
WHERE event_name = 'purchase'
```

**Good:**
```sql
SELECT event_name, user_id, event_timestamp
FROM `project.dataset.events`
WHERE DATE(event_timestamp) = '2024-06-15'
  AND event_name = 'purchase'
```

For ingestion-time partitioned tables, use the pseudo-columns `_PARTITIONTIME` or `_PARTITIONDATE`:
```sql
WHERE _PARTITIONDATE = '2024-06-15'
```

### Avoid Functions That Break Partition Pruning

**Bad:**
```sql
WHERE EXTRACT(YEAR FROM event_date) = 2024  -- scans all partitions
```

**Good:**
```sql
WHERE event_date BETWEEN '2024-01-01' AND '2024-12-31'  -- prunes to 2024 partitions only
```

---

## Clustering

- Clustering sorts data within partitions by up to 4 columns.
- Filters and aggregations on clustering columns are faster because BigQuery can skip irrelevant blocks.
- Put clustering columns in WHERE clauses and GROUP BY when possible.
- Order of clustering columns matters — filter on the first clustering column for best pruning.

---

## Column Pruning

Never use `SELECT *` in BigQuery — you pay for every byte scanned.

**Bad:**
```sql
SELECT * FROM `project.dataset.wide_table`  -- 200 columns, you need 3
```

**Good:**
```sql
SELECT user_id, event_name, event_timestamp FROM `project.dataset.wide_table`
```

---

## BigQuery-Specific Functions & Patterns

### QUALIFY Instead of Subquery Wrapping

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

### APPROX_COUNT_DISTINCT for Large-Scale Cardinality

When exact distinct counts aren't required (~2% error is acceptable):
```sql
-- Bad: exact, expensive
SELECT COUNT(DISTINCT user_id) FROM events

-- Good: approximate, much faster
SELECT APPROX_COUNT_DISTINCT(user_id) FROM events
```

### SAFE_DIVIDE Over CASE WHEN

```sql
-- Verbose
SELECT CASE WHEN impressions = 0 THEN 0 ELSE clicks / impressions END AS ctr

-- Clean
SELECT SAFE_DIVIDE(clicks, impressions) AS ctr  -- returns NULL on division by zero
```

### Use ARRAY_AGG + UNNEST Instead of Self-Joins

When you need to compare rows within a group, ARRAY_AGG can avoid expensive self-joins:
```sql
-- Instead of joining events to itself
SELECT user_id, ARRAY_AGG(event_name ORDER BY event_time) AS event_sequence
FROM events
GROUP BY user_id
```

### DATE_TRUNC Over EXTRACT for Grouping

```sql
-- Less efficient: EXTRACT loses the date context
SELECT EXTRACT(MONTH FROM order_date) AS month, SUM(total)
FROM orders GROUP BY 1

-- Better: DATE_TRUNC preserves date type, enables partition pruning
SELECT DATE_TRUNC(order_date, MONTH) AS month, SUM(total)
FROM orders GROUP BY 1
```

---

## Things to Avoid

- **JavaScript UDFs in hot paths** — orders of magnitude slower than native SQL functions. Use only when no SQL equivalent exists.
- **SELECT EXCEPT()** in performance-critical queries — forces schema inference at query time.
- **UNION when UNION ALL works** — UNION forces dedup across the full result.
- **STRING join keys when INT64 is available** — integer comparisons are significantly faster.
- **ORDER BY in views or CTEs** — BigQuery ignores it; it misleads readers.

---

## Cost Estimation Flags

When analyzing a query, flag these as cost concerns:
- Full table scans on tables > 1 GB without partition filters
- `SELECT *` on wide tables (many columns)
- Cross joins or joins that could produce large intermediate results
- Repeated scans of the same large table (suggest CTEs or temp tables)
