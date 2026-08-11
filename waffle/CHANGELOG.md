# Changelog — Waffle Chart

## 0.7.0

**New `useRowBreaks` param (default `false`)** — an opt-in staircase/row-
break layout for the grid, added because color can't be used as a channel
for set identity in this report and the color-gradient measures compress
too smoothly against a fixed brand palette to visually separate categories
on their own. When enabled, a category's block is followed by a break to a
new row instead of continuing to fill the current row's remaining cells —
the next category's first cell lands one row down but at the *same*
column its cells would have continued into, not back at column 0,
producing a cascading-indent look.

- Two new fields feed the layout: `CategoryCountInSeries` (a
  `joinaggregate` count per `SeriesCategory`, computed right after the
  main `aggregate` step where there's exactly one row per category) and
  `RowsSpanned` per category (`ceil((CellStart % gridCols + (CellEnd -
  CellStart)) / gridCols)` — how many grid rows that category's own cell
  count actually occupies, given where it starts).
- Each category's starting row (`StaircaseRowStart`) is an **exclusive
  cumulative sum of `RowsSpanned` across all prior categories** in the
  same panel, sorted the same way as `CellStart`/`CellEnd`. First attempt
  used a flat `+1`-per-category-rank row offset instead, which is wrong
  whenever a category's own block spans more than one row — it doesn't
  account for how many *extra* rows that category actually consumed,
  causing later categories to overlap earlier ones' cells outright. Caught
  by an automated overlap check (dumping `(CellRow, CellCol)` per row,
  asserting no two categories claim the same pair) before it reached a
  screenshot.
- The exclusive cumulative sum itself is a `lag` of an inclusive
  cumulative `sum` (`frame: [null, 0]`), reusing the exact pattern already
  proven for `CellStart`/`CellEnd` in this spec — a direct `window` with
  `frame: [null, -1]` was tried first and, confirmed via an isolated
  minimal Vega-Lite compile, does not produce an exclusive prefix sum in
  this Vega-Lite version.
- Grid row capacity when the flag is on: `EffectiveGridRows = gridRows +
  CategoryCountInSeries`, used instead of the bare `gridRows` param when
  sizing cells and the grid's footprint height. A deliberately generous
  fixed safety margin rather than an exact computed minimum — verified
  against a 2000-trial randomized simulation that actual rows used never
  exceeds this bound.
- Verified via `tools/validate.mjs`, rendered previews with both flag
  states against single-grid and small-multiples sample data (staircase
  pattern renders correctly, contained within each panel, no cell
  overlaps in either case; `useRowBreaks: false` is pixel-identical to the
  pre-existing continuous-fill layout), and `tools/test-interaction.mjs`
  with the flag temporarily defaulted on (click-to-highlight still dims/
  highlights the correct cells across all panels).
- See the README's "Implementation notes" for the full math writeup and
  "Known limitations" for the practical-lower-limit-on-`gridRows` caveat.

## 0.6.1

**Fixed a DAX syntax error in `category-color-gradient-rank.dax`.**
`VAR Rank = ...` threw "The syntax for 'Rank' is incorrect" — Power BI
DAX has a real built-in `RANK` function now (part of the newer window-
function family alongside `ROWNUMBER`/`OFFSET`/`INDEX`), so a variable
named exactly `Rank` collides with it. Renamed to `RankValue`; the
per-candidate `Rank_EngagementCodeList`-style variable names were
unaffected (only the bare word collided). Documented the general rule in
`docs/patterns.md`: don't name a `VAR` after any DAX function, including
newer ones that predate this repo's older DAX files.

## 0.6.0

**New `dax/category-color-gradient-rank.dax`** (measure `Category2 Color
Gradient Rank`) — a rank-based alternative to the magnitude-based
`category-color-gradient.dax`. Instead of interpolating by value (even
log-scaled), this ranks categories by `Measure A` and spaces colors
**equally by rank position** — step size is a flat `1 / (member count −
1)`, independent of how close or far apart the underlying values are.

- Direction is deliberately reversed from the magnitude-based measure:
  rank 1 (the *largest* `Measure A`) gets the *lightest* color
  (`#A5C1FF`); the smallest-ranked category gets the *darkest*
  (`#4555D6`) — confirmed with the report author, not a copy-paste
  mistake carried over from the other measure.
- Ties (identical `Measure A`, expected to be rare) break alphabetically
  by category name. Implemented as a manual "count how many rows are
  strictly better" rank (1 + count of rows with a higher value, or an
  equal value with an alphabetically-earlier name) rather than `RANKX`,
  since `RANKX` only ranks by one expression with no built-in secondary
  tie-break — this comparison is a strict total order, so no two rows can
  ever land on the same rank.
