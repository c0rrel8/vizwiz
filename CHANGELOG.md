# Changelog — Aster Plot

## 0.10.0

Design-system pass: explicit background/text-color palette, point-based
Segoe UI typography with scaling off by default, and a real title/subtitle
text hierarchy (not just a single color per text block). Verified locally
by rendering (default + interactive click-to-select + `labelPosition:
"inside"`) against a dark background; not yet re-confirmed live in Power BI
Desktop. This version's color/font/scaling decisions are also written up
in the repo-root `docs/` folder so the next visual can start from the same
system instead of rediscovering it — see `docs/design-system.md` and
`docs/patterns.md`.

- **Background and wedge `borderColor` now default to `#182231`** (both
  the same value, so wedges read as separated by a thin gap matching the
  page rather than a bright stroke) — was `"transparent"`/`#ffffff`.
- **Title-tier text (`labelColorOutside`, `centerFontColor`,
  `legendLabelColor`/`legendTitleColor`) now defaults to `#FCFCFD`**
  (previously `#F4F6FB`, chosen in 0.9.0 as a first approximation before
  exact brand values were provided).
- **New subtitle tier, `#A5B4CB`, and a real 2-mark title/subtitle split**
  — every text block in this visual (wedge label, both center-label views)
  now renders as two independently-colored marks instead of one: a
  title-tier mark for the primary text (category name; center title/
  selected-category name) and a new subtitle-tier mark for secondary text
  (MeasureA/MeasureB/percent "sublabels" on wedges; category count/totals
  or selected-wedge stats in the center). New params: `labelSubtitleColor`,
  `centerSubtitleColor` (legend already had its own pair from 0.9.0, now
  also defaulted to the same `#A5B4CB`).
  - Splitting the wedge label into two marks initially **reintroduced the
    dy-can't-be-per-datum overlap problem** documented back in 0.9.0/0.8.0
    — a naive dy split (assuming a 1-line title) collided the sublabel
    into the title's wrapped second line for any 2-line category name.
    Fixed properly, not approximated around: the wedge title/sublabel
    marks no longer use `theta`/`radius`/`dy` at all — they use a
    per-datum Cartesian anchor (`LabelAnchorX`/`LabelAnchorY`, the same
    trig already used for leader lines) plus each row's *actual*
    title/sublabel line counts, computed as plain calculated fields rather
    than mark-level `dy`. This also happens to **fully fix** the
    longstanding "wedge labels aren't perfectly centered when wrapped"
    known limitation, not just avoid regressing it.
- **Segoe UI, point-based font sizing, scaling off by default:**
  - `labelFontFamily` (new) and `centerFontFamily` both default to
    `"Segoe UI"` (was unset/`sans-serif`).
  - `labelFontSize` (12, was 11) and `centerFontSize` (14, was 20) are now
    interpreted as **points**, not pixels, converted via new derived
    `labelFontSizePx`/`centerFontSizePx` (`pt * 4/3`, the standard 96-DPI
    conversion) — matches how Word/PowerPoint/Power BI's Format pane think
    about font size. Anyone with a hand-edited `labelFontSize`/
    `centerFontSize` from an earlier version should expect a different
    rendered size now (a deliberate breaking semantic change, not a bug).
  - `labelAutoSize` now defaults to `false` (was `true`); new
    `centerAutoSize` param (default `false`) gates the center label's
    auto-shrink-past-4-lines math the same way — both label systems are
    now static-size by default, auto-scaling only when explicitly turned
    back on.
- **README**: reorganized "Color" around the title/subtitle/sublabel/legend
  hierarchy with a table of which param controls which tier, updated the
  Params reference and Config-tab gap note, and resolved (rather than just
  updated) the wedge-label word-wrap-centering "Known limitation" entry.

## 0.9.0

