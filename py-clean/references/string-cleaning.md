# String Cleaning — Recipe Guide

## Whitespace

### Strip Leading/Trailing Whitespace
```python
df['col'] = df['col'].str.strip()
```

### Collapse Internal Multiple Spaces
```python
# "John   Doe" -> "John Doe"
df['col'] = df['col'].str.replace(r'\s+', ' ', regex=True).str.strip()
```

### Remove All Whitespace
```python
# "A B C" -> "ABC" (use for codes, IDs, phone numbers)
df['col'] = df['col'].str.replace(r'\s', '', regex=True)
```

### Remove Zero-Width Characters and Non-Printable Characters
```python
import re
# Remove zero-width spaces, soft hyphens, BOM, and other invisible characters
df['col'] = df['col'].str.replace(r'[\u200b\u200c\u200d\ufeff\u00ad\u2060]', '', regex=True)

# Remove all non-printable characters (keep printable ASCII + common Unicode)
df['col'] = df['col'].apply(lambda x: re.sub(r'[^\x20-\x7E\u00A0-\uFFFF]', '', x) if isinstance(x, str) else x)
```

---

## Case Standardization

### Lowercase
```python
# Best for: matching, deduplication, lookups
df['col'] = df['col'].str.lower()
```

### Uppercase
```python
# Best for: codes, abbreviations (US, CA, NY)
df['col'] = df['col'].str.upper()
```

### Title Case
```python
# Best for: names, titles
df['col'] = df['col'].str.title()
```

**Title case pitfalls:** `"JOHN O'BRIEN"` becomes `"John O'Brien"` (correct), but `"IBM"` becomes `"Ibm"` (wrong). For columns with acronyms, consider a custom approach:

```python
# Preserve known acronyms
acronyms = {'ibm': 'IBM', 'usa': 'USA', 'nyc': 'NYC', 'api': 'API'}
def smart_title(s):
    if not isinstance(s, str):
        return s
    words = s.lower().split()
    return ' '.join(acronyms.get(w, w.title()) for w in words)

df['col'] = df['col'].apply(smart_title)
```

---

## Special Character Handling

### Remove Special Characters (Keep Alphanumeric + Spaces)
```python
df['col'] = df['col'].str.replace(r'[^a-zA-Z0-9\s]', '', regex=True)
```

### Remove Specific Characters
```python
# Remove hashtags, @mentions, etc.
df['col'] = df['col'].str.replace(r'[#@]', '', regex=True)
```

### Replace Special Characters with Alternatives
```python
# Replace & with "and"
df['col'] = df['col'].str.replace('&', 'and', regex=False)
```

---

## Unicode Normalization

### Remove Accents/Diacritics
```python
import unicodedata

def remove_accents(s):
    if not isinstance(s, str):
        return s
    # NFKD decomposition separates base characters from combining marks
    normalized = unicodedata.normalize('NFKD', s)
    return ''.join(c for c in normalized if not unicodedata.combining(c))

df['col'] = df['col'].apply(remove_accents)
# "café" -> "cafe", "naïve" -> "naive", "José" -> "Jose"
```

### Normalize Unicode Forms
```python
# NFKC: compatibility composition (best for matching)
# Converts things like "ﬁ" (ligature) to "fi", "²" to "2"
df['col'] = df['col'].apply(lambda x: unicodedata.normalize('NFKC', x) if isinstance(x, str) else x)
```

---

## Encoding Issues

### Detect Encoding
```python
import chardet

# Read raw bytes to detect encoding
with open('file.csv', 'rb') as f:
    result = chardet.detect(f.read(10000))  # sample first 10KB
print(result)  # {'encoding': 'utf-8', 'confidence': 0.99}

# Read with detected encoding
df = pd.read_csv('file.csv', encoding=result['encoding'])
```

### Fix Mojibake (Double-Encoded Text)
```python
# "CafÃ©" -> "Café" (UTF-8 interpreted as Latin-1)
def fix_mojibake(s):
    if not isinstance(s, str):
        return s
    try:
        return s.encode('latin-1').decode('utf-8')
    except (UnicodeDecodeError, UnicodeEncodeError):
        return s

df['col'] = df['col'].apply(fix_mojibake)
```