- Same field-parameter structure and `ALLSELECTED` scope as
  `category-color-gradient.dax`, and named to avoid the exact model-wide
  measure-name collision fixed in 0.5.3.

Verified before shipping: the rank/step-size formula numerically against
the real 10-category data (equal ~0.111 steps between every rank), a
synthetic tie test (two categories tied at the same value resolved to
adjacent ranks, alphabetically-earlier one better, not sharing a rank),
and a full render against the real data showing evenly-stepped colors
end to end.

## 0.5.4

**Confirmed and fixed `MappedParameterTwo`'s actual label column name.**
Every DAX file referencing it (`category-fixed-color.dax`,
`border-fixed-color.dax`, `category-color-gradient.dax`,
`category-two-field-parameter.dax`) used the placeholder
`'MappedParameterTwo'[MappedParameterTwo]`, flagged as unconfirmed. It's
actually `'MappedParameterTwo'[Category2]` — updated all four files and
removed the now-stale "not yet confirmed" language.

## 0.5.3

**Fixed a measure-name collision that overwrote the aster plot's own
gradient measure.** `dax/category-color-gradient.dax` (added in 0.5.2)
named its measure `Category Color Gradient` — the exact same name as the
aster plot's pre-existing `visuals/aster-plot/dax/
measure-a-color-gradient.dax`, in the same shared model. DAX measure
names are unique per model, not per visual, so pasting the new one
overwrote the aster plot's original live measure. Renamed to `Category2
Color Gradient` (the DAX itself is unchanged, only the `Name =` line and
its own comments). The aster plot's original measure is unaffected in
this repo — its file was never touched — but if the collision already
happened live, re-paste `visuals/aster-plot/dax/
measure-a-color-gradient.dax` to restore it.

Documented the general rule in `docs/patterns.md`'s "Shared data model"
section: a new visual's DAX template must use a measure name that's
unique across the whole shared model, not just sensible for that one
visual — this is exactly the kind of thing worth checking before writing
the next DAX template for the next visual.

## 0.5.2

- **New `dax/category-color-gradient.dax`** — a magnitude-based color
  gradient alternative to the fixed/pinned `category-fixed-color.dax`,
  log-scale interpolated between `#A5C1FF` (lowest category-level
  `Measure A` total) and `#4555D6` (highest). Same structure as the aster
  plot's own `measure-a-color-gradient.dax` (per-candidate `MINX`/`MAXX`
  summary tables, `MAX` instead of `SELECTEDVALUE` for the
  `MappedParameterTwo` composite-key constraint, log rather than linear
  scale). Verified the interpolation numerically against the real
  10-category data reported earlier in this visual's development before
  shipping — colors spread visibly across the full range instead of
  clustering near one end, and rendering all 10 against the current spec
  confirms every category stays distinct and correctly ordered.

## 0.5.1