Dark-canvas defaults + more color/legend control, prompted by a screenshot
of the actual dashboard this visual sits in (dark navy background, other
visuals' text rendering light/white). Verified locally by rendering against
a dark background and a real click-to-select interaction test; not yet
re-confirmed live in Power BI Desktop.

- **`labelColorOutside` and `centerFontColor` now default to `#F4F6FB`**
  (near-white) instead of black/near-black — the previous defaults assumed
  a light report canvas and were nearly invisible against this dashboard's
  dark theme. `labelColorInside` is unchanged (`#ffffff`), since inside
  labels sit on a colored wedge fill either way, not the page background.
- **New `legendLabelColor`/`legendTitleColor` params** — the legend's entry
  and title text now default to the same near-white, instead of silently
  inheriting Vega's plain-black default (which was equally invisible on a
  dark canvas, and wasn't controlled by any existing param).
- **New `centerSelectedColorFromWedge` param** (`false` by default) — when
  `true`, the selected-wedge center-label view uses that wedge's own
  `ColorOverride` value for its text color instead of the static
  `centerFontColor`. Answers "is there a color measure for the center
  label?" — not a DAX measure exactly, since the selected view's data is
  already the filtered single row (which already carries `ColorOverride`),
  so this just opts into reusing it. The default (no-selection) view has no
  equivalent, since it's a cross-category aggregate with no single row's
  color to use.
- **README overhaul**: reorganized all label/legend/center color behavior
  under "Color" (previously split oddly across two sections), added a
  "Dark-canvas defaults" note, fully documented every param including which
  ones are internal/derived and shouldn't be hand-edited, clarified exactly
  which mark/encoding properties this spec's params already override vs.
  what the Config tab can still affect (in short: almost everything now —
  the one remaining gap is the wedge-label text mark's font family/weight,
  which aren't yet params and still fall through to Config/Vega defaults),
  and explained the existing `borderColor`/`borderWidth` wedge-border
  params directly (no separate "border thickness" control — it *is*
  `borderWidth`).

## 0.8.1

- Removed `dax/measure-field-parameter-pattern.dax` — confirmed unused
  (nothing in the spec or other DAX files referenced it) and never actually
  adopted; `MeasureA`/`MeasureB` don't have the dynamic-naming problem it
  was written for, since ordinary measures rename freely to a fixed name
  per-visual. Updated the README's "Field mapping" section to drop the
  reference.

## 0.8.0

First cosmetics pass since the functional pipeline was confirmed working
end-to-end (0.7.0). All changes verified locally via `tools/validate.mjs`
and rendered/screenshotted comparisons (right-orient, corner-orient, and
hidden legend; a synthetic stress test mirroring the real TopN=10 report
with a dominant 73%-share wedge, many small wedges, and a row with no value
for any of the 4 category candidates); not yet re-confirmed live in Power
BI Desktop.

- **Outside wedge labels now sit on one constant ring** (`outerRadius +
  labelOutsideOffset`) instead of each wedge's own outer edge — fixes the
  crowded, staggered label layout reported on a real TopN=10 view. Leader
  lines now converge on a clean common radius instead of each ending at a
  different distance from center.
- **Wedge label font size auto-scales with each wedge's own angle** by
  default (new `labelAutoSize`, `labelFontSizeMin`, `labelFontSizeMax`,
  `labelAutoSizeAngleCap` params) — narrow wedges get smaller text, wide
  wedges get larger text, instead of one fixed size that's too big for
  slivers and too small for a dominant wedge. Set `labelAutoSize: false` to
  restore the old fixed-`labelFontSize` behavior. (Confirmed via a Vega-Lite
  compile test that mark `size` — unlike `dy`/`lineHeight` — *is* a valid
  per-datum text encoding channel.)
- **New optional `LabelColorOverride` field** (5th line of the field-mapping
  adapter) lets a DAX measure drive wedge label color per-category, falling
  back to the existing `labelColorOutside`/`labelColorInside` static logic
  when not bound or null for a given row.
- **New `showLegend`/`legendOrient` params** make the legend's visibility
  and position live spec settings instead of a hardcoded `"orient": "right"`.
  Choosing a corner orient (`top-left`/`top-right`/`bottom-left`/
  `bottom-right`) or hiding the legend also solves the reported off-center
  donut: corner orients (and a hidden legend) don't reserve layout space,
  so the donut centers on the full visual frame regardless of legend
  content, whereas edge orients (`left`/`right`/`top`/`bottom`) reserve
  space and shift the donut within what's left — this doubles as the
  requested "centering toggle" without a separate param. See the README's
  new "Legend position and chart centering" section.
  - Hiding the legend (`showLegend: false`) turned out to need two
    workarounds, both documented in the README's "Implementation notes":
    pushing `legendX`/`legendY` off-canvas to hide it blows up the
    rendered SVG size and shifts the whole chart out of view (Vega's
    autosize includes off-canvas content in its bounding box), and a
    `legend.encode` opacity toggle silently gets replaced by Vega-Lite's
    auto-generated `symbols` stroke encode (from the wedge border color
    feature) rather than merged with it. Fixed by keeping `legendX`/
    `legendY` on-canvas and zeroing the legend's own `symbolSize`/
    `labelFontSize`/`titleFontSize` properties instead of using `encode`.
- **`Category` now falls back to `'(Unspecified)'`** instead of a raw
  `"undefined"` string when a row has no value for any of the 4 mapped
  candidate fields — spotted in a real TopN=10 screenshot's legend.

## 0.7.0

- Fixed the real, confirmed root cause of the visual rendering a single
  `"undefined"` category for every row (legend collapsing to one value,
  center label reading "1 categories") despite `MeasureA`/`MeasureB`/
  `ColorOverride` all varying correctly. Two prior fixes assumed Deneb
  would receive the field-parameter-driven Category data keyed by either
  the field parameter table's name (`MappedParameter`) or its label
  column's name (`Category`) — both wrong. Confirmed live (Deneb's "view
  as table" and a plain Power BI table visual showed the same column
  header): Power BI exposes this data keyed by the **real underlying
  column's own name** — whichever of `EngagementCodeList`/`AlphaCode`/
  `CustodianDisplayName`/`EFileName` is currently selected — not by
  anything on the field parameter table itself. Since that name changes
  with the live slicer selection and no static line can cover a moving
  target, the adapter now enumerates all 4 known candidates via
  `isValid()`, mirroring the same enumeration the DAX `SWITCH` already
  uses for the same underlying constraint.
- **Confirmed live**: full pipeline working end-to-end against real
  production data for the first time — 41 real categories rendering with
  correct proportions and visibly varying gradient colors. Everything
  from here is cosmetic/polish, not functional.

## 0.6.9

- The composite-key fix (0.6.8) is confirmed working — verified live data
  showed the gradient formula was mathematically exact (hand-checked two
  rows' hex output against the formula precisely). The "every wedge looks
  the same color" symptom wasn't a bug: `MeasureA` here is heavily
  right-skewed (a few outliers over 1000+ against dozens of values under
  120), so a linear min→max gradient compresses almost everything near
  `MinVal` -- a value at 4% of the range is visually indistinguishable
  from 0%. Switched the interpolation to a log scale (`LOG(x + 1)`, guarding
  `LOG(0)`), verified against the real data to spread colors clearly across
  the whole range instead of collapsing most of it to near-identical dark
  blue.

## 0.6.8

- Found the actual, documented root cause of the composite-key error
  (confirmed via SQLBI, not a guess this time): Power BI sets a
  "Group By Columns" property on field-parameter-generated tables,
  linking the Value/Fields/Order columns into a composite key. Touching
  any single one of them via a filter-context function --
  `SELECTEDVALUE` or `ALLSELECTED` alike -- throws "part of composite
  key," confirmed live for both. The fix is `MAX` instead: it returns the
  same single value for a single-select slicer without triggering the
  check. Fixed `dax/measure-a-color-gradient.dax` and, since it carried
  the same latent bug, `dax/measure-field-parameter-pattern.dax` too
  (unused today, but kept as a reference pattern, so it shouldn't
  propagate the same mistake).

## 0.6.7

- Reordered all 3 `dax/*.dax` files so the `Name =` line comes first,
  with explanatory comments following it — reported live that a comment
  block preceding the name/assignment line breaks Power BI's DAX editor
  (it appears to parse the measure/table name from the first line).
  Content unchanged, only the ordering.

## 0.6.6 (gradient DAX restructured, not yet confirmed live)

- Restructured `dax/measure-a-color-gradient.dax` to remove `SWITCH`
  branching between whole `ADDCOLUMNS`-built tables stored in a `VAR` —
  `SWITCH` is built for scalar branches, and a table-valued `VAR` consumed
  later by `MINX`/`MAXX` proved unreliable in practice (reported live: a
  casing mismatch appeared between the variable's declaration and its
  later references, and the MinVal/MaxVal computation broke). Rebuilt so
  each of the 4 field-parameter candidates gets its own plain MinVal/MaxVal
  scalar variable (8 variables, each an ordinary, unambiguous `MINX`/`MAXX`
  over its own table expression computed inline — no table is ever stored
  in a `VAR` and branched on later). `SWITCH` now only ever picks between
  already-computed scalars, which is unambiguously valid DAX.

## 0.6.5

- Fixed the actual root cause behind the "column doesn't exist" error:
  the `MappedParameter` field parameter is a *table*, but the label/value
  column actually dragged into the Values well — the one Deneb sees — is
  a *column within that table* named `Category` (confirmed via the Fields
  pane: `MappedParameter` → `Category` / `MappedParameter Fields` /
  `MappedParameter Order`). Earlier fixes incorrectly conflated the two,
  assuming Deneb would see the table's own name. Reverted the spec's
  field-mapping adapter to read `datum.Category` (a no-op, matching the
  column's real name) and fixed the DAX to
  `SELECTEDVALUE('MappedParameter'[Category])`.
- Caught a silent bug this introduced in local testing:
  `sample-data.json` still keyed rows by `MappedParameter` from the
  previous (wrong) adapter version, so after reverting the adapter to
  `datum.Category`, every row's `Category` was silently `undefined` —
  `validate.mjs` still passed, since it only catches thrown errors, not
  semantic correctness. Caught by re-rendering and looking, not by the
  compile check alone. Fixed by re-keying the sample data to `Category`.

## 0.6.4 (gradient DAX known broken, pending input)

- Removed `@` from `ADDCOLUMNS`-created column names in
  `dax/measure-a-color-gradient.dax` (`"@CategoryTotal"` →
  `"CategoryTotalTemp"`) — reported live as breaking the measure; not
  worth the convention if it doesn't work in this editor.
- Confirmed live: `'MappedParameter'[MappedParameter]` does not exist in
  this model — the standard Power BI field-parameter naming convention
  (`<Name>`, `<Name> Fields`, `<Name> Order`) assumed for the label column
  was wrong for this table. Marked with `FIXME`/a placeholder column name
  rather than guessing again; waiting on the real column name before
  re-fixing.

## 0.6.3

- Wired the spec's field-mapping adapter and the gradient DAX to this
  report's real objects: `MappedParameter` (the Category field parameter,
  4 candidates over `Project_snapshot_Fact`), `[Data Recieved GB]`
  (MeasureA), `[Count Item]` (MeasureB), `[Category Color Gradient]`
  (ColorOverride). Added a concrete field-bindings table to the README so
  it's unambiguous what to drag into the Values well and rename, instead
  of only generic placeholder guidance.
- Replaced sample-data.json's values twice while settling on the right
  approach: first with numbers read by eye off a screenshot (too
  imprecise, and risked being mistaken for real data), then with clearly
  synthetic round-number placeholders. Local sample data has no live
  connection to any real Power BI data and never can from this
  environment, so it shouldn't imply otherwise — documented plainly in
  the root README.

## 0.6.2

- Fixed `dax/measure-a-color-gradient.dax` again: `ALLSELECTED()` with no
  arguments throws "ALLSELECTED function without parameters cannot be used
  as a table expression. It can appear only as a filter in CALCULATE" — it
  can modify a filter context inside `CALCULATE`, but can't be handed to
  `ADDCOLUMNS` as an iterable table, which is what the previous fix tried
  to do. There's no way to make this fully generic: DAX can't dynamically
  dereference "whichever column the field parameter currently points at,"
  the same static-reference constraint as Vega-Lite's own field bindings.
  Fixed by enumerating the field parameter's candidates explicitly via
  `SWITCH` on `SELECTEDVALUE('Category'[Category Order])`, with each
  branch doing a normal `ALLSELECTED` on one real physical column — no
  composite key involved, since it never touches the field parameter
  table. This needs to be kept in sync with
  `dax/category-field-parameter.dax`'s candidate list by hand — an
  inherent coupling, not a bug. Not yet confirmed live.

## 0.6.1

- Fixed `dax/measure-a-color-gradient.dax` for the field-parameter-driven
  Category case: `ALLSELECTED(FieldParameterTable[Category])` throws
  "Column [Category] is part of composite key, but not all columns of the
  composite key are included" (field parameter tables carry an internal
  composite key; touching one column of it directly isn't valid) — and
  even without that error, it was the wrong table: that column holds the
  field parameter's own candidate labels, not the actual category values
  within whichever field is currently selected. Switched to
  `ALLSELECTED()` with no arguments, which reflects "everything currently
  grouped in the visual, at whatever grain it's using" regardless of which
  physical column the field parameter currently points at. Not yet
  confirmed live — needs verification in Power BI Desktop.

## 0.6.0

- Added a field-mapping adapter to the top of the spec's `transform` array:
  4 `calculate` steps mapping the raw incoming `Category`/`MeasureA`/
  `MeasureB`/`ColorOverride` fields to the same-named internal canonical
  fields everything else in the spec references. A no-op today, but it
  isolates any future field rename down to editing these 4 lines instead
  of hunting through the whole spec — Vega-Lite field references are
  static strings with no indirection mechanism, so this is the practical
  equivalent of a configurable mapping.
- Added `dax/category-field-parameter.dax`: the confirmed-working pattern
  for a slicer-swappable Category dimension via a Power BI field
  parameter. Documented a real constraint hit during testing: a field
  parameter's name can't be overridden per-visual like an ordinary
  column's can, so it must be named exactly `Category` in the model —
  this framework assumes one such field parameter per report; the README's
  new "Field mapping" section documents the one-line change needed for a
  second, independent visual.
- Added `dax/measure-field-parameter-pattern.dax`: the equivalent optional
  pattern for making `MeasureA`/`MeasureB` slicer-swappable too (not
  required, since ordinary measures rename freely per-visual).

## 0.5.1

- Fixed `dax/measure-a-color-gradient.dax`: min/max were being computed over
  the underlying fact table's raw rows (`MIN`/`MAX` directly on the column),
  not over category-level totals. Rebuilt using a category-level summary
  table (`ADDCOLUMNS` over `ALLSELECTED(Category)`, each row computing that
  category's `SUM(MeasureA)` via `CALCULATE`) with `MINX`/`MAXX` over that
  summary's computed column instead — the two only coincide when every
  category has exactly one row.

## 0.5.0

- **Word wrap** for wedge labels and the center title: a regex-based
  character-width wrap (`labelWrapWidth`/`centerWrapWidth`), since Vega
  expressions have no real text-measurement capability.
- **Label overlap prevention**: `labelMinAngleDegrees` hides the label
  (and leader line) entirely for wedges narrower than the threshold — a
  heuristic, not true collision detection (documented limitation).
- **Leader lines** from each wedge's outer edge to its (outside-only)
  label, via a `rule` mark. Vega-Lite's `rule` mark doesn't support
  `theta`/`radius` polar encoding at all (confirmed by compiling), so the
  endpoints are computed manually with trigonometry — verified against
  Vega-Lite's own internal theta-to-pixel conversion to the exact pixel.
  Caught and fixed a real bug along the way: `rule` doesn't get the
  automatic center-of-view offset that arc/text marks do, so the first
  attempt scattered every line across the canvas instead of connecting
  wedges to labels.
- **Center label auto-shrink + wrap fit**: font size now scales down past
  4 lines (`centerDefaultFontSize`/`centerSelectedFontSize`), and the
  title wraps via the same regex technique — computed as a **param**
  expression (not a per-row calculation) since the title is spec-level
  static text, so its vertical centering stays exact regardless of how
  many lines it wraps to.
- **Calculated (gradient) colors**: added
  `dax/measure-a-color-gradient.dax`, a DAX measure that linearly
  interpolates a hex color between `#263449` (min `MeasureA`) and
  `#4555D6` (max `MeasureA`) per category, manually parsing/rebuilding hex
  since DAX has no native hex/decimal conversion. Switched the arc mark's
  `color` scale back to the `ColorOverride` field-passthrough (from
  `pbiColorNominal`) as the default, since a calculated gradient is a
  pinned-per-category value, not a theme scheme; `pbiColorNominal` is kept
  documented as an alternative.
- Fixed the actual root cause of a real-world `NaN` bug during testing:
  `MeasureA`/`MeasureB` were bound to non-numeric fields in the data
  model (confirmed by Power BI's own Σ-icon convention distinguishing
  recognized numeric measures from plain columns) — not a field-naming
  problem, which had already been ruled out by confirming the Values-well
  display names matched the spec exactly.

## 0.4.1

- Added `config/base-config.json` at the repo root: a corrected version of
  Deneb's bundled default Config, fixing text/axis/legend colors
  (`#CCD5E3`) that are nearly invisible against a light report canvas —
  confirmed by rendering the original and corrected versions side by side
  (the legend title/labels were washed out to the point of illegibility
  in the original). Everything else (transparent view stroke, empty `arc`
  styling, fonts, mark-type defaults) is unchanged from the bundled default.

## 0.4.0

- Switched default wedge color from the `ColorOverride` hex-passthrough
  fallback to Deneb's native `pbiColorNominal` scheme, which automatically
  matches the live Power BI report theme — confirmed via deneb.guide's
  Theme Colors & Schemes docs (`WebFetch` to deneb.guide is blocked by this
  environment's egress policy, but `WebSearch` isn't, so the docs remain
  reachable that way). `ColorOverride` is kept as a documented opt-in for
  pinning specific per-category colors instead.
- Removed the spec's top-level `config` key entirely. Deneb splits
  Specification/Config/Settings into separate editor tabs by design (per
  the Visual Editor docs), and a spec-level `config` risks conflicting with
  whatever's in the separate Config tab — Deneb's own default Config
  already provides a transparent view background and report-matching
  fonts, so no spec-level override was needed anyway.
- Added `tools/deneb-shims.mjs`: registers a stand-in for `pbiColorNominal`
  (Power BI's default theme palette) so local validate/preview tooling can
  still compile+run a spec that references Deneb-injected, runtime-only
  scheme names.
- Documented Deneb's Settings-tab interactivity toggles (Tooltip Handler,
  Cross-Filtering of Data Points, Resolve Data Points in Context Menu) and
  flagged Cross-Filtering as a likely no-code way to bridge the existing
  `wedgeSelect` param to Power BI's native cross-filter.

## 0.3.0

- Added `labelPosition` (`"outside"`/`"inside"`) to control wedge label
  placement, replacing the previous fixed just-outside-the-arc position.
  Default label color now follows position (black outside, white inside)
  via `labelColorOutside`/`labelColorInside`, replacing the earlier
  per-category luminance-contrast heuristic — both easily overridable.
- Fixed a real vertical-centering bug: multi-line text marks only center
  their *first* line on the anchor point, then stack remaining lines below
  it, so the 3-line center label was rendering ~23px below the donut's true
  center. Fixed with a computed `dy` correction based on the actual number
  of visible lines; applied to the center layers and wedge labels alike.
  Confirmed via direct SVG coordinate inspection, not just a screenshot.

## 0.2.0

- Renamed generic `Weight`/`Score` fields to `MeasureA`/`MeasureB`.
- Added `ColorOverride` field: per-category wedge color via direct hex
  passthrough (`scale.range.field`), since Deneb can't expose a native
  per-category Format-pane swatch picker for a generic spec. Documented the
  DAX pattern to populate it.
- Auto-contrast wedge label color via WCAG `luminance()` against each
  wedge's own color — no extra field needed.
- Configurable border color/width (`borderColor`, `borderWidth`).
- Configurable wedge labels: toggle category name, MeasureA, MeasureB, and
  % of grand total independently; unit-scaled number formatting
  (`measureFormat`, default D3's `~s`).
- Added a center label with configurable font family/size/weight/color that
  shows a dataset-wide summary (title, category count, measure totals) by
  default, and switches to the clicked wedge's own detail (actual + % of
  grand total per measure) via a click-to-select param (`wedgeSelect`).
- Added `tools/test-interaction.mjs` + `tools/interactive-harness.html`: a
  headless-browser click-simulation test, since compile+run validation alone
  can't catch interaction bugs (used to find and fix two of the three bugs
  below).
- Fixed three real bugs caught via compile+run and headless-browser testing
  (see the visual's README "gotchas" section for detail): a stack-groupby
  quirk when combining quantitative `theta`+`radius`; a per-datum vs. global
  selection-emptiness test mismatch that kept the default center label
  visible after a click; and `{"filter": {"param": ...}}` matching every row
  by default when the selection is empty, which rendered all 7 categories'
  text stacked at the center before any click.

## 0.1.0

- Initial migration from the D3 v3 `d3.layout.pie()` + `d3.svg.arc()`
  reference implementation to a Vega-Lite spec for Deneb.
- Wedge angle (`theta`) encodes `Weight`; wedge radius encodes `Score`,
  scaled between a container-responsive inner and outer radius.
- Category color legend, per-wedge tooltips, and centered wedge labels.
