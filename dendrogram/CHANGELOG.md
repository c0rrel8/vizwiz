# Dendrogram changelog

## v0.1.0 — 2026-08-11

Initial dendrogram visual. Raw Vega (tree/cluster are hierarchy layouts
with no Vega-Lite equivalent).

- 3-level hierarchy with same field-mapping adapter and 3-way exclusion
  cascade as sunburst and circle packing.
- Layout method toggle: `cluster` (all leaves at same depth — classic
  dendrogram) vs `tidy` (Reingold-Tilford spacing).
- Orientation toggle: `vertical` (root top, leaves bottom), `horizontal`
  (root left, leaves right), `radial` (root center, leaves on perimeter).
- Branch style toggle: `stepped` (right-angle elbows), `angled` (straight
  diagonal lines), `spline` (smooth cubic curves).
- Two optional metrics: Measure A scales node circle size, Measure B scales
  branch stroke width. When absent, the visual is purely structural.
- Node mark visibility toggle: `all` (circles at every node), `leaves`
  (Level 3 only), `none` (branches and labels only).
- Per-level fill and stroke color overrides (same isValid fallback pattern).
- Click-to-select with ancestor/descendant highlighting on nodes and links.
- Cross-highlighting via Measure A__highlight.
- Tooltips on both nodes and links.
- Per-level label show/hide, with orientation-aware positioning (radial
  labels rotated tangentially with readability flip).
- Design system defaults: dark canvas #182231, Segoe UI, pt-to-px font
  conversion.
