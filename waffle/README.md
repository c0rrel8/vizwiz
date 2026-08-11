# Waffle Chart

A grid of shapes (default: circles) per category, where each category's
share of the grid is proportional to its `MetricA` value — the classic
"percentage waffle" / pictogram-array chart. Optionally splits into one
small-multiple grid per `SeriesCategory` (a second, independent grouping
dimension) — leave that role unbound for a single grid.

Shares its color/typography/architecture conventions with
`visuals/aster-plot/` — see the repo root's `docs/design-system.md` and
`docs/patterns.md` before changing anything here; this README only covers
what's specific to the waffle chart.

## Fields expected

| Spec field name | Role | Type | Notes |
|---|---|---|---|
| `Category` | Grid grouping / legend | Text column | One color/group per distinct value — "Mapped Parameter B" from the original design conversation |
| `SeriesCategory` *(optional)* | Small-multiple split | Text column | When bound, one grid is drawn per distinct value (small multiples); when left unbound, everything renders as a single grid — "Mapped Parameter" |
| `MetricA` | Relative element count | Measure | Each category's share of its grid = `MetricA / sum(MetricA)` *within that category's own `SeriesCategory` group* |
| `ColorOverride` | Per-category element fill color | Text column (hex, e.g. `#01B8AA`) | See "Color" below |
| `BorderColorOverride` *(optional)* | Per-category element border color | Text column (hex) | Falls back to `elementBorderColor` when not bound — see "Color" below |
| `ShapeOverride` *(optional)* | Per-category element shape | Text column (SVG path string, or one of Vega's built-in shape names: `circle`, `square`, `cross`, `diamond`, `triangle-up`, etc.) | Falls back to `defaultShape` (`"circle"`) when not bound |

`Category` and `MetricA` are required; everything else is optional and
degrades gracefully (see "Color" and "Field mapping").

**This report's concrete bindings** (same underlying model as the aster
plot, shared across this library for now): **both `Category` and
`SeriesCategoryRaw` are field-parameter-driven**, unlike the earlier
assumption in this README — `Category` comes from `MappedParameterTwo`
(bound to the visual's "Category2" role) and `SeriesCategoryRaw` comes from
the aster plot's own `MappedParameter` (bound to the "Category" role),
**both over the exact same 4 candidate columns**
(`EngagementCodeList`/`AlphaCode`/`CustodianDisplayName`/`EFileName`, see
`visuals/aster-plot/dax/category-field-parameter.dax` and
[`dax/category-two-field-parameter.dax`](dax/category-two-field-parameter.dax)).

| Spec field name | Actual Power BI field parameter | Candidate columns |
|---|---|---|
| `SeriesCategoryRaw` | `MappedParameter` (bound to the "Category" role) | `EngagementCodeList`/`AlphaCode`/`CustodianDisplayName`/`EFileName` |
| `Category` | `MappedParameterTwo` (bound to the "Category2" role) | same 4 candidates |
| `MetricA` | `Measure A` | — (plain rename) |
| `ColorOverride` | `ColorOverrideFill` | — (plain rename) |
| `BorderColorOverride` | `ColorOverrideBorder` | — (plain rename) |

**This creates a real problem the aster plot never had to solve**: since
both field parameters draw from the *same* 4 candidate columns, and a
field parameter's resolved key can't be renamed per-visual (confirmed
during the aster plot's own build), the two roles can end up needing to
read from overlapping keys in the *same* row. Confirmed live (via a
"view as table" screenshot with `MappedParameter` sliced to `AlphaCode` and
`MappedParameterTwo` sliced to `EngagementCodeList`): Deneb *does* expose
both as separate, simultaneous keys in that case — no forced collision —
but a naive `isValid()` cascade checking the same 4 candidates in the same
order for both roles still resolved **both** `Category` and
`SeriesCategoryRaw` to the same value, because whichever candidate is
checked first and happens to be present wins for both roles regardless of
which field parameter it actually came from. Verified by reproducing it
locally before fixing it — see "Field mapping" below for the actual fix.

`ShapeOverride` has no confirmed binding yet — leave it unbound (falls back
to `defaultShape`) until there's a measure for it.

## Field mapping

The spec's `transform` array opens with the field-mapping adapter — 8
`calculate` steps this time (one more than the usual pattern in
`docs/patterns.md`), because `Category` and `SeriesCategoryRaw` are each
field-parameter-driven *and* need to stay correctly attributed to their own
field parameter rather than both grabbing whichever candidate happens to
resolve first:

```json
{
  "calculate": "isValid(datum.EngagementCodeList) ? 'EngagementCodeList' : isValid(datum.AlphaCode) ? 'AlphaCode' : isValid(datum.CustodianDisplayName) ? 'CustodianDisplayName' : isValid(datum.EFileName) ? 'EFileName' : ''",
  "as": "CategorySource"
},
{
  "calculate": "isValid(datum.EngagementCodeList) ? datum.EngagementCodeList : isValid(datum.AlphaCode) ? datum.AlphaCode : isValid(datum.CustodianDisplayName) ? datum.CustodianDisplayName : isValid(datum.EFileName) ? datum.EFileName : '(Unspecified)'",
  "as": "Category"
},
{
  "calculate": "(isValid(datum.EngagementCodeList) && datum.CategorySource !== 'EngagementCodeList') ? datum.EngagementCodeList : (isValid(datum.AlphaCode) && datum.CategorySource !== 'AlphaCode') ? datum.AlphaCode : (isValid(datum.CustodianDisplayName) && datum.CategorySource !== 'CustodianDisplayName') ? datum.CustodianDisplayName : (isValid(datum.EFileName) && datum.CategorySource !== 'EFileName') ? datum.EFileName : null",
  "as": "SeriesCategoryRaw"
},
{ "calculate": "datum['Measure A']", "as": "MetricA" },
{ "calculate": "datum.ColorOverrideFill", "as": "ColorOverride" },
{ "calculate": "datum.ColorOverrideBorder", "as": "BorderColorOverride" },
{ "calculate": "datum.ShapeOverride", "as": "ShapeOverride" },
{ "calculate": "isValid(datum.SeriesCategoryRaw) ? datum.SeriesCategoryRaw : 'All'", "as": "SeriesCategory" }
```

**How this works**: `CategorySource` records which candidate column's
name (as a *string*) actually fed `Category` — computed with the exact
same priority-order cascade as `Category` itself, just returning the
column's name instead of its value. `SeriesCategoryRaw`'s cascade then
runs the *same* 4 candidates in the *same* priority order, but each branch
additionally requires that candidate's name **not** match
`CategorySource` — i.e. "give me the highest-priority candidate that
`Category` didn't already claim." This is why `SeriesCategoryRaw`'s
terminal fallback is the literal `null`, not `'(Unspecified)'` like
`Category`'s: `isValid('(Unspecified)')` is `true` (it's a normal string),
which would have silently broken the "unbound → single grid" degradation
(`SeriesCategory` falling back to `'All'`) — confirmed by testing
`isValid()` against a string vs. `null` directly. `Category` doesn't need
this exclusion treatment itself since it's always resolved first, with no
prior claim to avoid.

**Verified against three scenarios** (via a stripped-down copy of the
adapter compiled and run directly, not just reasoned about): both
field parameters bound to different columns (`Category`/`SeriesCategoryRaw`
correctly end up different — matches the live screenshot exactly),
`SeriesCategoryRaw`'s own field parameter completely unbound
(`SeriesCategory` correctly falls back to `'All'`, single-grid mode), and
both field parameters happening to be sliced to the identical column
(`SeriesCategoryRaw` safely falls back to `'All'` rather than duplicating
`Category`'s value under a false pretense of being a distinct series).

**Practical implication**: if `MappedParameter` and `MappedParameterTwo`
are ever both sliced to the *same* underlying column at the same time,
`SeriesCategory` will read as `'All'` for every row (as if the series role
were unbound) rather than showing that column's value twice. Point the two
slicers at different candidates to get real small multiples.

`MetricA`/`ColorOverride`/`BorderColorOverride`/`ShapeOverride` remain
plain renames — no field-parameter mechanics involved for those.

## Color

**Fill color** (`ColorOverride`) can be either a fixed/pinned color per
distinct `Category` value, or a magnitude-based gradient — two DAX
templates cover both:

- [`dax/category-fixed-color.dax`](dax/category-fixed-color.dax) — a
  `SWITCH`-based pinned color per category (same shape as the aster
  plot's "Pinned/fixed colors per category" example), for when the only
  job is telling categories apart, with no "least to greatest" comparison.
- [`dax/category-color-gradient.dax`](dax/category-color-gradient.dax)
  (measure named **`Category2 Color Gradient`** — deliberately *not*
  `Category Color Gradient`, since the aster plot already has a measure
  by that exact name in this same shared model, and measure names must
  be unique per model; see the file's own comment and this changelog's
  0.5.3 entry for the mix-up this avoids) — log-scale interpolated
  between `#A5C1FF` (lowest category-level total of `Measure A`) and
  `#4555D6` (highest), same structure as the aster plot's own gradient
  measure (`visuals/aster-plot/dax/measure-a-color-gradient.dax`):
  per-candidate `MINX`/`MAXX` over an `ADDCOLUMNS` summary table, `MAX`
  (not `SELECTEDVALUE`) for the field parameter's own composite-keyed
  column, log rather than linear scale because `Measure A` is heavily
  right-skewed here (one dominant category against many much smaller
  ones — confirmed live in this visual's own data). Verified the
  interpolation numerically against the real reported data before
  shipping: colors spread visibly across the whole range
  (`#A5C1FF` → `#9FBAFD` → `#8BA4F4` → ... → `#4555D6`) rather than
  clustering near one end, which a linear scale would have done with this
  data's skew.
- [`dax/category-color-gradient-rank.dax`](dax/category-color-gradient-rank.dax)
  (measure named **`Category2 Color Gradient Rank`**) — a third option:
  instead of interpolating by *magnitude* (even log-scaled), this ranks
  categories by `Measure A` and spaces colors **equally by rank position**
  — step size is a flat `1 / (member count − 1)`, regardless of how close
  or far apart the underlying values actually are. Direction is
  **reversed** from the other gradient measure: rank 1 (the *largest*
  `Measure A`) gets the *lightest* color (`#A5C1FF`); the smallest-ranked
  category gets the *darkest* (`#4555D6`) — confirmed intentional, not a
  copy-paste mistake. Ties (identical `Measure A`) break alphabetically by
  category name, via a manual "count how many rows are strictly better"
  rank rather than `RANKX` (which has no built-in secondary tie-break) —
  see the file's own comments for why this guarantees no two rows ever
  share a rank, sidestepping the usual "dense vs. skip" ties question
  entirely. Verified both the ranking and the tie-break numerically before
  shipping (a synthetic 3-row tie test: two categories tied at the same
  value resolved to adjacent ranks in alphabetical order, not the same
  rank), then rendered against the real 10-category data — evenly-stepped
  colors end to end, distinct from the other gradient measure's uneven,
  magnitude-driven spacing.

All three fall back to `elementDefaultColor` for any row where the bound
measure isn't populated. Only bind one to `ColorOverrideFill` at a time.

**Border color** (`BorderColorOverride`) is optional and rarely needed —
`elementBorderColor` already gives every element a clean, uniform border
by default (matching `docs/design-system.md`'s background/border
convention, `#182231`, so elements read as separated by a gap rather than
a drawn line). Only bind this if specific categories need a visually
distinct border (e.g. flagging an outlier) — see
[`dax/border-fixed-color.dax`](dax/border-fixed-color.dax).

**Shape** (`ShapeOverride`) defaults to `"circle"` (`defaultShape`), a
plain spec param, but can be driven per-category from a measure returning
either one of Vega's built-in shape names or a custom SVG path string —
both work interchangeably in the same field. This was a deliberate design
choice (see the design conversation): one mechanism (an optional field
with an `isValid()` fallback) handles both "just use the default circle"
and "give me a custom shape per category" without needing two separate
params.

**Legend** text uses the subtitle tier (`legendLabelColor`/
`legendTitleColor`, `#A5B4CB`) exactly like the aster plot — see that
visual's README ("Legend position and chart centering") for the
corner-vs-edge orient mechanism, which works identically here
(`showLegend`/`legendOrient`).

## Small multiples

`SeriesCategory` controls whether this renders as one grid or several:

- **Unbound** → every row resolves to the same constant `'All'` value via
  the field-mapping adapter, so there's exactly one implicit "series" and
  the layout math (below) naturally collapses to a single full-size grid
  with no panel title reserved.
- **Bound** → one grid ("panel") is drawn per distinct value, arranged in
  a wrapped grid-of-panels layout controlled by `seriesColumns` (how many
  panels per row) and `panelGutter` (pixel gap between panels). Each
  panel gets its own title (the `SeriesCategory` value) unless
  `showSeriesLabel` is `false`.

This isn't Vega-Lite's native `facet`/`repeat` operator — those restructure
the spec itself (a structural, spec-authoring-time choice), which can't be
toggled at runtime by simply binding/unbinding a field. Instead, every
panel's position (`PanelX`/`PanelY`/`PanelWidth`/`PanelHeight`) is computed
as an ordinary per-row `calculate` field, the same way the aster plot
computes per-wedge geometry — see "Implementation notes" for the mechanism
(`dense_rank` for panel indexing, manual grid-of-panels math).

**Labels are otherwise not used in this chart** (unlike the aster plot's
wedge/center labels) — the only text on-chart is the optional panel title.
Hover tooltips carry `SeriesCategory`/`Category`/`MetricA`/percent-of-total
instead.

## Setup in Deneb

Same three-tab structure as the aster plot (Specification/Config/Settings)
— see that visual's README for the full explanation; the short version:
paste `spec/waffle-chart.vl.json` into **Specification**, leave **Config**
alone (this spec has no top-level `config` key and controls everything via
its own `params`, per `docs/patterns.md`), and consider enabling
**Cross-Filtering (Selection) of Data Points** in **Settings** to test
whether it bridges the existing `categorySelect` click-to-highlight
interaction to Power BI's native cross-filtering (not yet verified live).

1. Add a Deneb visual, drop `Category` and `MetricA` into the Values well
   at minimum (`SeriesCategory`/`ColorOverride`/`BorderColorOverride`/
   `ShapeOverride` as needed — see "Fields expected").
2. Paste `spec/waffle-chart.vl.json` into the **Specification** tab.
3. Tune the `params` block (see reference table below) — grid size, panel
   layout, colors, fonts. All hand-edited directly in the JSON, same as
   the aster plot.

## Params reference

| Param | Default | Purpose |
|---|---|---|
| `gridRows` | `10` | Grid row count (fixed cell count, not auto-computed — `gridRows * gridCols` is the total number of elements the grid is divided into) |
| `gridCols` | `10` | Grid column count |
| `gridArea` | `1` | Fraction (`0`–`1`) of the available panel space the grid should occupy — `1` fills it edge-to-edge (after `panelGutter`/`panelTitleHeight`), anything less shrinks the grid and centers it within the panel, both horizontally and vertically. **Decimals need a leading zero** (`0.753`, not `.753`) — this is edited as raw JSON, and bare `.753` isn't valid JSON (confirmed: `JSON.parse('.753')` throws); there's no param-side restriction on precision beyond that. |
| `elementSizeRatio` | `0.8` | Element diameter as a fraction of its cell's available space (after `gridHorizPadding`/`gridVertPadding` are subtracted) |
| `gridHorizPadding` | `0` | Minimum horizontal gap between adjacent elements, in pixels |
| `gridVertPadding` | `0` | Minimum vertical gap between adjacent elements, in pixels |
| `defaultShape` | `"circle"` | Fallback element shape when `ShapeOverride` isn't bound/has no value — a Vega built-in shape name or an SVG path string |
| `useRowBreaks` | `false` | Insert a row break before each category's block within a grid, instead of letting categories flow continuously into the same row (see "Implementation notes") — a way to visually separate categories inside one grid without using color for set identity |
| `elementDefaultColor` | `#4555D6` | Fallback element fill color when `ColorOverride` isn't bound/has no value |
| `elementBorderColor` | `#182231` | Fallback element border color when `BorderColorOverride` isn't bound/has no value — matches the background, so elements read as separated by a gap, not a stroke |
| `elementBorderWidth` | `1` | Element border thickness, in pixels |
| `elementFillBorderGap` | `0` | Pixel gap between the element's fill and the *inside* edge of its border — `0` matches a normal single-outline element (the default look before this param existed); higher values pull the fill inward, leaving a visible ring of background color between fill and border |
| `seriesColumns` | `4` | *Maximum* panels per row — automatically reduced to the actual number of distinct `SeriesCategory` values when there are fewer than this (so a single grid always gets the full width instead of a quarter of it; see "Implementation notes") |
| `panelGutter` | `24` | Pixel gap between adjacent panels |
| `showSeriesLabel` | `true` | Show each panel's title (the `SeriesCategory` value) — only ever renders when there's more than one panel, regardless of this toggle |
| `panelTitleHeight` | `22` | Vertical space reserved at the top of a panel for its title, in pixels — only reserved when there's more than one panel |
| `panelTitleFontFamily` | `"Segoe UI"` | Panel title font family |
| `panelTitleFontSize` | `11` | Panel title font size, **in points** (see `docs/design-system.md`'s pt→px convention) |
| `panelTitleFontWeight` | `bold` | Panel title font weight |
| `panelTitleColor` | `#FCFCFD` | Panel title color (title tier) |
| `showLegend` | `true` | Show/hide the category legend |
| `legendOrient` | `"right"` | Legend position — same edge-vs-corner centering mechanism as the aster plot (see that visual's README) |
| `legendLabelColor` | `#A5B4CB` | Legend entry text color (subtitle tier) |
| `legendTitleColor` | `#A5B4CB` | Legend title text color (subtitle tier) |

A few other params in the same array (`panelTitleFontSizePx`,
`legendOrientEff`, `legendXEff`, `legendYEff`) are `expr`-computed from the
ones above — don't hand-edit those directly, per the same convention as
the aster plot.

## Implementation notes / gotchas

- **A single grid was rendering into roughly a quarter of the visual
  instead of the full width** — root cause: `RawPanelWidth` was computed
  as `width / seriesColumns` unconditionally, so even with only one panel
  (no `SeriesCategory` bound at all) it divided by the *configured*
  `seriesColumns` (default `4`) instead of the *actual* number of panels
  (`1`). Fixed with `EffectiveSeriesColumns = min(seriesColumns,
  TotalSeriesCount)`, used everywhere `seriesColumns` previously was —
  a single grid now gets `EffectiveSeriesColumns = 1` (the full width),
  and small multiples with fewer series than `seriesColumns` get wider
  panels too, instead of leaving unused columns of empty space. Confirmed
  by rendering a single 20×20 grid before and after — before, it occupied
  the left quarter of the frame; after, it fills the full square.
- **`gridArea` and centering** work by computing one **uniform** cell size
  (not independent width/height per axis) that fits `gridRows × gridCols`
  within `gridArea` fraction of the available panel space, preserving
  the grid's aspect ratio: `CellSize = min((GridAreaWidth * gridArea) /
  gridCols, (GridAreaHeight * gridArea) / gridRows)`. The resulting grid
  footprint (`CellSize * gridCols` by `CellSize * gridRows`) is then
  centered within the *full* available panel space (not just the target
  `gridArea` box) via `GridOriginX/Y = GridAreaX/Y + (GridAreaWidth/Height
  - GridFootprintWidth/Height) / 2`. `gridArea < 1` visibly shrinks and
  centers the grid; `gridArea = 1` (default) preserves the original
  edge-to-edge behavior. Confirmed via a render at `gridArea: 0.6`
  showing the grid shrunk and centered with even margins on all sides.

- **Vega-Lite has no "repeat this row N times" transform — `sequence()` +
  `flatten` fills that gap.** Each incoming row (one per `Category` within
  a `SeriesCategory`) is expanded into potentially dozens of per-cell rows
  via `{"calculate": "sequence(datum.CellStart, datum.CellEnd)", "as":
  "CellIndices"}` followed by `{"flatten": ["CellIndices"]}` — `sequence`
  generates an array of consecutive integers, and `flatten` explodes an
  array-valued field into one output row per element, carrying every other
  field along unchanged. Verified empirically (compiled + ran a minimal
  repro compiling 3 category rows down to exactly 100 flattened cell rows,
  correctly distributed) before building the rest of the spec around it.
  This is the one technique from this visual most likely to be useful
  elsewhere in the library — any future "N marks per row of data" need
  should reach for this first.
- **`joinaggregate` doesn't collapse rows — this silently multiplied a
  category's total once per underlying row instead of counting it once,
  badly distorting every category's cell count.** `CategoryTotal` was
  originally computed via `joinaggregate` (grouped by `SeriesCategory`/
  `Category`), which *adds* the aggregate to every existing row without
  reducing the row count. Fine as long as there's exactly one row per
  category — but `Category` here is fact-grain data (e.g. a customer/
  custodian name), which real data has multiple underlying rows for. With
  5 raw rows for one category (each carrying that category's already-
  correct total, as Power BI's own SUM would hand it to Deneb) and 1 row
  for another, the later cumulative-sum window (which sums `CategoryTotal`
  across *all* rows in sorted order, not deduplicated first) added the
  5-row category's total 5 times over — reproduced locally with exactly
  that shape (5 rows @ 840.368 + 1 row @ 204.46 → a 100-cell grid that
  should split 95/5 came out 477/4 with 481 total cells instead of 100).
  This read as "sorting isn't working" when reported live, but the sort
  itself was correct — the *magnitudes* feeding it were wrong. Fixed by
  replacing that `joinaggregate` with a real `aggregate` (which *does*
  collapse to one row per group) immediately after the field-mapping
  adapter, before anything cumulative runs — `ColorOverride`/
  `BorderColorOverride`/`ShapeOverride` are carried through via `"op":
  "min"` (picking any single value, since they're expected to already be
  constant per category). Re-verified the exact reproduction case
  afterward: correct 95/5 split, exactly 100 total cells.
- **`CellStart`/`CellEnd` (the per-category cell range fed into
  `sequence()`) use the exact same cumulative-window pattern as the aster
  plot's wedge angles** (`window` cumulative sum + `lag` for the start,
  grouped by `SeriesCategory` this time instead of ungrouped) — just
  producing integer cell indices instead of continuous angle degrees. This
  is also why rounding "just works": the last category in each series
  always lands on `CellEnd = gridRows * gridCols` exactly, since its
  cumulative share reaches exactly 1.0 — any rounding slack gets absorbed
  by earlier categories rather than causing the grid to over/under-fill.
- **Descending fill order is one `"sort"` clause on that same cumulative
  window transform** — `"sort": [{"field": "CategoryTotal", "order":
  "descending"}]`, scoped per `groupby: ["SeriesCategory"]` like the rest
  of the window. Vega-Lite's `window` transform orders rows within each
  partition before running the window op, so this reorders *which category
  gets which cell range* (largest `CategoryTotal` gets cells `0..N` first)
  without touching any of the cell-index math itself.
- **A `"sort"` on one `window` transform does not carry over to a later,
  separate `window` transform — each one needs its own, and a mismatch
  between them silently corrupts cell ranges and drops categories
  entirely, not just misorders them.** The cumulative `sum` (producing
  `CellEnd`) had `"sort": [{"field": "CategoryTotal", "order":
  "descending"}]`, but the very next `window` step — a `lag` on `CellEnd`
  to produce `CellStart` — had none, so it fell back to incoming
  (alphabetical) row order. Each category's `CellStart` ended up being
  "the previous row *alphabetically*'s `CellEnd`" instead of "the previous
  row *by size*'s `CellEnd`," producing wrong, sometimes **negative-width**
  ranges (`CellEnd < CellStart`) for categories whose alphabetical and
  size-sorted neighbors differed enough. `sequence(start, stop)` with
  `start > stop` returns an **empty array**, and `flatten` on an empty
  array produces **zero rows** — so those categories didn't just render
  in the wrong place, they vanished completely (from the grid *and* the
  legend, since the legend's domain comes from whatever `Category` values
  actually reach the mark). Reported as "sort isn't working" and "is it
  not rendering the full set of members" — both were the same bug,
  traced by dumping the pre-`flatten` rows directly and finding `CellEnd
  < CellStart` on exactly the categories missing from the legend. Fixed
  by adding the identical `"sort"` to the `lag` window transform too.
  **The general lesson: every `window` transform in a chain that depends
  on a consistent row order needs its own explicit, identical `"sort"` —
  it is not inherited from a previous step, and Vega-Lite compiles this
  without any warning either way.** Confirmed via a render with 10
  distinctly-colored categories of very different magnitudes — the
  largest one's color fills the grid starting from the top-left, in
  strictly decreasing order after that, all 10 present, independently
  within every panel in a small-multiples layout.
- **Panel indexing uses a `dense_rank` window op with no `groupby` and a
  sort on `SeriesCategory`** — this ranks each row's `SeriesCategory` among
  all distinct values in sorted order (confirmed via an isolated compile
  test: `X`/`Y`/`X`/`Z` sorted and dense-ranked gave `1`/`2`/`1`/`3`,
  exactly the "which distinct series is this" index needed), which
  combined with `seriesColumns` gives each row's panel row/column via plain
  arithmetic (`% ` and `floor`/`ceil`) — no native faceting involved (see
  "Small multiples" above for why).
- **The color legend swatches came out as a smooth, nearly-invisible fade
  during development** — traced to the *sample data*, not a spec bug: two
  of the placeholder `ColorOverride` hex values chosen for testing
  (`#263449`, `#1E2B3A`) were nearly identical to the new dark background
  (`#182231`), so those categories' fill and legend swatches blended into
  the canvas. Fixed by picking sample colors from Power BI's actual default
  10-color theme palette (already used elsewhere in this repo's
  `tools/deneb-shims.mjs`), which stays visibly distinct against the dark
  background. Worth remembering when picking *real* category colors too —
  verify they're not too close to `#182231`.
- The point mark's `size` channel is the element's bounding **area** in
  px², not its diameter — `ElementSize = PI * ElementDiameter² / 4` derives
  the area from a target diameter (the standard circle-area formula),
  which is what actually makes the shape "automagically scale" to fit its
  cell as grid dimensions/container size change.
- **`elementFillBorderGap` needs two separate marks, not one** — a single
  Vega-Lite point mark with both `fill` and `stroke` set draws the stroke
  *centered* on the shape's own boundary (half overlapping the fill, half
  outside it), with no way to open a gap between them on one mark. Fixed
  by splitting the element into two layers sharing the same `x`/`y`/
  `shape`: an outer `filled: false` (stroke-only) mark at the full
  `ElementSize`, and an inner `filled: true` (fill-only) mark at a smaller
  `FillSize`, where `FillDiameter = ElementDiameter - elementBorderWidth -
  2 * elementFillBorderGap` (subtracting the border's own half-width on
  each side, then the gap on each side, clamped to `0` so an oversized gap
  just hides the fill rather than going negative). At `elementFillBorderGap:
  0` this reduces to exactly the same edge as the old single-mark version —
  confirmed by checking the math resolves identically, not just visually.
- **The click-to-highlight selection (`categorySelect`) is declared on the
  border layer but also drives the fill layer's opacity** — Vega-Lite
  selection params declared in one layer are readable from sibling layers'
  encodings within the same view; confirmed by simulating a selection
  directly (bypassing pixel-coordinate clicking, which is finicky to get
  exactly right in a script) and checking the compiled SVG: both layers'
  matching elements dropped to `fill-opacity`/`stroke-opacity` `0.25`
  together, non-matching elements stayed at `1` on both.
- **`useRowBreaks` exists because color can't be used as a channel for set
  identity in this report** — with a fixed brand palette, the color-gradient
  measures (see "Color" above) compress too smoothly to visually separate
  categories on their own. Turning this on inserts a row break before each
  category's block instead, so category boundaries are visible as gaps in
  the grid shape itself, independent of color.
  - **The break preserves column position — it does not reset to column
    0.** A category's last cell lands at some `(row, col)`; if that isn't
    the last column, the *next* category starts at `(row + 1, col + 1)` —
    the same column its cells would have continued into on the current
    row, just pushed one row down. This is what produces the staircase/
    cascading-indent look rather than a ragged left-aligned block per
    category. If a category's block happens to end exactly on the last
    column, no break is needed — the next category's natural continuation
    already starts a fresh row at column 0.
  - **Implementation is a non-recursive cumulative sum, not a per-category
    row offset.** The first attempt computed each category's row shift as
    a flat `+1` per category rank (`0, 1, 2, ...`), on the theory that each
    category simply pushes everything after it down by one row. That's
    wrong whenever any category's own cell count spans more than one grid
    row on its own: the flat offset doesn't know how many *extra* rows a
    large category actually consumed, so later categories' blocks
    overlapped earlier ones' cells outright (caught by an automated
    overlap check — dumping `(CellRow, CellCol)` pairs per row and
    asserting no two categories in the same panel ever claim the same
    pair — before this ever reached a screenshot). The fix: each
    category's own `RowsSpanned = ceil((CellStart % gridCols + (CellEnd -
    CellStart)) / gridCols)` is computed first (how many rows *this*
    category's block actually occupies, given where it starts), and the
    row each category *starts* on is the running total of every prior
    category's `RowsSpanned` in the same panel — an exclusive prefix sum,
    grouped by `SeriesCategory` and sorted the same as `CellStart`/
    `CellEnd`. This guarantees no two categories' cells can ever land on
    the same `(row, col)`, regardless of how many rows any one category
    spans.
  - **The exclusive prefix sum is a `lag` of an inclusive cumulative
    `sum`, not a `window` with `frame: [null, -1]`.** `frame: [null, -1]`
    looks like the natural way to ask for "everything before the current
    row" directly, but empirically (confirmed with a minimal isolated
    Vega-Lite compile, independent of this spec, sorting on a tie-free
    field) it did not produce an exclusive prefix sum in this Vega-Lite
    version — it consistently returned each row's *inclusive* cumulative
    sum from one row further into the partition than expected. The
    existing, already-proven pattern in this same spec (`CellEnd` = an
    inclusive cumulative `sum` with `frame: [null, 0]`, then `CellStart` =
    a `lag` of `CellEnd`, both with their own identical `"sort"` per the
    lesson above) sidesteps the question entirely and was reused verbatim
    for `RowsSpannedCumulative` → `StaircaseRowStart`.
  - **Row capacity is a fixed safety margin, not an exact computed
    minimum**: `EffectiveGridRows = useRowBreaks ? gridRows +
    CategoryCountInSeries : gridRows`, used in place of the bare `gridRows`
    param when sizing cells and the grid's footprint height. This is
    deliberately generous rather than exact — the real number of rows a
    given series ends up using depends on how the categories' cell counts
    happen to align to `gridCols` (see `RowsSpanned` above), and can be
    less than `gridRows + CategoryCountInSeries` in practice. Verified
    against a 2000-trial randomized simulation (varying category counts,
    `gridCols`, `gridRows`, and cell-count distributions) that actual rows
    used never exceeds this bound. There is likely a *practical* lower
    limit on how low `gridRows` can usefully be set relative to how many
    categories a series has (too low and cells shrink a lot to fit the
    padded row count) — this isn't validated or clamped automatically; it's
    on the report author to keep `gridRows` reasonable for the category
    counts actually in play, same as any other manually-tuned grid param.

## Known limitations / roadmap

- **Fill order is row-major top-to-bottom** — cell 0 is the top-left of
  each grid, filling left-to-right then wrapping down, one category block
  at a time. **Categories are always ordered by `MetricA` descending
  within their own grid/panel** (largest category first), not incoming
  data order — see "Implementation notes" for how. No alternate
  fill-direction (e.g. bottom-up, common in some pictogram charts) or
  alternate sort (alphabetical, ascending) is exposed yet.
- **No per-cell labels** — deliberate, per the original design
  conversation (labels didn't seem to make sense here except possibly one
  label per small-multiple panel, which is exactly what `showSeriesLabel`
  already provides). Tooltips carry the same information on hover instead.
- **`categorySelect` only drives its own dimming/highlight effect right
  now, not Power BI's native cross-filtering** — same open question as the
  aster plot's `wedgeSelect`; worth testing Deneb's **Cross-Filtering
  (Selection) of Data Points** setting before building anything custom.
- **Two overlapping field parameters in one visual is confirmed working,
  but only tested against a static screenshot, not a live re-slice** — the
  exclusion-cascade fix (see "Field mapping" above) was verified by
  reproducing the exact data shape from a "view as table" screenshot
  locally, not by watching it update live in Power BI Desktop as someone
  actually changes either slicer. Worth a live re-confirmation, especially
  re-slicing one or both parameters to a few different candidates in a row
  and checking the small multiples update as expected.
- **If `MappedParameter` and `MappedParameterTwo` are both sliced to the
  same underlying column**, `SeriesCategory` falls back to `'All'` for
  every row rather than showing a real (duplicate) series value — see
  "Field mapping" above for why this happens and why it's the safer of the
  two possible failure modes.
- Color/font/grid-size params are edited directly in the spec JSON, same
  tradeoff as the aster plot (see that visual's README's "Setup in Deneb"
  section for why).
