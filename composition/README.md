# Composition Chart

A parameterized bar/column chart with stacked and clustered group modes.
Vega-Lite engine. Set `chartMode` (`'column'` / `'bar'`) and `groupMode`
(`'stacked'` / `'clustered'`) params to switch between four arrangements
from a single spec.

Shares color/typography/architecture conventions with the rest of this
library — see `designsystem.md` and `patterns.md` at the repo root.

**Provider: Vega-Lite** — set the Deneb visual's provider to Vega-Lite
before pasting.

## Fields expected

| Spec field name | Role | Type | Notes |
|---|---|---|---|
| `Category` | Primary grouping (x-axis for column, y-axis for bar) | Text column | Field-parameter-driven via `MappedParameter` |
| `SubCategory` | Color/stack grouping | Text column | Field-parameter-driven via `MappedParameterTwo`; uses exclusion cascade to avoid colliding with Category |
| `Measure A` | Bar length/height | Measure | Renamed to `MetricA` internally |
| `ColorOverrideFill` *(optional)* | Per-subcategory fill color | Text column (hex) | Falls back to a 6-color palette indexed by subcategory rank |

## Field mapping

Both `Category` and `SubCategory` are field-parameter-driven over the
same 4 candidate columns (`EngagementCodeList`, `AlphaCode`,
`CustodianDisplayName`, `EFileName`). The spec uses the exclusion cascade
pattern (documented in `patterns.md`): `Category` resolves first via a
standard `isValid()` cascade, recording which candidate it claimed in
`CategorySource`; `SubCategory`'s cascade then skips whichever candidate
`CategorySource` already claimed, falling back to `Category` itself when
no second candidate is bound.

## Params

### Mode

| Param | Default | Values |
|---|---|---|
| `chartMode` | `"column"` | `"column"`, `"bar"` |
| `groupMode` | `"stacked"` | `"stacked"`, `"clustered"` |

### Bar styling

| Param | Default | Description |
|---|---|---|
| `barCornerRadius` | `2` | Corner radius (clustered mode only) |
| `barStrokeColor` | `"#182231"` | Bar border / axis domain color |
| `barStrokeWidth` | `0.5` | Bar border width |
| `fillColorDefault` | `"#4555D6"` | Legacy — palette override now uses SubCategoryRank |
| `barPadding` | `0.2` | Padding between category groups |
| `clusterPadding` | `0.1` | Padding between bars within a cluster |

### Data labels

| Param | Default | Description |
|---|---|---|
| `showDataLabels` | `false` | Toggle data labels |
| `dataLabelFontSize` | `10` | Font size in points |
| `dataLabelColor` | `"#FCFCFD"` | Label text color |
| `dataLabelFormat` | `"~s"` | d3-format string for values |
| `showLabelPercent` | `false` | Show percentage instead of value |

### Axes

| Param | Default | Description |
|---|---|---|
| `axisFontFamily` | `"Segoe UI"` | Font for all axis text |
| `axisLabelColor` | `"#A5B4CB"` | Axis label color |
| `axisLabelFontSize` | `10` | Axis label size in points |
| `axisTitleColor` | `"#A5B4CB"` | Axis title color |
| `axisTitleFontSize` | `11` | Axis title size in points |
| `categoryAxisTitle` | `""` | Category axis title text |
| `valueAxisTitle` | `""` | Value axis title text |
| `valueFormat` | `"~s"` | d3-format string for value axis |
| `showGrid` | `true` | Toggle grid lines |
| `gridColor` | `"#2D3B68"` | Grid line color |
| `gridOpacity` | `0.5` | Grid line opacity |

### Legend

| Param | Default | Description |
|---|---|---|
| `showLegend` | `true` | Toggle legend |
| `legendOrient` | `"right"` | Legend position |
| `legendLabelColor` | `"#A5B4CB"` | Legend label color |
| `legendTitleColor` | `"#A5B4CB"` | Legend title color |
| `legendFontSize` | `10` | Legend font size in points |
| `legendTitle` | `""` | Legend title text |

## Color

Default fill uses a 6-color palette indexed by subcategory rank
(alphabetical order): `#4555D6`, `#8C97E8`, `#2D3B68`, `#6B7CFF`,
`#B3BBF0`, `#3A4A90`. When `ColorOverrideFill` is bound, those values
take precedence.

The legend is driven by the column-stacked sublayer's fill encoding
(the primary sublayer). The other 3 sublayers use `scale: null` to pipe
`FillColor` directly, avoiding Vega-Lite's 4-legend duplication bug with
nested layers sharing a data-driven `range: { field }` scale.

## Architecture

The spec uses a nested layer structure with filter-based mode switching:

- **Top level**: 2 layer groups filtered by `chartMode` (column / bar)
- **Each group**: 4 sublayers filtered by `groupMode` + `showDataLabels`
  - Stacked bars (or horizontal bars)
  - Clustered bars (with xOffset / yOffset for grouping)
  - Stacked data labels (aggregate totals per category)
  - Clustered data labels (per-bar values)

`resolve: { scale: { x: "independent", y: "independent" } }` lets the
column group and bar group define their own axis orientations without
conflict.

Selection (`barSelect`) is defined on the column-stacked sublayer only,
referenced via `fillOpacity` conditions on all bar sublayers.

## DAX templates

| File | Measure name | Purpose |
|---|---|---|
| `categoryfieldparameter.dax` | — | Documents `MappedParameter` (Level 1 / Category) |
| `categorytwofieldparameter.dax` | — | Documents `MappedParameterTwo` (Level 2 / SubCategory) |
| `compositionfixedcolorfill.dax` | `Composition Fixed Color Fill` | Per-column-name color, bound to `ColorOverrideFill` |

## Future work

- Radial composition mode (pie/donut) — additional `chartMode` value
- Lollipop mark variant — circle + line instead of solid bar
- Per-value color gradient (rank-based, like waffle's
  `category-color-gradient-rank.dax`)
