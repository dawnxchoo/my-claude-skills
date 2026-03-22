# Color Palettes

All palettes are colorblind-accessible (verified via perceptual distance testing across protanopia, deuteranopia, and tritanopia simulations). Every option here is safe to use. Each palette has 4 colors in priority order — if a chart only needs 2 categories, use colors 1 and 2.

## 1. Modern (Default)

Vibrant and contemporary. Feels fresh and modern — great for dashboards, social media, and any context where you want charts that pop.

| Priority | Role    | Hex     |
|----------|---------|---------|
| 1        | Indigo  | #6366F1 |
| 2        | Coral   | #FB923C |
| 3        | Teal    | #2DD4BF |
| 4        | Rose    | #F472B6 |

## 2. Grayscale + Blue Accent

Clean and understated. Blue highlights the key finding; grays provide context. Works well for technical reports and academic presentations.

| Priority | Role    | Hex     |
|----------|---------|---------|
| 1        | Accent  | #3174A1 |
| 2        | Dark    | #4A4A4A |
| 3        | Medium  | #8C8C8C |
| 4        | Light   | #C0C0C0 |

## 3. Corporate

Polished and professional. Navy, burnt sienna, and neutral tones. Consulting-deck energy — great for stakeholder presentations and business reports.

| Priority | Role    | Hex     |
|----------|---------|---------|
| 1        | Primary | #2874A6 |
| 2        | Warm    | #A04000 |
| 3        | Neutral | #7B7D7D |
| 4        | Dark    | #2E4053 |

## 4. Bold

High contrast, vibrant. Pops on slides and social media. Good when you need maximum visual distinction between categories.

| Priority | Role    | Hex     |
|----------|---------|---------|
| 1        | Primary | #56B4E9 |
| 2        | Accent  | #AA3377 |
| 3        | Green   | #228833 |
| 4        | Yellow  | #CCBB44 |

## 5. Soft

Muted and approachable. Great for blog posts, portfolio pieces, and contexts where the chart should feel inviting rather than authoritative.

| Priority | Role    | Hex     |
|----------|---------|---------|
| 1        | Primary | #A8D8EA |
| 2        | Warm    | #E8C170 |
| 3        | Rose    | #C97B84 |
| 4        | Purple  | #9B8EC1 |

## Custom Palette ("Other")

If the user selects "Other" or describes a custom color scheme:
- Ask them to describe the vibe, or provide specific hex codes
- Ensure the custom colors have sufficient perceptual contrast from each other
- Warn the user if the chosen colors may not be colorblind-safe, and suggest adjustments

## Usage Notes

- **Highlighting**: When the user wants to highlight specific data points, use color 1 from the chosen palette as the accent and render all other data in a light gray (#D0D0D0). This creates a clear "signal vs. context" visual.
- **Sequential data**: For ordered/continuous data (heatmaps, gradients), generate a sequential colormap from the palette's primary color (light to dark).
- **Diverging data**: For data with a meaningful midpoint, use color 1 and color 2 from the palette as the two poles with white/light gray as the midpoint.
