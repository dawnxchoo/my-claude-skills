---
name: py-clean
description: Clean and prepare DataFrames using pandas. Use when the user needs to handle missing values, remove duplicates, fix data types, standardize strings, parse dates, handle outliers, rename columns, recode values, or assess data quality. Trigger on "clean this data", "fix missing values", "data quality", "prepare for analysis", or any pandas data cleaning operation. 
---

## What This Skill Does

Generates complete, runnable Python code (pandas + numpy) that profiles, cleans, and validates tabular data. The skill walks the user through a structured workflow: profile the data, suggest fixes, get approval, generate code, run it, and produce a cleaning report.

**Assumption**: The user has a CSV, DataFrame, or tabular data file. This skill does NOT perform analysis or visualization. If the user wants charts after cleaning, hand off to py-charts.

**Core principle**: Never modify or overwrite the original data. Always work on a copy.

## Workflow

Follow these steps in order:

### Step 1: Load and Profile the Data

**Pre-profile check:** If column names are unclear or ambiguous (e.g., `col1`, `x`, `val`, `tmp`), ask the user what those columns represent BEFORE profiling. Understanding what a column means changes how we interpret types, values, and issues.

Load the data and run a comprehensive profiling pass. Check ALL of the following:

1. **Shape & size** — row count, column count, memory usage
2. **Column types** — dtypes, identify mismatched types (e.g., numeric stored as object)
3. **Missing values** — null count and % per column, pattern of missingness (random vs systematic)
4. **Duplicate rows** — exact duplicates, near-duplicates (same values in key columns)
5. **Sample values** — head(5), random sample of 5 rows
6. **Numeric columns** — min, max, mean, median, std, skewness, obvious outliers (IQR or z-score)
7. **Categorical columns** — unique count, top values and frequencies, cardinality assessment
8. **String columns** — leading/trailing whitespace, mixed casing, empty strings vs nulls, string length distribution
9. **Date columns** — min/max dates, gaps, mixed formats, timezone presence
10. **Constant columns** — columns with only 1 unique value (suggest removal to user, never auto-drop)
11. **High-cardinality columns** — columns where unique count = row count (possible IDs)
12. **Correlation between columns** — highly correlated numeric columns (>0.95)
13. **Column naming** — inconsistent naming conventions (camelCase vs snake_case vs spaces)

Present this as a clean summary table/report, not a wall of raw output. Flag issues clearly.

### Step 2: Suggest Fixes

Based on the profiling results, generate a numbered list of suggested fixes grouped by category:

> "I found these data quality issues:"
> 1. [issue] — suggested fix
> 2. [issue] — suggested fix
> ...
> "Which of these should I fix? (all / pick numbers / tell me what else)"

Use AskUserQuestion to present the issues. The user picks which to address. Also ask if there are issues the profiling missed that they know about.

### Step 3: Build the Cleaning Plan

Before writing any code, present a plain-language cleaning plan as an ordered list:
- What operation, on which column, with what strategy
- Example: "1. Drop 47 exact duplicate rows. 2. Parse `start_date` to datetime using format `%Y-%m-%d`. 3. Standardize `region` to lowercase. 4. Fill missing `salary` with median by department."

Ask the user to confirm or adjust the plan before proceeding.

### Step 4: Generate the Cleaning Code

Write a complete, runnable Python script. The script must:

1. Load the data from the original file
2. Create a copy immediately (`df_clean = df.copy()`)
3. Apply each cleaning step in order, with a clear comment block per step explaining WHY this step is needed
4. After each step that removes rows, print a count of rows affected
5. Run post-cleaning profiling to show before/after stats
6. Save the cleaned DataFrame to a NEW file (never overwrite the original)
7. Generate the cleaning report (see Step 6)

Follow the Code Template and Best Practices sections below.

### Step 5: Run and Validate

Execute the script using Bash, then show the user:
- Row count before and after
- Null counts before and after
- Dtype changes
- A sample of the cleaned data (head)

If any cleaning step produced unexpected results (e.g., dropped more rows than expected, >20% of rows affected), flag it and ask the user to verify.

### Step 6: Generate Cleaning Report

After cleaning, generate a comprehensive report documenting everything that happened:

**Rows:**
- Total rows before and after cleaning
- Rows removed: count, reason, which cleaning step removed them
- Sample of removed rows (first 5) so user can verify they should have been removed

**Columns:**
- Columns removed: name, reason (e.g., "constant value", "95% null")
- New columns created: name, what it maps to (e.g., "`age_group` — derived from `age` via binning")
- Columns renamed: old name -> new name

**Column-level changes:**
For each column that was modified:
- Column name
- What changed (e.g., "filled 23 nulls with median", "standardized casing to lowercase")
- Count of values affected
- Before/after sample (3-5 example values showing the transformation)

**Summary stats:**
- Before/after comparison: row count, column count, null count, dtype summary
- Total values modified across all steps

Print the report to console AND save as `{input_filename}_cleaning_report.md`.

### Step 7: Iterate

Ask the user if the output looks right and if they want further adjustments. If the data is clean and ready for visualization, mention that py-charts is available.

---

## Cleaning Operations Reference

### Missing Values

Read [references/missing-values.md](references/missing-values.md) for the full strategy guide.

Key operations (all row/column drops require explicit user approval):
- Drop rows where all/most values are null — show affected rows first
- Drop columns above a null threshold — show column name and null %
- Fill numeric nulls (median, mean, mean/median by group, zero, constant, forward fill, backward fill, interpolation, KNN, random sample)
- Fill categorical nulls (mode, mode by group, "Unknown"/"Missing"/"Other", constant)
- Fill datetime nulls (forward fill, backward fill, interpolation)
- Detect sentinel values masking as non-null (-1, 0, 999, 9999, "N/A", "null", "none", "", " ") — convert to NaN
- Flag nulls with a companion boolean column (`col_is_missing`)

