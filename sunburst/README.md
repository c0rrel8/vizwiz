# Sunburst

A 3-level hierarchy (Level 1 > Level 2 > Level 3) rendered as concentric
arc rings — the classic sunburst / radial partition diagram. Each level
occupies its own ring; the angular sweep of each arc is proportional to
that node's metric value relative to its siblings. A single metric sizes
arcs; per-level fill and stroke colors are independently overridable from
DAX.

**Raw Vega** (`.vg.json`, not Vega-Lite) — the `partition` transform is a
hierarchy layout with no Vega-Lite equivalent. Same reasoning as circle
packing; see `docs/patterns.md`'s "When to use raw Vega instead of
Vega-Lite" for the library-wide rationale. Shares this library's color/
typography conventions (`docs/design-system.md`).

## Fields expected

| Spec field name | Role | Type | Notes |
|---|---|---|---|
| `Level1`/`Level2`/`Level3` | The 3 hierarchy levels, outermost ring is Level 3 | Text columns | Each is its own field-parameter-driven role — see "Field mapping" |
| `Metric` | Arc angular sweep | Measure | Sizes every arc at every level proportionally within its parent |
| `Level1ColorOverrideFill` / `Level2ColorOverrideFill` / `Level3ColorOverrideFill` *(optional)* | Per-category fill color, one per level | Text columns (hex) | Falls back to `level{N}DefaultFillColor` param |
| `Level1ColorOverrideBorder` / `Level2ColorOverrideBorder` / `Level3ColorOverrideBorder` *(optional)* | Per-category stroke color, one per level | Text columns (hex) | Falls back to `level{N}DefaultStrokeColor` param |

## Field mapping

Identical 3-way exclusion cascade to circle packing — same 4 candidate
columns (`EngagementCodeList`/`AlphaCode`/`CustodianDisplayName`/
`EFileName`), same `Level1Source`/`Level2Source` tracking, same `'All'`
fallback for unbound levels. See `circlepacking/README.md`'s "Field
mapping" section for the full mechanism.

## Hierarchy construction

Same pattern as circle packing: three independent `aggregate` datasets
(`level1Nodes`, `level2Nodes`, `leafNodes`), a synthetic `rootNode`, all
unioned into `nodes`, then `stratify` + `partition`.

