# Sunburst changelog

## v0.1.0 — 2026-08-11

Initial sunburst visual. 3-level hierarchy rendered as concentric arc
rings via Vega's `partition` transform.

- 3-way exclusion cascade for field mapping (same as circle packing)
- Per-level fill/stroke color overrides with DAX-driven `isValid()` fallback
- `LayoutValue` field on non-leaf nodes set to 0 to prevent partition's
  `sum()` from double-counting angular proportions (see README's
  "Hierarchy construction" for why this matters for partition but not pack)
- Click-to-select with subtree/ancestor highlighting (same string-prefix
  test as circle packing)
- Cross-highlighting in via `MetricHighlight` / `IsHighlighted`
- Tangential labels with 180° readability flip, per-level visibility and
  min-angle thresholds
- `innerRadius` signal for hollow center; `arcPadAngle` and
  `arcCornerRadius` for arc styling
