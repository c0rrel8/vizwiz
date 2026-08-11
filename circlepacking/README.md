# Circle Packing

A 3-level hierarchy (Level 1 > Level 2 > Level 3) rendered as nested,
non-overlapping circles — the classic circle-packing / enclosure diagram
(see [data-to-viz.com's circular packing reference](https://www.data-to-viz.com/graph/circularpacking.html)).
A single metric sizes the leaf (Level 3) circles; parent circles at every
other level are computed by the packing algorithm itself to just enclose
their children, not independently sized from data.

**This is the first raw-Vega visual in this library**, not Vega-Lite like
the aster plot and waffle chart — see "Why raw Vega" below before touching
this spec's structure, and `docs/patterns.md`'s "When to use raw Vega
instead of Vega-Lite" section for the library-wide version of that
reasoning. Shares this library's color/typography conventions
(`docs/design-system.md`) and as many structural conventions as translate
from Vega-Lite to raw Vega (`docs/patterns.md`).

## Why raw Vega

True circle packing needs an actual packing algorithm — fitting circles
without overlap, sizing each parent to enclose its children — which isn't
expressible in Vega-Lite's row-wise `calculate`/`window` transforms the
way the aster plot's wedge angles or the waffle chart's grid cells were.
Vega (the lower-level engine Vega-Lite itself compiles down to, which
Deneb also exposes directly via a "Provider: Vega" option in Settings) has
native `stratify` (flat rows → tree) and `pack` (tree → circle layout)
transforms built for exactly this. Nothing else in this library needs raw
Vega — the aster plot and waffle chart stay on Vega-Lite, and should.

## Fields expected

| Spec field name | Role | Type | Notes |
|---|---|---|---|
| `Level1`/`Level2`/`Level3` | The 3 hierarchy levels, outermost to innermost | Text columns | Each is its own field-parameter-driven role — see "Field mapping" |
| `Metric` | Leaf circle size | Measure | Only Level 3 (leaf) circles are sized directly from this — Level 1/2 circles are pack-computed to enclose their children |
| `Level1ColorOverrideFill`/`Level2ColorOverrideFill`/`Level3ColorOverrideFill` *(optional)* | Per-category fill color, one per level | Text columns (hex) | Falls back to that level's own `level{N}DefaultFillColor` param when not bound — see "Color" |
| `Level1ColorOverrideBorder`/`Level2ColorOverrideBorder`/`Level3ColorOverrideBorder` *(optional)* | Per-category stroke color, one per level | Text columns (hex) | Falls back to that level's own `level{N}DefaultStrokeColor` param when not bound — see "Color". Aliased internally to `Level{N}StrokeOverride`, per `docs/design-system.md`'s naming convention |

`Level1`, `Level2`, `Level3`, and `Metric` are required; the 6 color
override fields are all optional and degrade gracefully. `Measure
A__highlight` isn't a field you bind — Deneb adds it automatically once
**Expose cross-highlight values for measures** is on in Settings; see
"Cross-filtering and cross-highlighting" below.

**No live report bindings confirmed yet** — this is a brand-new visual.
Unlike the aster plot's `MappedParameter` or the waffle chart's
`MappedParameterTwo`, no field parameter tables exist yet for Level 1/2/3.
See "Field mapping" for the assumed shape (reusing this library's shared 4
candidate columns) and the `dax/` templates for illustrative, not yet
bound, DAX measures.

## Field mapping

Three independent field-parameter-driven levels need a **3-way exclusion
cascade**, extending the waffle chart's proven 2-way `CategorySource`
pattern (see that visual's own README's "Field mapping" section for the
full background on why a field parameter's resolved value can't be
statically named, and why two-or-more field parameters sharing the same
candidate columns need to explicitly exclude whatever a higher-priority
level already claimed).

The adapter (in the `mapped` dataset, `visuals/circle-packing/spec/circle-packing.vg.json`)
checks the same 4 candidates (`EngagementCodeList`/`AlphaCode`/
`CustodianDisplayName`/`EFileName`) for all three levels, in the same
priority order, each level additionally excluding whatever `Level1Source`/
`Level2Source` already claimed:

```json
{ "type": "formula", "as": "Level1Source", "expr": "isValid(datum.EngagementCodeList) ? 'EngagementCodeList' : ... : ''" },
{ "type": "formula", "as": "Level1", "expr": "isValid(datum.EngagementCodeList) ? datum.EngagementCodeList : ... : '(Unspecified)'" },
{ "type": "formula", "as": "Level2Source", "expr": "(isValid(datum.EngagementCodeList) && datum.Level1Source !== 'EngagementCodeList') ? 'EngagementCodeList' : ... : ''" },
{ "type": "formula", "as": "Level2", "expr": "... : 'All'" },
{ "type": "formula", "as": "Level3", "expr": "(... && datum.Level1Source !== '...' && datum.Level2Source !== '...') ? ... : 'All'" }
```

**`Level2`/`Level3` fall back to `'All'`, not `'(Unspecified)'`** — a
deliberate difference from `Level1` and from the aster/waffle chart's own
single-level `Category` field. If a level's own field parameter isn't
bound at all (not just unbound for one row — genuinely not wired up),
every row resolves to the same `'All'` value at that level, collapsing it
to a single pass-through wrapper circle rather than showing a raw
`"undefined"`-style placeholder. This means the visual degrades gracefully
from 3 levels down to 2 or 1 simply by leaving a level's field parameter
unbound — no spec change needed, matching the waffle chart's own
`SeriesCategory` → `'All'` degradation.

**Raw Vega's `formula` transform is Vega-Lite's `calculate`, same
expression language** — this cascade is a direct syntax translation of the
waffle chart's own 2-way version, just extended to 3 levels and written as
`{"type": "formula", "expr": "...", "as": "..."}` instead of
`{"calculate": "...", "as": "..."}`.

`Level{N}ColorOverrideFill`/`Level{N}ColorOverrideBorder` are plain
renames (`Level{N}FillOverride`/`Level{N}StrokeOverride`) — no
field-parameter mechanics, same as the other visuals' single-level color
override fields.

**IDs for the hierarchy are built from these resolved values**, prefixed
per level to avoid any cross-level collision even if two different levels
happen to share a literal string value:
```json
{ "type": "formula", "as": "L1Id", "expr": "'L1:' + datum.Level1" },
{ "type": "formula", "as": "L2Id", "expr": "datum.L1Id + '|L2:' + datum.Level2" },
{ "type": "formula", "as": "L3Id", "expr": "datum.L2Id + '|L3:' + datum.Level3" }
```
**Assumption, not yet stress-tested**: category values don't contain the
`|` separator character. A category value containing a literal `|` would
corrupt the id-prefix scheme this depends on — worth keeping in mind if a
future bound field could contain one.

## Hierarchy construction (the part with no Vega-Lite equivalent)

Three independent `aggregate` datasets — `level1Nodes` (`groupby:
["L1Id"]`), `level2Nodes` (`groupby: ["L2Id", "L1Id"]`), `leafNodes`
(`groupby: ["L3Id", "L2Id"]`) — each produce one row per distinct
combination up to that level, with an `Id`/`ParentId` pair, that level's
own `FillOverride`/`StrokeOverride` (`min` — expected constant per
category, same convention as the other visuals), a `Value` (`sum` of
`Metric`), and a `HighlightValue` (`sum` of `MetricHighlight`) —
**computing `Value`/`HighlightValue` independently at all 3 grouping
levels this way means every node's own subtree-highlight state is already
correct with no post-pack tree walk needed** (see "Cross-filtering and
cross-highlighting" below).

A synthetic `rootNode` dataset (a single inline row, `{"Id": "root",
"ParentId": null}`, no `source`/`transform`) gives `stratify`/`pack` the
single common root they need when there's more than one Level 1 value.
Never rendered (`packedNodes` filters out `depth === 0`).

`nodes` unions all four (`"source": ["rootNode", "level1Nodes",
"level2Nodes", "leafNodes"]`) — **Vega's `data.source` accepts an array
specifically to union same-shaped datasets together**, confirmed against
Vega's own data-spec documentation (`String|String[]` — "If array-valued,
specifies a collection of data source names that should be merged
(unioned) together"), not assumed. Then:

- `stratify` (`key: "Id"`, `parentKey: "ParentId"`) turns the flat unioned
  rows into a real tree.
- `pack` (`field: "Value"`, `size: [width * packArea, height * packArea]`,
  `padding: packPadding`) computes `x`/`y`/`r`/`depth` for every node.
  Leaf radii derive from `Value`; parent radii are the algorithm's own
  enclosing-circle computation — any `Value` already present on a
  level1Nodes/level2Nodes row (there from the aggregate above, coincidentally
  correct since it's the same sum either way) is not what actually drives
  parent sizing; `pack` re-derives it bottom-up from descendants
  regardless, per its own documented behavior ("sums accumulate across
  descendants as the value property").

## Sizing & color

- `packArea` (`1`) — fraction of available space the pack occupies,
  mirroring the waffle chart's `gridArea`; values under `1` shrink the
  pack and center it within the full frame (`packOffsetX`/`packOffsetY`).
- `packPadding` (`3`) — passed straight to the `pack` transform's own
  `padding` parameter (spacing between sibling circles).
- `strokeWidth` (`1`) — shared across all 3 levels. Only fill/stroke
  *color* is independently overridable per level, per the original
  request; a separate stroke width per level wasn't asked for.
- `fillStrokeGap` (`0`) — the same two-layer outline+inset-fill technique
  as the other two visuals (see "Implementation notes"), shared across all
  levels.
- Per-level fill defaults form a visible depth progression even with zero
  DAX bound: `level1DefaultFillColor` (`#2D3B68`, darkest),
  `level2DefaultFillColor` (`#4555D6`, the library's shared default fill
  accent — see `docs/design-system.md`), `level3DefaultFillColor`
  (`#8C97E8`, lightest).
- Per-level stroke defaults all default to `#182231` (the shared
  background-matching convention) — the ask was for *independent override
  capability* per level, not necessarily different *default* stroke
  colors.
- `FillColor`/`StrokeColor` resolve the standard way per node, using
  whichever level's params apply to that node's own `LevelIndex`:
  `isValid(datum.FillOverride) ? datum.FillOverride :
  (datum.LevelIndex === 1 ? level1DefaultFillColor : datum.LevelIndex
  === 2 ? level2DefaultFillColor : level3DefaultFillColor)` (and the
  stroke equivalent).

## Cross-filtering and cross-highlighting

Same two distinct mechanisms as the waffle chart (see that visual's
README section of the same name for the full background) — click-to-filter
*out* of this visual is presumed to hit the same confirmed Deneb-level
limitation (Simple cross-filtering can't trace a mark back to real dataset
rows once the transform pipeline reshapes row counts, which this spec's
`aggregate`+`stratify`+`pack` chain does even more heavily than the
waffle chart's), though **not independently re-verified for this specific
spec** — treat as a known limitation by strong analogy, not a fresh
confirmed finding.

Cross-highlighting *in* is implemented and follows naturally from how the
hierarchy is built: because `HighlightValue` is aggregated independently
at all 3 levels (see "Hierarchy construction" above), every node already
knows whether it or any of its descendants are part of the active
cross-highlight (`IsHighlighted = HighlightValue > 0`) — no post-pack tree
traversal needed, unlike click-select's ancestor/descendant test below.

## Interactivity

**Click-to-select** (`packSelect` — a Vega *signal*, not a Vega-Lite
`param`; raw Vega manages interactivity via signals + event streams
instead of the `params`/`select` shorthand): clicking a node highlights
**that node, its full descendant subtree, and its ancestor chain**
(breadcrumb-style) — dims everything else. Clicking the same node again
clears the selection. Implemented via plain string-prefix tests against
each node's own `Id` (already built as a `|`-joined path,
`L1:Region A|L2:Team 1|L3:Alice`), avoiding any tree traversal at encode
time:
```
!packSelect
  || datum.Id === packSelect
  || indexof(datum.Id, packSelect + '|') === 0       // datum is a descendant of the selection
  || indexof(packSelect, datum.Id + '|') === 0        // datum is an ancestor of the selection
```
**This subtree/ancestor-highlighting behavior is a first-pass default, not
locked in** — same status as every other first interaction default in this
project's history (e.g. the waffle chart's row-break capacity formula).
Worth reacting to once it's visible live: an alternative would highlight
only the exact clicked node, dimming even its own ancestors and
descendants.

**Tooltip**: built per-node from an object expression (works uniformly
across levels since `LevelIndex`/`Name`/`Value`/`PercentOfParent` are
already normalized fields by the time `pack` runs):
```
{'Level': ..., 'Category': datum.Name, 'Value': format(datum.Value, '~s'), '% of parent': format(datum.PercentOfParent, '.1%')}
```
`PercentOfParent` is computed via a `joinaggregate` *before* `pack` runs
(grouped by the parent's own id at each level), not by reading
`datum.parent.value` after packing — deliberately avoiding any dependence
on whether Vega's post-`stratify` tree nodes expose usable `.parent`
references in mark-encode expressions, which wasn't worth the risk to rely
on when a pre-computed field does the same job more predictably.

## Labels

No external legend (see the design conversation that shaped this visual —
at 3 hierarchy levels with independent per-node coloring, a flat legend
listing every distinct value at every level could run far longer than a
flat category list ever would for the aster plot or waffle chart).
Categories are identified via **per-level in-circle labels**, each level
independently configurable:

- `level{N}ShowLabel` (bool) — visibility. Level 3 (leaf) defaults to
  `false` — with potentially dozens of small leaf circles, labeling every
  one by default would likely read as clutter; Level 1/2 default to `true`.
- `level{N}LabelPosition` (`"center"` | `"top"` | `"outside"`) — location
  relative to the node's own circle, extending the aster plot's
  `labelPosition: "outside"/"inside"` concept with a `"center"` default
  suited to circle packing's usual style (Level 1 defaults to `"outside"`,
  Level 2/3 to `"center"`).
- `level{N}LabelPath` (`"straight"` | `"radial"`) — `"straight"` (the
  default for all 3 levels) is plain unrotated text, the common
  circle-packing convention. `"radial"` rotates the label to the angle
  from the pack's center through that node's position, reusing the aster
  plot's own leader-line angle math (`atan2`-based) rather than
  reinventing it — see "Implementation notes" for the readability fix this
  needed.
- `level{N}LabelMinRadius` (px, `0`/`14`/`10` for levels 1/2/3) — hides a
  label on circles too small to hold legible text, mirroring the aster
  plot's `labelMinAngleDegrees`.
- Shared across all levels: `labelFontFamily` (`"Segoe UI"`),
  `labelFontSize` (`11`pt, converted via the established `* 4/3`
  pattern), `labelColor` (title tier, `#FCFCFD`, used for Level 1),
  `labelSubtitleColor` (subtitle tier, `#A5B4CB`, used for Level 2/3) —
  kept shared rather than per-level-configurable to bound scope; flag if
  per-level fonts/sizes turn out to matter in practice.

Three separate `text` marks (one per level, each reading from its own
pre-filtered dataset — `level1NodeLabels`/`level2NodeLabels`/
`level3NodeLabels`), rather than one mark with heavy per-datum branching —
matches this library's general preference for a few straightforward
layers over one clever conditional one.

## Setup in Deneb

1. In Deneb's **Settings** tab, switch **Provider** from Vega-Lite to
   **Vega** — this spec will not run under the Vega-Lite provider.
2. Add a Deneb visual, drop `Level1`, `Level2`, `Level3`, and `Metric`
   into the Values well at minimum (the 6 color override fields as
   needed — see "Fields expected").
3. Paste `spec/circle-packing.vg.json` into the **Specification** tab.
4. Tune the `signals` block (see reference table below) — raw Vega's
   direct equivalent of Vega-Lite's `params`, edited the same way
   (hand-edited JSON, not exposed as Format-pane controls).

## Params reference

Raw Vega has no `params` array — every one of these lives in the spec's
top-level `signals` array instead, each with a `"name"`/`"value"` pair
(or `"update"` for a derived one), the direct equivalent of a Vega-Lite
`param`.

| Signal | Default | Purpose |
|---|---|---|
| `packArea` | `1` | Fraction of available space the pack occupies |
| `packPadding` | `3` | Spacing between sibling circles, in pixels |
| `strokeWidth` | `1` | Circle stroke thickness, in pixels, shared across all levels |
| `fillStrokeGap` | `0` | Pixel gap between a circle's fill and its own stroke outline, shared across all levels |
| `level1DefaultFillColor` / `level2DefaultFillColor` / `level3DefaultFillColor` | `#2D3B68` / `#4555D6` / `#8C97E8` | Fallback fill color per level when that level's override field isn't bound/has no value |
| `level1DefaultStrokeColor` / `level2DefaultStrokeColor` / `level3DefaultStrokeColor` | `#182231` (all three) | Fallback stroke color per level |
| `level{N}ShowLabel` | `true`/`true`/`false` | Per-level label visibility |
| `level{N}LabelPosition` | `"outside"`/`"center"`/`"center"` | Per-level label location |
| `level{N}LabelPath` | `"straight"` (all three) | Per-level label rotation style |
| `level{N}LabelMinRadius` | `0`/`14`/`10` | Per-level minimum circle radius (px) to show a label at all |
| `labelFontFamily` | `"Segoe UI"` | Shared label font |
| `labelFontSize` | `11` | Shared label font size, in points |
| `labelColor` | `#FCFCFD` | Level 1 label color (title tier) |
| `labelSubtitleColor` | `#A5B4CB` | Level 2/3 label color (subtitle tier) |

A few other signals (`labelFontSizePx`, `packOffsetX`/`packOffsetY`,
`packCenterX`/`packCenterY`) are `update`-computed from the ones above —
don't hand-edit their values directly.

## Implementation notes / gotchas

- **A raw Vega symbol mark's `size` for shape `"circle"` is the area of
  its *bounding square* (`(2r)²`), not the circle's own area (`πr²`) —
  confirmed empirically, not assumed from the Vega-Lite convention this
  library had already established.** The aster plot's and waffle chart's
  README both document `size = π × diameter² / 4` (equivalent to `π × r²`)
  as the correct Vega-Lite point-mark convention — carrying that exact
  formula over to a raw Vega `symbol` mark produced circles rendered at
  `√π⁄2 ≈ 0.886×` the intended radius (confirmed by rendering a single
  circle with a known target radius and measuring the actual SVG path's
  radius directly, then testing the corrected formula the same way).
  **Fixed: `size = 4 × r²` for raw Vega symbol/circle marks.** This is a
  Vega-Lite-vs-raw-Vega discrepancy, not a Vega version issue — Vega-Lite
  evidently applies its own conversion when compiling a point/circle mark
  down to the underlying symbol mark, which a spec written directly
  against raw Vega doesn't get for free.
- **A mark's `"from"` property does not support an inline `"transform"`
  array the way one might expect from Vega-Lite's per-layer transforms —
  confirmed by a real bug, not just an assumption.** First attempt
  filtered the root node out of the circle/label marks via `"from": {
  "data": "nodes", "transform": [{"type": "filter", ...}] }`; validation
  and rendering both succeeded with no error, but the *root node rendered
  anyway* — a large, unintended background circle plus a stray "root"
  label, both centered on the pack's own center. Traced by dumping the
  actual `nodes` dataset directly (confirming `depth`/`LevelIndex` were
  computed correctly) and then inspecting the compiled SVG's raw `<path>`
  elements (finding a circle at the pack's exact center coordinates that
  shouldn't have existed) before concluding the inline `from.transform`
  was silently not being applied at all. **Fixed by creating real
  top-level filtered datasets instead** (`packedNodes`, and
  `level{N}NodeLabels` for each label mark) and pointing every mark's
  `"from": {"data": ...}` at one of those, with no inline transform.
- **Event stream selectors for a named mark are `"@markName:eventType"`
  — no trailing `!`.** First attempt used `"@fillMark:click!"` (guessing
  the `!` meant something like "consume this event"); the signal simply
  never updated on click, with no compile error. Confirmed the correct
  form directly against Vega's own event-stream documentation before
  retesting — `"@fillMark:click"` fixed it immediately, verified via
  `view.signal('packSelect')` after a simulated click, not just a visual
  screenshot diff (the visual diff alone had already passed on a bad
  coordinate — see the next point — so a second, independent bug was
  briefly masked by the first).
- **Text labels drawn on top of a circle intercept clicks meant for that
  circle underneath, unless explicitly marked non-interactive.** Found
  because a click at a coordinate landing exactly on a "Team 2"-style
  label (dead center of its circle, where a `"center"`-position label
  naturally sits) never registered a selection, while the *identical*
  click coordinate on a circle with a `"top"`/`"outside"` label (not
  covering the click point) worked correctly — isolated by testing a grid
  of coordinates and comparing which ones updated `packSelect`. Fixed by
  setting `"interactive": false` on all 3 label text marks, letting clicks
  pass through to the fill mark beneath. Worth remembering for any future
  mark that's purely decorative and layered on top of an interactive one.
- **Radial labels need a 180° flip on the far side of the pack to stay
  readable, not just a direct angle-to-rotation mapping.** A first version
  rotated each label directly by its polar angle from the pack center
  (`atan2(...) * 180/π`); this produced correctly-angled but frequently
  *upside-down* text for any node on the "far" side of the pack (roughly
  the left half, confirmed by rendering and visually finding "Region C"
  and "Team 3" inverted while "Region A"/"Team 1" read fine) — the classic
  radial-text problem also seen in sunburst/radial-tree charts. Fixed with
  the standard technique: flip the rotation by 180° whenever the raw angle
  magnitude exceeds 90° (`RadialLabelAngleDeg = angleDeg + (abs(angleDeg)
  > 90 ? 180 : 0)`), computed once as a shared data field rather than
  duplicated per label mark. Re-verified by rendering all 3 levels with
  `"radial"` paths active and confirming every label reads left-to-right,
  not upside down, at every position around the pack.
- **`PercentOfParent` is pre-computed via `joinaggregate` before `pack`
  runs, not read from a post-pack `datum.parent.value` reference** — a
  deliberate choice to avoid depending on whether Vega's `stratify`/`pack`
  transforms expose usable `.parent` object references inside mark-encode
  expressions (plausible, based on how d3-hierarchy's own node objects
  work, but not confirmed for Vega specifically, and not worth the risk
  when a `joinaggregate` grouped by each node's own parent id gives the
  identical number with no such dependency).

## Known limitations / roadmap

- **Click-to-filter-out is presumed broken by the same Deneb-level
  limitation as the waffle chart, but not independently re-verified for
  this spec** — see "Cross-filtering and cross-highlighting" above.
- **No small-multiples split** — confirmed out of scope for this version;
  one pack per visual. Could be added later the same way the waffle chart
  has `SeriesCategory`, if needed.
- **The `|` id-separator assumption is untested against real category
  values** — see "Field mapping" above.
- **Nothing here has been verified against a live Power BI report** — no
  field parameters exist yet for Level 1/2/3, the `dax/` files are
  illustrative templates, and Deneb's raw-Vega "Provider: Vega" mode
  itself has not been exercised outside this repo's local Node/Playwright
  tooling. Every claim above is "verified locally," not "confirmed live,"
  per this project's standing distinction.
- **Per-level label font/size isn't configurable**, only shared across all
  3 levels — a deliberate scope decision (see "Labels" above), open to
  revisiting if it turns out to matter.