**Key difference from circle packing**: the `partition` transform sizes
arcs by angular proportion, so non-leaf nodes must contribute
`LayoutValue: 0` to avoid double-counting (a non-leaf node's `Value` is
already the sum of its descendants' metrics; if partition read that value
AND summed descendants, each non-leaf's angular sweep would be inflated).
Only leaf nodes (Level 3, `LevelIndex === 3`) pass their `Value` through
as `LayoutValue`; partition computes non-leaf values bottom-up from leaf
values. The original `Value` field is preserved on all nodes for tooltips
and `PercentOfParent`.

This double-counting issue doesn't affect circle packing because `pack`
sizes parent circles to enclose children geometrically — the parent's own
value is irrelevant to layout.

## Layout

- **`innerRadius`** (`60`) — hollow center radius in pixels; set to `0`
  for a filled center (Level 1 arcs start from the origin).
- **`outerRadius`** (derived: `min(width, height) / 2 - 5`) — outer edge
  of the Level 3 ring.
- **`ringWidth`** (derived: `(outerRadius - innerRadius) / 3`) —
  equal-width bands, one per level. Ring radii are computed from `depth`,
  not from partition's own `r0`/`r1` output — partition's radius
  allocation is ignored in favor of uniform rings.
- **`arcPadAngle`** (`0.01` radians ≈ 0.6°) — angular gap between
  sibling arcs within the same ring.
- **`arcCornerRadius`** (`0`) — rounded arc corners; increase for a
  softer look.
- **`strokeWidth`** (`1`) — arc stroke thickness, shared across all
  levels. Stroke color defaults to background (`#182231`), creating clean
  radial separation between rings.

## Color

Same per-level fill/stroke override pattern as circle packing:

- Per-level fill defaults form a depth progression: `level1DefaultFillColor`
  (`#2D3B68`, darkest), `level2DefaultFillColor` (`#4555D6`),
  `level3DefaultFillColor` (`#8C97E8`, lightest).
- Per-level stroke defaults all match the background (`#182231`).
- `FillColor`/`StrokeColor` resolve via `isValid(datum.FillOverride) ?
  override : default`, same as circle packing.

## Cross-filtering and cross-highlighting

Same mechanisms as circle packing — see that visual's README for the full
background. Cross-highlighting in works via `MetricHighlight` /
`IsHighlighted`. Click-to-filter out is presumed to hit the same
Deneb-level limitation (not independently verified for this spec).

## Interactivity

**Click-to-select** (`sunburstSelect`): clicking an arc highlights that
arc, its full descendant subtree, and its ancestor chain — dims everything
else. Same string-prefix test as circle packing's `packSelect`. Clicking
the same arc again clears the selection.

**Tooltip**: same per-node format as circle packing — Level, Category,
Value, % of parent.

## Labels

Tangential text centered at each arc's midpoint (mid-angle, mid-radius).
Per-level configuration:

| Signal | Default | Purpose |
|---|---|---|
| `level{N}ShowLabel` | `true`/`true`/`false` | Per-level label visibility |
| `level{N}LabelMinAngle` | `0.15`/`0.1`/`0.08` radians | Minimum arc angular span to show a label |

Labels use the standard 180° flip for readability: text on the left half
of the sunburst (mid-angle between π/2 and 3π/2) is rotated 180° so it
always reads left-to-right, same technique as circle packing's radial
labels.

Level 1 labels use the title tier color (`#FCFCFD`); Level 2/3 use the
subtitle tier (`#A5B4CB`). All levels share `labelFontFamily` and
`labelFontSize`.

Label text marks are `interactive: false` so clicks pass through to the
arc beneath — same fix as circle packing's labels.

## Params reference

| Signal | Default | Purpose |
|---|---|---|
| `innerRadius` | `60` | Hollow center radius (px) |
| `outerRadius` | derived | Outer edge of Level 3 ring |
| `strokeWidth` | `1` | Arc stroke thickness (px), shared across all levels |
| `arcCornerRadius` | `0` | Rounded corner radius for arcs |
| `arcPadAngle` | `0.01` | Angular gap between sibling arcs (radians) |
| `level{N}DefaultFillColor` | `#2D3B68`/`#4555D6`/`#8C97E8` | Fallback fill per level |
| `level{N}DefaultStrokeColor` | `#182231` (all three) | Fallback stroke per level |
| `level{N}ShowLabel` | `true`/`true`/`false` | Per-level label visibility |
| `level{N}LabelMinAngle` | `0.15`/`0.1`/`0.08` | Min arc span to show label (radians) |
| `labelFontFamily` | `Segoe UI` | Shared label font |
| `labelFontSize` | `11` | Shared label font size (points) |
| `labelColor` | `#FCFCFD` | Level 1 label color (title tier) |
| `labelSubtitleColor` | `#A5B4CB` | Level 2/3 label color (subtitle tier) |

Derived signals (`labelFontSizePx`, `centerX`, `centerY`, `ringWidth`,
`outerRadius`) are computed from the ones above — don't hand-edit.

## Setup in Deneb

1. In Deneb's **Settings** tab, switch **Provider** to **Vega**.
2. Add a Deneb visual, drop `Level1`, `Level2`, `Level3`, and `Metric`
   into the Values well (color override fields as needed).
3. Paste `sunburst.vg.json` into the **Specification** tab.
4. Tune the `signals` block.

## Known limitations

- Click-to-filter out is presumed broken (same Deneb limitation as circle
  packing / waffle chart — not independently re-verified).
- No small-multiples split.
- The `|` id-separator assumption is untested against real category values.
- Nothing here has been verified against a live Power BI report.
- Per-level label font/size isn't independently configurable.
