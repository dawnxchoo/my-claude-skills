---
name: py-charts
description: Generate publication-quality Python charts using matplotlib and seaborn. Use this skill whenever the user wants to create a chart, plot, graph, or data visualization — including bar charts, line charts, scatter plots, heatmaps, histograms, box plots, or any other visual representation of data. Trigger this even if the user just says "visualize this", "plot this", "graph it", "make a chart", or mentions matplotlib/seaborn. This skill is specifically designed for data scientists and analysts who need clean, professional visuals.
---

## What This Skill Does

Generates complete, runnable Python code (matplotlib + seaborn) that produces publication-quality charts. The skill handles chart type selection, color palette application, labeling, formatting, and best-practice styling — then runs the code and shows the user the resulting chart.

**Assumption**: The data is already loaded or aggregated. This skill does NOT perform data analysis, cleaning, or transformation. If the user's data needs prep work, help them with that first, then invoke this workflow for the visualization step.

## Workflow

Follow these steps in order:

### Step 1: Understand the Data

Before writing any code, understand what you're working with:
- What columns/fields exist? What are their types (numeric, categorical, datetime)?
- What's the shape of the data — how many rows, how many categories?
- What's the value range? This matters for axis scaling and number formatting.

If the user provides a file or dataframe, read/inspect it. If they describe the data verbally, confirm your understanding before proceeding.

### Step 2: Ask for Color Palette

Present the user with palette options using AskUserQuestion. The choices are:

1. **Modern** — Vibrant and contemporary, feels fresh and polished (default if the user has no preference)
2. **Grayscale + Blue Accent** — Clean and understated, blue highlights key findings
3. **Corporate** — Polished and professional, consulting-deck energy
4. **Bold** — High contrast, vibrant, pops on slides and social media
5. **Soft** — Muted and approachable, great for blog posts and portfolios
6. **Other** — Describe what you want

Read [references/palettes.md](references/palettes.md) for the exact hex codes and usage guidance for whichever palette is selected.

### Step 3: Ask About Highlights (If Applicable)

If the data has categories or notable data points, ask:
> "Are there any specific data points or categories you want to highlight?"

Present concrete options based on what you found in the data (e.g., specific category names, a threshold, a time period). The user can pick from those or skip. If they skip, no highlighting is applied.

When highlighting: use the palette's primary color for the highlighted element and render everything else in light gray (#D0D0D0). This creates a clear "signal vs. context" visual.

### Step 4: Select Chart Type

Choose the right chart type based on the data and what the user wants to show. Use this guide:

| Chart Type | When to Use |
|---|---|
| **Bar (vertical)** | Comparing values across categories. The default workhorse. |
| **Bar (horizontal)** | Same as vertical, but use when labels are long or there are many categories. |
| **Line** | Trends over time or continuous ordered data. Multiple lines OK for 2-3 series. |
| **Scatter** | Relationship between two continuous variables. Add trendline if relationship matters. |
| **Heatmap** | Patterns across two categorical dimensions (correlation matrix, pivot table). |
| **Histogram** | Distribution of a single continuous variable. |
| **Box / Violin** | Comparing distributions across categories, especially for spotting outliers. |
| **Stacked bar** | Composition (parts of a whole) across categories when you also want to compare totals. |
| **Pie** | ONLY when there are exactly 2 categories showing a simple split (e.g., 70/30). |
| **Small multiples** | When you'd otherwise cram too many series into one chart. Split into a grid with shared scales. |

**Never use**: 3D charts of any kind, pie charts with more than 2 slices, or area charts when a line chart would work.

### Step 5: Generate the Code

Write complete, runnable Python code from imports through `plt.savefig()`. Apply every relevant best practice from the section below. Include inline comments only where the logic isn't self-evident (e.g., explaining a specific formatting choice or transform).

Save the chart to a PNG file (e.g., `chart.png`) with `dpi=150` and `bbox_inches='tight'`.

### Step 6: Run and Display

Execute the Python script using Bash, then display the resulting PNG to the user using the Read tool so they can see the chart immediately. If the script errors, diagnose and fix.

### Step 7: Iterate

Ask the user if they want any changes. Common requests: adjust colors, change chart type, modify labels, resize, add/remove elements. Apply changes and re-run.

---

## Charting Best Practices

Apply these principles to every chart you generate. They're organized by what they address.

### Layout & Structure

- **Set figure size intentionally.** Don't use matplotlib's default (6.4x4.8). Size for the destination — wider for slides (12x6), squarer for notebooks (8x6), taller for many categories.
- **Use `constrained_layout=True`** to prevent labels from getting clipped.
- **Remove all spines** and use a rounded plot background with a subtle border for a clean, modern frame.
- **Use small multiples** instead of cramming many series into one overloaded chart. Shared scales across panels so they're directly comparable.

