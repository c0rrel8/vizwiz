# Changelog — Composition Chart

## 0.1.0 — 2026-08-12

First version. A single parameterized Vega-Lite spec supporting four
composition chart arrangements: column-stacked, column-clustered,
bar-stacked, bar-clustered. Controlled by `chartMode` and `groupMode`
params — all four share the same transform pipeline, data labels, legend,
and axis styling.

- **Field-mapping adapter** with the exclusion cascade pattern for two
  field parameters (`MappedParameter` → Category, `MappedParameterTwo` →
  SubCategory) over the same 4 candidate columns, matching the waffle
  chart's established approach.
- **Per-subcategory default colors** via a `dense_rank` window transform
  sorted alphabetically by SubCategory, indexing into a 6-color palette
  (`#4555D6`, `#8C97E8`, `#2D3B68`, `#6B7CFF`, `#B3BBF0`, `#3A4A90`).
  `ColorOverrideFill` takes precedence when bound.
- **Single legend** — only the column-stacked sublayer carries the fill
  encoding with a scale and legend config; the other 3 sublayers use
  `scale: null` to apply `FillColor` directly, avoiding a Vega-Lite bug
  where nested layers with data-driven `range: { field }` scales produce
  duplicate legends that can't be merged via `resolve`.
- **Selection interaction** (`barSelect`, point selection on Category) —
  defined on the column-stacked sublayer, referenced via `fillOpacity`
  conditions on all bar sublayers for click-to-highlight.
- **Data labels** — optional per-bar or per-stack-total labels, with a
  percent-of-grand-total toggle (`showLabelPercent`).
- Full design system compliance: dark canvas (`#182231`), Segoe UI,
  title/subtitle tier colors, font size in points with px conversion,
  stroke naming convention, params-only (no Config tab).
- 3 DAX templates: `categoryfieldparameter.dax` (documents
  `MappedParameter`), `categorytwofieldparameter.dax` (documents
  `MappedParameterTwo`), `compositionfixedcolorfill.dax` (`Composition
  Fixed Color Fill` measure — per-column-name color via SWITCH on MAX).

Verified locally: Vega-Lite compile and SVG render of all 4 mode
combinations against sample data. Not yet confirmed live in Power BI
Desktop.
