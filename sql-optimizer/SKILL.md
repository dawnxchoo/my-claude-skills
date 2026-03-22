---
name: sql-optimizer
description: Review and optimize SQL queries for performance. Use this skill whenever the user pastes a SQL query and asks to optimize, review, improve, speed up, or fix performance — or mentions slow queries, query cost, or asks "why is this query slow". Also trigger when the user asks to rewrite SQL for a specific dialect. Supports BigQuery, PostgreSQL, MySQL, SQL Server, and Snowflake. Trigger on phrases like "optimize this query", "this query is slow", "review my SQL", "rewrite for BigQuery", or simply pasting SQL and asking for help.
---

## What This Skill Does

Takes a SQL query, detects the dialect, analyzes it for performance anti-patterns and optimization opportunities, and returns an optimized version with explanations of what changed and why.

**Does NOT**: execute queries, access table schemas or execution plans (unless the user provides them), or perform data analysis.

## Workflow

### Step 1: Receive the Query

Accept the SQL query — pasted inline or from a file reference. If the query is a fragment or ambiguous, ask for the full query before proceeding.

### Step 2: Detect & Confirm Dialect

Infer the dialect from syntax signals:

| Signal | Dialect |
|---|---|
| Backtick-quoted `project.dataset.table` paths | BigQuery |
| `::` type casting, `ILIKE`, `LATERAL` | PostgreSQL |
| Backtick identifiers + `LIMIT`, `AUTO_INCREMENT` | MySQL |
| Square bracket identifiers `[table]`, `TOP`, `CROSS APPLY` | SQL Server |
| `FLATTEN`, `VARIANT`, `TRY_CAST`, `QUALIFY` + `FLATTEN` | Snowflake |

Always confirm with the user: "This looks like [dialect] — is that correct?"

If ambiguous (standard SQL with no dialect markers), ask: "Which SQL dialect are you using? (BigQuery / PostgreSQL / MySQL / SQL Server / Snowflake / Standard SQL)"

Once confirmed, load the corresponding reference file from [references/](references/).

### Step 3: Understand Context

Ask dialect-appropriate questions (unless the user says "just optimize it"):

**All dialects:**
- Approximate table sizes (thousands? millions? billions of rows?)
- Do you have an EXPLAIN / execution plan you can share?
- What is the query for? (ad-hoc analysis / scheduled pipeline / dashboard)

**Index-based dialects** (PostgreSQL, MySQL, SQL Server):
- Are there indexes on the columns used in WHERE clauses and JOINs? If so, which columns?

**Partition-based dialects** (BigQuery, Snowflake):
- Is the table partitioned? If so, on which column?
- Are there clustering keys / clustering columns?

If the user says "just optimize it" or provides no context, proceed with sensible defaults: assume medium-large tables, prioritize both performance and readability.

### Step 4: Analyze the Query

Walk through the query and identify issues using [references/anti-patterns.md](references/anti-patterns.md) and the dialect-specific reference file.

Categorize each finding:

| Severity | Meaning |
|---|---|
| **Critical** | Will cause significant performance degradation — full table scans, Cartesian joins, correlated subqueries on large tables |
| **Warning** | Likely suboptimal but depends on data — unnecessary DISTINCT, ORDER BY in CTEs, implicit type conversions |
| **Suggestion** | Readability or style improvements that may also help the optimizer — CTE restructuring, consistent aliasing, explicit JOIN types |

### Step 5: Generate Optimized Query

Produce the rewritten SQL:
- Clear formatting: indentation, uppercase keywords, aligned clauses
- Inline comments on changed lines: `-- OPTIMIZED: reason`
- Preserve the original query's logic and output — never change what the query returns, only how it computes it

### Step 6: Explain Changes

Present a summary table:

```
| # | Change | Category | Severity | Why |
|---|--------|----------|----------|-----|
| 1 | Replaced correlated subquery with JOIN | Joins | Critical | Subquery executed once per row |
| 2 | Added partition filter | Filtering | Critical | Without it, scans entire table |
| ... | ... | ... | ... | ... |
```

After the table, provide a brief narrative explaining the most impactful changes in plain language.

### Step 7: Iterate

Ask if the user wants to:
- See original vs optimized side by side
- Adjust for a different dialect
- Prioritize readability over performance (or vice versa)
- Get a deeper explanation of any specific change

---

## General Optimization Principles

Always apply these regardless of dialect:

1. **Filter early, join late.** Push WHERE conditions as close to the source tables as possible.
2. **No SELECT \*.** Specify only the columns needed — reduces I/O and enables covering indexes (row stores) or reduces bytes scanned (columnar stores).
3. **JOINs over correlated subqueries.** Correlated subqueries execute per-row; JOINs let the optimizer choose a strategy.
4. **CTEs for readability, with awareness.** CTE materialization varies by dialect — see dialect-specific references.
5. **No functions on indexed/partitioned columns in WHERE.** `WHERE YEAR(date_col) = 2024` prevents index and partition pruning. Rewrite as a range.
6. **Explicit JOIN types.** Never use implicit comma-joins. Always `JOIN ... ON`.
7. **DISTINCT is a red flag.** If you need DISTINCT, the upstream logic may be producing duplicates — fix the root cause.
8. **ORDER BY only at the outermost query.** Sorting in subqueries/CTEs is wasted or ignored.
9. **UNION ALL over UNION.** When duplicates aren't a concern, avoid the sort + dedup cost.
10. **Match data types.** Implicit type conversions in joins and filters prevent index usage and can cause subtle bugs.