### Text & Labels

- **Every axis gets a descriptive label with units** — "Revenue (USD)" not just "Revenue".
- **Titles should state the insight**, not describe the axes — "Sales peaked in Q3" not "Sales by Quarter". If the user hasn't specified an insight, use a descriptive title and suggest they replace it with a takeaway.
- **Format numbers in titles the same way you'd format them on axes** — use dollar signs, percent symbols, commas, and proper spacing. Titles are natural language sentences, so numbers should read naturally.
- **Be aware that matplotlib interprets special characters** — `$` triggers LaTeX math mode, `%` can cause string formatting issues, `_` renders as subscript, etc. When putting formatted numbers or symbols in titles, labels, or annotations, escape them or use raw strings so the text renders as intended. Always visually verify that titles display correctly.
- **Use a clean sans-serif font** — set `plt.rcParams['font.family'] = 'sans-serif'` so the OS picks the best available (Helvetica on macOS, Arial on Windows, DejaVu Sans on Linux). Minimum 10pt for labels, 12pt+ for titles.
- **Direct-label data points or bars** instead of using legends when practical — it reduces eye travel.
- **Rotate x-axis labels only when necessary.** Prefer horizontal text, abbreviations, or switching to a horizontal bar chart.
- **Format numbers by data type** — currency gets `$`, percentages get `%`, dates get readable formats. Only show decimal places that are meaningful. Scale to the data range: if revenue is $8K–$15K, show "$10.3K" not "$10,354.32".

### Color

- **Color encodes meaning, not decoration.** Default to a single accent color for the key finding, with grays for context.
- **Use sequential palettes** for ordered data, **diverging** for data with a meaningful midpoint, **qualitative** for categories.
- All palette options provided by this skill are colorblind-safe.

### Data Ink & Clutter

- **Maximize data-ink ratio** — every visual element should represent data or help interpret it. If it does neither, remove it.
- **No gridlines.** Remove them entirely — direct labels on data points and bars make gridlines unnecessary. They add clutter without adding information.
- **No 3D charts.** Ever. They distort perception of values.

### Style

- **Use rounded corners on bars** — replace `ax.bar()` with `FancyBboxPatch` rectangles using `boxstyle="round,pad=0,rounding_size=..."`. This creates a softer, more contemporary look. Adjust `rounding_size` relative to bar dimensions (typically 0.05–0.15).
- **Use a rounded plot background** — remove all spines and add a `FancyBboxPatch` behind the axes with rounded corners and a subtle light gray border (`#E5E7EB`). This frames the chart without harsh edges.

### Scale & Accuracy

- **Bar charts must start y-axis at zero** — truncating exaggerates differences.
- **Line charts can use non-zero baselines** for trends, but label the axis clearly.
- **Keep aspect ratios sensible** — don't stretch or compress to exaggerate slopes.
- **Consistent scales across small multiples** so panels are directly comparable.
- **Offer log scale** when the data range is large or heavily skewed — present this as an option to the user.

### Accessibility

- **Don't rely on color alone** to convey information — pair with markers, patterns, line styles, or annotations. Someone printing in grayscale or with color vision deficiency should still be able to read the chart.

---

## Code Template

Every generated script should follow this general structure:

```python
import matplotlib.pyplot as plt
import seaborn as sns
import matplotlib.ticker as mticker  # if needed for formatting
from matplotlib.patches import FancyBboxPatch

plt.rcParams['font.family'] = 'sans-serif'

# Data setup
# ...

# Create figure
fig, ax = plt.subplots(figsize=(W, H), constrained_layout=True)

# Remove all spines
for spine in ax.spines.values():
    spine.set_visible(False)

# Rounded plot background
bg = FancyBboxPatch(
    (0, 0), 1, 1, transform=ax.transAxes,
    boxstyle="round,pad=0.02,rounding_size=0.03",
    facecolor='white', edgecolor='#E5E7EB', linewidth=1, zorder=-1
)
ax.add_patch(bg)

# Plot (for bar charts, use FancyBboxPatch for rounded bars)
# ...

# Labels, title, formatting
ax.set_xlabel('Label (units)', fontsize=11)
ax.set_ylabel('Label (units)', fontsize=11)
ax.set_title('Insight-driven title', fontsize=13, fontweight='bold')

# No gridlines — direct labels carry the information

# Save
plt.savefig('chart.png', dpi=150, bbox_inches='tight', facecolor='white')
print("Chart saved to chart.png")
```

This is a starting point, not a rigid template. Adapt as needed for the specific chart type and data.