### Duplicates

All duplicate removal requires explicit user approval — show count and sample first.
- Remove exact duplicate rows (keep first/last)
- Remove duplicates based on key columns (keep first/last)
- Flag near-duplicates for manual review (add flag column, don't remove)

### Type Conversion & Parsing

Read [references/type-conversion.md](references/type-conversion.md) for conversion recipes.

- String-to-numeric (strip currency symbols, commas, whitespace)
- String-to-datetime (detect format, handle mixed formats)
- String-to-boolean (yes/no, true/false, Y/N, 1/0)
- Numeric-to-categorical (e.g., status codes to labels)
- Object-to-category dtype (for memory efficiency and semantics)
- Downcast numeric types (int64 -> int32 when safe)

### String Standardization

Read [references/string-cleaning.md](references/string-cleaning.md) for regex patterns and recipes.

- Strip leading/trailing whitespace
- Collapse internal multiple spaces to single space
- Standardize casing (lower, upper, title)
- Remove or replace special characters
- Normalize Unicode (accented characters, ligatures)
- Standardize common abbreviations (St/Street, Ave/Avenue, Inc/Incorporated)
- Remove zero-width characters and non-printable characters
- Fix encoding issues (mojibake, UTF-8 vs Latin-1)

### Date Parsing & Standardization

- Parse mixed date formats into consistent datetime
- Standardize timezone (convert all to UTC, or strip timezones)
- Extract date components (year, month, day, day of week)
- Identify and handle impossible dates (Feb 30, future dates where not expected)
- Standardize date format for output (ISO 8601)

### Outlier Handling

- Flag outliers using IQR method (1.5x IQR)
- Flag outliers using z-score (>3 std)
- Cap/floor outliers (winsorization)
- Remove outlier rows — requires user approval, show count and sample
- Keep outliers but add a flag column
- Log-transform heavily skewed columns

### Column Operations

All column drops require explicit user approval.
- Rename columns to snake_case
- Rename columns to remove special characters and spaces
- Drop constant columns — show column name and the constant value
- Drop high-null columns — show column name and null %
- Reorder columns logically (IDs first, then categories, then numerics, then dates)
- Select/drop specific columns

### Recoding & Mapping Values

- Map category values (e.g., "M"/"F" -> "Male"/"Female")
- Bin continuous values into categories (age -> age_group)
- Merge rare categories into "Other"
- Fix typos in categorical values (fuzzy matching)
- Standardize boolean-like values to True/False

### Structural Issues

- Split combined columns (e.g., "City, State" -> two columns)
- Merge related columns
- Pivot/unpivot (wide to long or long to wide)
- Handle mixed data in a single column (numbers and text)

### Validation & Assertions

Post-cleaning sanity checks to confirm the cleaning worked as intended:
- Assert no nulls in required columns — verify columns that should be fully populated actually are
- Assert unique values in ID columns — verify no unexpected duplicates remain after dedup
- Assert values within expected ranges — e.g., age between 0-120, percentages between 0-100
- Assert referential integrity between columns — e.g., `end_date` >= `start_date`
- Assert row count is within expected bounds — catch unexpected mass drops (e.g., lost >10% of rows)
- Generate before/after summary comparison — side-by-side stats for the full dataset

---

## Best Practices

### Code Style
- Clear, clean inline comments — explain the "why", not the "what"
- One comment block per cleaning step
- Use method chaining where it improves readability
- Use `.pipe()` for custom transforms
- Descriptive variable names

### Reproducibility
- **Never overwrite source files** — always create a new copy (`df_clean = df.copy()`)
- Save cleaned data to a new file (e.g., `input_cleaned.csv`)
- Log what changed: before/after row counts, columns modified, values affected
- Include the input filename in the output filename
- Generated script should be fully rerunnable from scratch

### Safety
- Always work on a copy of the DataFrame (`df.copy()`), never modify the original in memory
- **NEVER drop rows or columns without explicit user approval** — show exactly what will be removed (count + sample) and wait for confirmation
- **NEVER remove duplicates without explicit user approval** — show the duplicates first
- Show the user how many rows/values will be affected before executing ANY destructive operation
- Use `errors='coerce'` for type conversions so bad values become NaN instead of crashing
- After each cleaning step, assert row count hasn't dropped unexpectedly
- Save the cleaning script alongside the output so the user can audit/replay
- Warn if a cleaning step affects >20% of rows (might indicate a logic issue)
- Never delete or overwrite the original data file
- All changes must be traceable in the cleaning report

---

## Code Template

Every generated script should follow this structure:

```python
import pandas as pd
import numpy as np

# --- Load ---
df = pd.read_csv('input.csv')
print(f"Loaded: {df.shape[0]:,} rows, {df.shape[1]} columns")

# --- Profile (before) ---
# ... profiling code ...

# --- Clean (always work on a copy) ---
df_clean = df.copy()

# Step 1: [description — why this step is needed]
# ...
print(f"After step 1: {df_clean.shape[0]:,} rows")

# Step N: [description — why this step is needed]
# ...
print(f"After step N: {df_clean.shape[0]:,} rows")

# --- Profile (after) ---
# ... before/after comparison ...

# --- Cleaning Report ---
# ... generate report (see Step 6) ...

# --- Save (new file, never overwrite original) ---
df_clean.to_csv('input_cleaned.csv', index=False)
print(f"Saved: {df_clean.shape[0]:,} rows, {df_clean.shape[1]} columns → input_cleaned.csv")
```

This is a starting point, not a rigid template. Adapt as needed for the specific data and cleaning operations.