### Common Encoding Pairs

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Ã©` instead of `é` | UTF-8 read as Latin-1 | `.encode('latin-1').decode('utf-8')` |
| `Ã¨` instead of `è` | Same | Same |
| `â€™` instead of `'` | UTF-8 read as Windows-1252 | `.encode('cp1252').decode('utf-8')` |
| `Â` appearing before characters | Extra byte from encoding mismatch | Strip `Â` or fix encoding |

---

## Common Standardization Patterns

### Phone Numbers
```python
# Normalize to digits only, then format
df['phone'] = df['phone'].str.replace(r'[^\d]', '', regex=True)
# "10" digits -> "(XXX) XXX-XXXX"
df['phone'] = df['phone'].apply(
    lambda x: f"({x[:3]}) {x[3:6]}-{x[6:]}" if isinstance(x, str) and len(x) == 10 else x
)
```

### Email Addresses
```python
# Lowercase and strip
df['email'] = df['email'].str.strip().str.lower()

# Basic validation
email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
df['email_valid'] = df['email'].str.match(email_pattern, na=False)
```

### US State Abbreviations
```python
state_map = {
    'alabama': 'AL', 'alaska': 'AK', 'arizona': 'AZ', 'arkansas': 'AR',
    'california': 'CA', 'colorado': 'CO', 'connecticut': 'CT', 'delaware': 'DE',
    'florida': 'FL', 'georgia': 'GA', 'hawaii': 'HI', 'idaho': 'ID',
    'illinois': 'IL', 'indiana': 'IN', 'iowa': 'IA', 'kansas': 'KS',
    'kentucky': 'KY', 'louisiana': 'LA', 'maine': 'ME', 'maryland': 'MD',
    'massachusetts': 'MA', 'michigan': 'MI', 'minnesota': 'MN', 'mississippi': 'MS',
    'missouri': 'MO', 'montana': 'MT', 'nebraska': 'NE', 'nevada': 'NV',
    'new hampshire': 'NH', 'new jersey': 'NJ', 'new mexico': 'NM', 'new york': 'NY',
    'north carolina': 'NC', 'north dakota': 'ND', 'ohio': 'OH', 'oklahoma': 'OK',
    'oregon': 'OR', 'pennsylvania': 'PA', 'rhode island': 'RI', 'south carolina': 'SC',
    'south dakota': 'SD', 'tennessee': 'TN', 'texas': 'TX', 'utah': 'UT',
    'vermont': 'VT', 'virginia': 'VA', 'washington': 'WA', 'west virginia': 'WV',
    'wisconsin': 'WI', 'wyoming': 'WY'
}
df['state'] = df['state'].str.strip().str.lower().map(state_map).fillna(df['state'].str.upper().str.strip())
```

### Company Name Suffixes
```python
# Standardize Inc./Inc/Incorporated, LLC/L.L.C., Corp./Corporation, etc.
suffix_map = {
    r'\bInc\.?\b': 'Inc',
    r'\bIncorporated\b': 'Inc',
    r'\bL\.?L\.?C\.?\b': 'LLC',
    r'\bLimited Liability Company\b': 'LLC',
    r'\bCorp\.?\b': 'Corp',
    r'\bCorporation\b': 'Corp',
    r'\bLtd\.?\b': 'Ltd',
    r'\bLimited\b': 'Ltd',
    r'\bCo\.?\b': 'Co',
    r'\bCompany\b': 'Co',
}
for pattern, replacement in suffix_map.items():
    df['company'] = df['company'].str.replace(pattern, replacement, regex=True, flags=re.IGNORECASE)
```

### Address Standardization
```python
address_abbrevs = {
    r'\bStreet\b': 'St',
    r'\bAvenue\b': 'Ave',
    r'\bBoulevard\b': 'Blvd',
    r'\bDrive\b': 'Dr',
    r'\bLane\b': 'Ln',
    r'\bRoad\b': 'Rd',
    r'\bCourt\b': 'Ct',
    r'\bPlace\b': 'Pl',
    r'\bApartment\b': 'Apt',
    r'\bSuite\b': 'Ste',
}
for pattern, replacement in address_abbrevs.items():
    df['address'] = df['address'].str.replace(pattern, replacement, regex=True, flags=re.IGNORECASE)
```