**Fixed the real bug behind both "sort isn't working" and "is it not
rendering the full set of members" — they were the same root cause.**
The cumulative-sum `window` transform (producing `CellEnd`) had `"sort":
[{"field": "CategoryTotal", "order": "descending"}]`, but the very next
`window` transform (a `lag` on `CellEnd` producing `CellStart`) had no
`sort` of its own — a `"sort"` on one `window` step does not carry over
to another. The `lag` step silently used incoming (alphabetical) row
order instead, so each category's `CellStart` became "the previous row
*alphabetically*'s `CellEnd`" rather than "the previous row *by size*'s
`CellEnd`," producing wrong — sometimes **negative-width** — ranges for
categories whose alphabetical and size-sorted positions differed enough.
`sequence(start, stop)` with `start > stop` returns an empty array, and
`flatten` on an empty array produces zero rows, so affected categories
didn't just render out of order — they vanished completely, including
from the legend (built from whatever `Category` values actually reach
the mark). Traced by dumping the pre-`flatten` rows directly against the
real reported data and finding `CellEnd < CellStart` on exactly the four
categories missing from the reported legend (Goldman Sachs, Mehta
Dharmesh, Chopra Rajiv, Bhutani Gitanjali). Fixed by adding the identical
`"sort"` to the `lag` window transform. Documented as a general pattern
in `docs/patterns.md` — chained `window` transforms never inherit a
prior step's sort, and Vega-Lite gives no warning when they disagree.

Re-verified against the exact real data reported: all 10 categories
present in the legend, contiguous non-overlapping cell ranges for every
category, Goldman Sachs (the largest) correctly filling first from the
top-left through strictly decreasing categories after that. Re-checked
the existing 5-series small-multiples scenario and the click-to-highlight
interaction for regressions — both still correct.

## 0.5.0

**Fixed a severe cell-allocation bug, reported as "sort isn't working."**
The sort added in 0.3.1 was actually correct — the *magnitudes* feeding it
weren't. Root cause: `CategoryTotal` was computed via `joinaggregate`,
which adds an aggregate to every existing row without reducing the row
count. That's fine with exactly one row per category, but `Category` is
fact-grain data (e.g. a customer/custodian name) with real data spanning
multiple underlying rows per category — and the later cumulative-sum
window summed `CategoryTotal` across every one of those rows, counting
each category's total once per underlying row instead of once total.
Reproduced locally with 5 rows for one category and 1 for another: a
100-cell grid that should split 95/5 came out 477/4 across 481 total
cells. Fixed by replacing that `joinaggregate` with a real `aggregate`
(collapses to one row per category) immediately after the field-mapping
adapter, before any cumulative math runs. Re-verified against the exact
reproduction case (now correct: 95/5, 100 total cells) and the original
10-real-category sort-order render (Goldman Sachs still fills first,
Arya Shivank still last, colors now correctly proportioned rather than
just correctly ordered).

Tooltip's measure field renamed from `MetricA` to `CategoryTotal` (titled
"Measure A") to reflect that it's now genuinely the category's total,
computed once, rather than a raw per-row value that happened to look
right only when there was one row per category.

## 0.4.0

