# Missing Values — Strategy Guide

## Decision Tree: What to Do with Missing Values

### When to Drop Rows
- Row has ALL values null → safe to drop (confirm with user)
- Row has >80% of columns null → suggest dropping (confirm with user)
- A small number of rows (<1%) have nulls in critical columns → dropping is low-risk (confirm with user)

### When to Drop Columns
- Column is >70% null AND not critical to analysis → suggest dropping (confirm with user)
- Column is >90% null → strongly suggest dropping (confirm with user)
- Always show the user the column name, null %, and sample non-null values before dropping

### When to Fill

Choose the fill strategy based on the column's dtype and the data context:

---

## Numeric Fill Strategies

| Strategy | When to Use | Pandas Code |
|---|---|---|
| **Median** (per column) | Default for skewed numeric data. Resistant to outliers. | `df['col'].fillna(df['col'].median())` |
| **Mean** (per column) | Normally distributed numeric data with no extreme outliers. | `df['col'].fillna(df['col'].mean())` |
| **Median by group** | When the value depends on a category (e.g., salary by department). Better than global median. | `df['col'].fillna(df.groupby('group')['col'].transform('median'))` |
| **Mean by group** | Same as above, for normally distributed data. | `df['col'].fillna(df.groupby('group')['col'].transform('mean'))` |
| **Zero** | When null genuinely means "none" (e.g., count of events, revenue for a new product). | `df['col'].fillna(0)` |
| **Constant value** | User specifies a meaningful default. | `df['col'].fillna(value)` |
| **Forward fill** | Time series where the last known value carries forward (e.g., stock price, sensor reading). | `df['col'].ffill()` |
| **Backward fill** | Time series where you want to use the next known value. Less common than forward fill. | `df['col'].bfill()` |
| **Linear interpolation** | Evenly spaced time series. Assumes linear change between known points. | `df['col'].interpolate(method='linear')` |
| **Time-based interpolation** | Irregularly spaced time series. Respects actual time gaps. | `df['col'].interpolate(method='time')` |
| **KNN imputation** | When the value correlates with other columns. Uses k-nearest neighbors to impute. Requires sklearn. | `from sklearn.impute import KNNImputer; imputer = KNNImputer(n_neighbors=5); df[num_cols] = imputer.fit_transform(df[num_cols])` |
| **Random sample** | Preserves the original distribution. Good when you don't want to bias toward the center. | `df['col'].fillna(df['col'].dropna().sample(df['col'].isna().sum(), replace=True).values)` |

**Choosing between median and mean:**
- Use **median** when the data is skewed or has outliers (most real-world data)
- Use **mean** only when the data is roughly symmetric and has no extreme values
- When in doubt, use median — it's safer

**Choosing between global and group-based fill:**
- Use **group-based** when there's a natural grouping variable that explains the variation (department, region, category)
- Use **global** when no meaningful grouping exists or when groups are too small

---

## Categorical Fill Strategies

| Strategy | When to Use | Pandas Code |
|---|---|---|
| **Mode** (most frequent) | Default for categorical data. Uses the most common value. | `df['col'].fillna(df['col'].mode()[0])` |
| **Mode by group** | When the category depends on another column (e.g., default shipping method by region). | `df['col'].fillna(df.groupby('group')['col'].transform(lambda x: x.mode()[0] if not x.mode().empty else np.nan))` |
| **"Unknown"** | When the absence of a value is meaningful and should be preserved as its own category. | `df['col'].fillna('Unknown')` |
| **"Missing"** | Same as "Unknown" but more explicit about the reason. | `df['col'].fillna('Missing')` |
| **"Other"** | When nulls should be grouped with rare categories. | `df['col'].fillna('Other')` |
| **Constant value** | User specifies a domain-appropriate default. | `df['col'].fillna('value')` |

**When to use "Unknown" vs fill with mode:**
- Use **"Unknown"** when the fact that data is missing is informative (e.g., a customer didn't provide their industry)
- Use **mode** when the missing data is likely just a data entry gap and the most common value is a reasonable assumption

---

## Datetime Fill Strategies

| Strategy | When to Use | Pandas Code |
|---|---|---|
| **Forward fill** | Time series with missing timestamps. Carries the last known datetime forward. | `df['col'].ffill()` |
| **Backward fill** | Fill with the next known datetime. | `df['col'].bfill()` |
| **Interpolation** | Evenly spaced time series with missing entries. | `df['col'].interpolate(method='time')` |

---

## Sentinel Value Detection

Common sentinel values that mask as non-null but actually represent missing data:

| Sentinel | Common In | Detection |
|---|---|---|
| `-1` | IDs, counts, codes | `df[df['col'] == -1]` |
| `0` | Revenue, quantity (when 0 is impossible) | Context-dependent |
| `999`, `9999`, `99999` | Numeric codes, placeholder values | `df[df['col'].isin([999, 9999, 99999])]` |
| `"N/A"`, `"n/a"`, `"NA"` | String columns from manual entry | `df[df['col'].str.strip().str.lower().isin(['n/a', 'na'])]` |
| `"null"`, `"NULL"`, `"None"` | String columns from system exports | `df[df['col'].str.strip().str.lower().isin(['null', 'none'])]` |
| `""` (empty string) | String columns | `df[df['col'] == '']` |
| `" "` (whitespace only) | String columns | `df[df['col'].str.strip() == '']` |
| `"TBD"`, `"TBA"`, `"pending"` | Status/text columns | `df[df['col'].str.strip().str.lower().isin(['tbd', 'tba', 'pending'])]` |
| `1900-01-01`, `1970-01-01` | Date columns (epoch defaults) | `df[df['col'] == pd.Timestamp('1900-01-01')]` |

**How to handle:** Convert sentinel values to `np.nan` first, then apply the appropriate fill strategy.

```python
# Convert sentinels to NaN
sentinels = [-1, 999, 9999]
df['col'] = df['col'].replace(sentinels, np.nan)

# For string sentinels
string_sentinels = ['N/A', 'n/a', 'NA', 'null', 'NULL', 'None', 'none', '', 'TBD']
df['col'] = df['col'].replace(string_sentinels, np.nan)
df['col'] = df['col'].apply(lambda x: np.nan if isinstance(x, str) and x.strip() == '' else x)
```

---

## Flag-Only Strategy

Sometimes you want to note that a value was missing without filling it, or alongside filling it:

```python
# Add a flag column, then fill
df['salary_was_missing'] = df['salary'].isna()
df['salary'] = df['salary'].fillna(df['salary'].median())
```

This preserves the information that the value was imputed, which can matter for downstream analysis.
