# Network Diagram changelog

## v0.1.1 — 2026-08-12

Fix: network diagram renders nothing in Deneb.

- Remove experimental drag interaction code — mark `on` triggers used
  `modify: "dragNode"` (a signal) instead of a dataset name, causing Vega
  to silently fail on the nodeMark. Drag can be re-added later with
  correct dataset-level mutation.
- Make force transform always static (300 iterations) — remove restart/
  static signal dependencies that relied on drag code.
- Fix window transform: add `frame: [null, null]` so `count` op returns
  total node count, not a running count (affected circular/grid layout
  positioning).

## v0.1.0 — 2026-08-12

Initial network diagram visual. Raw Vega (force transform is a physics
simulation with no Vega-Lite equivalent).

- Dual data mode: hierarchy (reuses 3-level field parameter model with
  same exclusion cascade) and edge-list (Source/Target columns).
- Four layout algorithms: force-directed, circular, grid, hierarchical
  (hierarchy mode only — falls back to force in edge-list mode).
- Force presets (compact/balanced/spread) with individual parameter
  overrides for link distance, repulsion strength, and link strength.
- Directed/undirected toggle with arrowheads (angle follows edge tangent,
  including on curved edges).
- Draggable nodes (experimental) — force simulation re-settles around
  pinned nodes during drag.
- Node sizing: three modes — fixed radius, metric-scaled (Measure A),
  degree-scaled (connection count).
- Edge weight: three independent toggle channels — thickness, opacity,
  sequential color — each driven by Measure B.
- Edge style toggle: straight lines or curved (quadratic bezier). Multi-
  edges between the same node pair auto-curve with perpendicular offset
  regardless of style setting.
- Labels: three modes — show (with min-degree threshold), hide, hover
  (shows hovered node + direct neighbors, overrides threshold).
- Click-to-select on nodes with neighbor highlighting (dimming on both
  nodes and edges).
- Cross-highlighting via Measure A__highlight.
- Per-level fill and stroke color overrides (hierarchy mode) with same
  isValid fallback pattern.
- Tooltips on both nodes and edges.
- Collision force prevents node overlap.
- Design system defaults: dark canvas #182231, Segoe UI, pt-to-px font
  conversion.