- **New `elementFillBorderGap` param** (pixels, default `0`) — opens a
  visible gap between an element's fill and the inside edge of its
  border. Required splitting the single fill+stroke point mark into two
  layers (an outer stroke-only ring at the existing `ElementSize`, an
  inner fill-only circle at a new, smaller `FillSize`), since one mark
  can't have its stroke stop short of its own fill. `elementFillBorderGap:
  0` renders identically to before (verified the math resolves to the
  same edge, not just eyeballed). The click-to-highlight selection
  (`categorySelect`) now lives on the border layer and is read by the
  fill layer's opacity condition — confirmed both layers dim/highlight
  together by simulating a selection directly and checking the compiled
  SVG's opacity attributes, rather than relying on exact pixel-coordinate
  clicks.
- **Clarified `gridArea` accepts any decimal between `0` and `1`, but
  needs a leading zero** (`0.753`, not `.753`) — confirmed `.753` alone
  isn't valid JSON (`JSON.parse` throws on it) while `0.753` parses fine;
  this is a JSON syntax requirement from hand-editing the Specification
  tab as raw JSON, not a restriction the param itself imposes.

## 0.3.1

- **Categories now fill each grid/panel in descending `MetricA` order**
  (largest first, top-left) instead of incoming data order — a single
  `"sort"` clause added to the existing cumulative-window transform that
  computes each category's cell range (`"sort": [{"field":
  "CategoryTotal", "order": "descending"}]`, scoped per `SeriesCategory`
  like the rest of that window). No new params, no change to the cell-index
  math itself — just which category claims which range. Verified via a
  render with 10 distinctly-colored, very-differently-sized categories:
  the largest fills first from the top-left, strictly decreasing after
  that, independently within every panel of a small-multiples layout.

## 0.3.0

Fixed a real, reported layout bug and added the requested grid-sizing
control:

- **Fixed: a single grid (no `SeriesCategory` bound) rendered into roughly
  a quarter of the visual instead of using the full width** — caught from
  a screenshot showing a 20×20 grid squeezed into the top-left. Root
  cause: panel width was always computed as `width / seriesColumns`
  (default `4`), even with only one actual panel. Fixed with
  `EffectiveSeriesColumns = min(seriesColumns, TotalSeriesCount)`, used
  everywhere `seriesColumns` previously was — a single grid now correctly
  gets the full width, and small multiples with fewer series than
  `seriesColumns` get wider panels instead of wasted empty columns.
- **New `gridArea` param (`0`–`1`, default `1`)** — the grid now computes
  one uniform cell size to fit `gridRows × gridCols` within that fraction
  of the available panel space, then centers the resulting grid (both
  axes) within the full panel. `gridArea: 1` preserves the original
  edge-to-edge fill; anything less visibly shrinks and centers the grid.

Verified locally: rendered a 20×20 single grid before/after the
`seriesColumns` fix (quarter-width → full-width), rendered at `gridArea:
0.6` (shrunk and evenly centered), re-rendered the existing 3-series and a
new 2-series small-multiple scenario (both still lay out and center
correctly), and re-ran `validate.mjs`.

## 0.2.0

`Category`/`SeriesCategoryRaw` are now correctly field-parameter-driven,
fixing a real bug caught before it shipped, not a preemptive change:

- **Both `Category` (from a new field parameter, `MappedParameterTwo`) and
  `SeriesCategoryRaw` (from the aster plot's existing `MappedParameter`)
  are field-parameter-driven, over the same 4 candidate columns.** A live
  "view as table" screenshot (with the two parameters sliced to different
  candidates) confirmed Deneb exposes both as separate, simultaneous keys
  — no forced collision — but a naive `isValid()` cascade (the single-role
  pattern from `docs/patterns.md`, just copy-pasted for the second role)
  resolved **both** roles to the same value, reproduced locally before it
  was understood. Root cause: both cascades check the same 4 candidates in
  the same order, so whichever one is present and checked first wins for
  both roles regardless of which field parameter it actually came from.
- **Fixed with an exclusion cascade**: `Category` resolves first (normal
  priority-order cascade), also recording which candidate name it used
  (`CategorySource`); `SeriesCategoryRaw`'s cascade then only considers
  candidates that name doesn't match. `SeriesCategoryRaw`'s terminal
  fallback had to change from `'(Unspecified)'` to `null` — a string
  fallback passes `isValid()`, which was silently breaking the "series
  role unbound → `SeriesCategory` = `'All'`, single-grid mode" degradation.
  Verified against three scenarios (compiled and run directly, not just
  reasoned about): both roles on different candidates (matches the live
  screenshot exactly), the series role's field parameter fully unbound,
  and both field parameters sliced to the identical column.
- Documented the general "two field parameters, same candidates, one
  visual" pattern in `docs/patterns.md`, since this is very likely to come
  up again for a future visual needing two independent swappable roles.
- Added `dax/category-two-field-parameter.dax` (the new
  `MappedParameterTwo` table) and updated both fixed-color DAX templates
  to reference it correctly (`MAX`, not `SELECTEDVALUE`, per the same
  composite-key constraint as the aster plot's `MappedParameter`).
- Updated `sample-data.json` to exercise the real field-parameter shape
  (`EngagementCodeList`/`AlphaCode` as separate simultaneous keys) instead
  of the plain `Category1`/`Category2` renames assumed in 0.1.1.

## 0.1.1

Wired the field-mapping adapter to this report's real field names (same
underlying model as the aster plot, intended to be shared across every
visual in this library going forward):

- `Category1` → `SeriesCategoryRaw` (the series/small-multiple role)
- `Category2` → `Category` (the grid grouping/legend role)
- `Measure A` → `MetricA` (note the space in the real field name, handled
  via `datum['Measure A']` bracket notation instead of dot notation)
- `ColorOverrideFill` → `ColorOverride`
- `ColorOverrideBorder` → `BorderColorOverride`

`Category1`/`Category2` weren't labeled by role when given, so which one
feeds the series vs. the legend/grouping is a documented assumption (see
the README's "Fields expected" section) rather than a confirmed mapping —
a one-line swap if it's backwards. `ShapeOverride` has no confirmed
binding yet and stays unbound (falls back to `defaultShape`).

Updated `sample-data.json` and both DAX templates to match. Re-verified
locally (compile, static render, click-interaction) against the renamed
fields — unchanged behavior, as expected for a pure rename.

## 0.1.0

First version. A grid of shapes per category, sized proportionally to
`MetricA`'s share within its `SeriesCategory` group, with `SeriesCategory`
optional — bound, it produces one small-multiple grid per distinct value
(with a panel title); unbound, everything collapses into a single grid.

Built following `docs/design-system.md`/`docs/patterns.md` (established
while building the aster plot) — dark-canvas palette (`#182231` background/
border, `#FCFCFD` title tier, `#A5B4CB` subtitle/legend tier), Segoe UI,
point-based font sizing, the field-mapping adapter pattern, `params`-only
(no Config-tab dependency), and the same corner-vs-edge legend/centering
mechanism.

