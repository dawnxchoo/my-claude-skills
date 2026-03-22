# Type Conversion — Recipe Guide

## String to Numeric

### Basic Conversion
```python
# Safe conversion — bad values become NaN instead of crashing
df['col'] = pd.to_numeric(df['col'], errors='coerce')
```

### Strip Currency Symbols and Formatting
```python
# Remove $, commas, whitespace, then convert
df['revenue'] = (
    df['revenue']
    .str.replace(r'[$,\s]', '', regex=True)
    .pipe(pd.to_numeric, errors='coerce')
)
```

### Handle Percentage Strings
```python
# "45.2%" -> 0.452
df['rate'] = (
    df['rate']
    .str.replace('%', '', regex=False)
    .pipe(pd.to_numeric, errors='coerce')
    .div(100)
)
```

### Handle Parentheses for Negative Numbers
```python
# "(1,234.56)" -> -1234.56 (accounting format)
df['amount'] = (
    df['amount']
    .str.replace(r'[($,\s)]', '', regex=True)
    .str.replace(r'^\((.*)\)$', r'-\1', regex=True)
    .pipe(pd.to_numeric, errors='coerce')
)
```

### European Number Formats
```python
# "1.234,56" -> 1234.56 (period as thousands separator, comma as decimal)
df['col'] = (
    df['col']
    .str.replace('.', '', regex=False)   # remove thousands separator
    .str.replace(',', '.', regex=False)  # swap decimal separator
    .pipe(pd.to_numeric, errors='coerce')
)
```

**Warning:** If the data has a mix of US and European formats, ask the user to clarify before converting.

---

## String to Datetime

### Basic Conversion
```python
# Let pandas infer the format
df['date'] = pd.to_datetime(df['date'], errors='coerce')

# Specify format for consistency and speed
df['date'] = pd.to_datetime(df['date'], format='%Y-%m-%d', errors='coerce')
```

### Common Format Strings

| Format | Example | Code |
|---|---|---|
| ISO 8601 | `2024-01-15` | `format='%Y-%m-%d'` |
| US date | `01/15/2024` | `format='%m/%d/%Y'` |
| European date | `15/01/2024` | `format='%d/%m/%Y'` |
| Long date | `January 15, 2024` | `format='%B %d, %Y'` |
| Short month | `15-Jan-2024` | `format='%d-%b-%Y'` |
| Datetime | `2024-01-15 14:30:00` | `format='%Y-%m-%d %H:%M:%S'` |
| Unix timestamp | `1705276800` | `pd.to_datetime(df['col'], unit='s')` |
| Millisecond timestamp | `1705276800000` | `pd.to_datetime(df['col'], unit='ms')` |

### Mixed Date Formats
```python
# When a column has multiple formats, parse in order of specificity
def parse_mixed_dates(series):
    result = pd.to_datetime(series, format='%Y-%m-%d', errors='coerce')
    mask = result.isna()
    result[mask] = pd.to_datetime(series[mask], format='%m/%d/%Y', errors='coerce')
    mask = result.isna()
    result[mask] = pd.to_datetime(series[mask], format='%d-%b-%Y', errors='coerce')
    return result

df['date'] = parse_mixed_dates(df['date'])
```

### Ambiguous Dates (01/02/2024 — Jan 2 or Feb 1?)
Always ask the user: "Is `01/02/2024` January 2nd (US format) or February 1st (European format)?"
- US: `dayfirst=False` (default)
- European: `dayfirst=True`

```python
df['date'] = pd.to_datetime(df['date'], dayfirst=True, errors='coerce')
```

### Timezone Handling
```python
# Convert to UTC
df['date'] = df['date'].dt.tz_convert('UTC')

# Strip timezone (make naive)
df['date'] = df['date'].dt.tz_localize(None)

# Add timezone to naive datetime
df['date'] = df['date'].dt.tz_localize('US/Eastern')
```

---

## String to Boolean

### Common Truthy/Falsy Patterns
```python
true_values = {'yes', 'y', 'true', 't', '1', 'on', 'active', 'enabled'}
false_values = {'no', 'n', 'false', 'f', '0', 'off', 'inactive', 'disabled'}

def to_boolean(series):
    lowered = series.str.strip().str.lower()
    result = pd.Series(np.nan, index=series.index)
    result[lowered.isin(true_values)] = True
    result[lowered.isin(false_values)] = False
    return result.astype('boolean')  # nullable boolean dtype

df['is_active'] = to_boolean(df['is_active'])
```

### Using pd.read_csv Built-in
```python
# Specify true/false values at read time
df = pd.read_csv('file.csv',
    true_values=['yes', 'Y', 'True', '1'],
    false_values=['no', 'N', 'False', '0']
)
```

---

## Numeric to Categorical

### Status Codes to Labels
```python
status_map = {1: 'Active', 2: 'Inactive', 3: 'Pending', 4: 'Cancelled'}
df['status_label'] = df['status_code'].map(status_map)
```

### Binning Continuous Values
```python
# Equal-width bins
df['age_group'] = pd.cut(df['age'], bins=[0, 18, 30, 50, 65, 120],
                         labels=['Under 18', '18-30', '31-50', '51-65', '65+'])

# Quantile-based bins (equal-frequency)
df['income_quartile'] = pd.qcut(df['income'], q=4,
                                 labels=['Q1', 'Q2', 'Q3', 'Q4'])
```

---

## Object to Category Dtype

Convert string columns with limited unique values to `category` dtype for memory efficiency:

```python
# Good candidate: column with <50 unique values relative to total rows
if df['region'].nunique() / len(df) < 0.05:  # less than 5% unique
    df['region'] = df['region'].astype('category')
```

**When to use:** Columns with a small set of repeating values (region, status, department).
**When NOT to use:** Free-text columns, IDs, columns with mostly unique values.

---

## Downcast Numeric Types

Reduce memory usage by using smaller numeric types when safe:

```python
# Downcast integers
df['count'] = pd.to_numeric(df['count'], downcast='integer')

# Downcast floats
df['price'] = pd.to_numeric(df['price'], downcast='float')
```

**Safety check:** Only downcast when you've confirmed the value range fits the smaller type. `int8` holds -128 to 127, `int16` holds -32768 to 32767.