- Core grid mechanism: `sequence()` + `flatten` to expand one row per
  category into many per-cell rows — verified via an isolated compile test
  (3 category rows → exactly 100 correctly-distributed flattened rows)
  before building the rest of the spec around it.
- Cell allocation per category reuses the aster plot's cumulative-window
  pattern (`window` cumulative sum + `lag`), grouped by `SeriesCategory`,
  so rounding always lands each series exactly on a full grid.
- Small multiples via manually-computed panel layout (`dense_rank` for
  panel indexing + plain row/col arithmetic), not Vega-Lite's native
  `facet`/`repeat` — chosen specifically so a single spec can degrade from
  small-multiples to a single grid at runtime just by unbinding
  `SeriesCategory`, which a structural `facet` wrapper can't do.
- Optional `BorderColorOverride` and `ShapeOverride` fields, both following
  the same `isValid()`-fallback pattern as the aster plot's
  `LabelColorOverride` — shape specifically supports both Vega's built-in
  shape names and custom SVG path strings interchangeably in the same
  field, so one mechanism covers "default circle" and "custom per-category
  shape."
- `categorySelect` click-to-highlight interaction (dim non-matching
  elements), matching the aster plot's `wedgeSelect` convention — confirmed
  via a real click-interaction test that clicking one category highlights
  every matching element across all panels simultaneously.
- `gridHorizPadding`/`gridVertPadding` params added shortly after the
  initial build, for explicit pixel spacing between adjacent elements
  (on top of the existing `elementSizeRatio` proportional fill control) —
  confirmed via a render test that increasing either visibly shrinks
  elements and opens a gap without affecting grid/panel layout.
- Sample data's initial `ColorOverride` values (copied loosely from the
  aster plot's old gradient example) turned out nearly invisible against
  the new dark background for two categories — not a spec bug, just poorly
  chosen synthetic colors; replaced with Power BI's default theme palette
  (already used in `tools/deneb-shims.mjs`), which stays visibly distinct.

Verified locally: compile/validate, static render (single-grid and
small-multiple, default and with padding), and a real click-interaction
test. Not yet confirmed live in Power BI Desktop, and no real Power BI
model/field names have been confirmed against it yet — see the README's
"Fields expected" note.
