# Aster Plot

A radial chart where each wedge's **angle** encodes a category's share of a
whole (like a pie chart), and each wedge's **radius** independently encodes a
second measure (e.g. a score or performance value) — so category size and
category performance are both visible at a glance. Also known as a
Nightingale/Coxcomb chart with a per-wedge radius override.

Click a wedge to swap the center label from the dataset-wide summary to that
wedge's own detail (actual value + % of grand total for both measures).

Migrated from a D3 v3 reference implementation using `d3.layout.pie()` for
the angle and `d3.svg.arc()` with a per-datum `outerRadius` function for the
radius:

```js
var pie = d3.layout.pie().sort(null).value(d => d.weight);
var arc = d3.svg.arc()
  .innerRadius(innerRadius)
  .outerRadius(d => (radius - innerRadius) * (d.data.score / 100.0) + innerRadius);
```

## Fields expected

| Spec field name  | Role                       | Type         | Notes                                     |
|-------------------|---------------------------|--------------|---------------------------------------------|
| `Category`        | Category                  | Text column  | One wedge per distinct value                 |
| `MeasureA`         | Wedge angle/share          | Measure      | Wedge angle = `MeasureA / sum(MeasureA)` of 360° |
| `MeasureB`         | Wedge radius                | Measure      | Scaled from 0 to the wedge's max radius       |
| `ColorOverride`    | Per-category wedge color    | Text column (hex, e.g. `#4c78a8`) | Drives wedge color directly — see "Color" below |
| `LabelColorOverride` *(optional)* | Per-category wedge **label** color | Text column (hex, e.g. `#4c78a8`) | Falls back to `labelColorOutside`/`labelColorInside` when not bound — see "Color" below |

Rename your Power BI fields to match, or use Deneb's field-mapping/dataset
editor to alias them to these names. **Fields must be numeric where expected**
— `MeasureA`/`MeasureB` need to actually be numeric measures/columns in the
data model; a text-typed field bound there will silently produce `NaN`
everywhere it's summed, with no error (a real issue hit during development —
see the root cause below).

**This report's concrete bindings** (verified against the `Project_snapshot_Fact`
table structure and the `MappedParameter` field parameter definition shared
so far):

| Spec field name | Actual Power BI field | How it gets there |
|---|---|---|
| `Category` | The `Category` column on the `MappedParameter` table (a field parameter over `Project_snapshot_Fact`'s `EngagementCodeList`/`AlphaCode`/`CustodianDisplayName`/`EFileName`) | Drop `Category` (from the `MappedParameter` table) into the Values well as-is; **do not** rename it (see "Field mapping"). |
| `MeasureA` | `[Data Recieved GB]` | Drop into Values well, then right-click → **Rename for this visual** → `MeasureA`. |
| `MeasureB` | `[Count Item]` | Drop into Values well, then right-click → **Rename for this visual** → `MeasureB`. |
| `ColorOverride` | `[Category Color Gradient]` (see `dax/measure-a-color-gradient.dax`) | Drop into Values well, then right-click → **Rename for this visual** → `ColorOverride`. |

All 4 happen to need no spec change today — the `Category` column already
happens to be named exactly `Category` in the model (see "Field mapping"
for why that's not a coincidence), and `MeasureA`/`MeasureB`/`ColorOverride`
rename freely per-visual regardless.

## Field mapping

The spec's `transform` array opens with 5 `calculate` steps — a field-mapping
adapter, isolating every place a Power-BI-side rename could ever require a
spec change down to these 5 lines:

```json
{
  "calculate": "isValid(datum.EngagementCodeList) ? datum.EngagementCodeList : isValid(datum.AlphaCode) ? datum.AlphaCode : isValid(datum.CustodianDisplayName) ? datum.CustodianDisplayName : isValid(datum.EFileName) ? datum.EFileName : '(Unspecified)'",
  "as": "Category"
},
{ "calculate": "datum.MeasureA", "as": "MeasureA" },
{ "calculate": "datum.MeasureB", "as": "MeasureB" },
{ "calculate": "datum.ColorOverride", "as": "ColorOverride" },
{ "calculate": "datum.LabelColorOverride", "as": "LabelColorOverride" }
```

The `Category` line's final fallback (`'(Unspecified)'`) covers rows where
*none* of the 4 candidate fields have a value — this showed up live as a raw
`"undefined"` legend entry/wedge once real production data included such
rows (some records apparently have no value for any of the 4 mapped
columns); it now renders as a legible placeholder label instead.

`MeasureA`/`MeasureB`/`ColorOverride`/`LabelColorOverride` are no-ops (each external field
already matches the internal canonical name), but every line exists as a
deliberate seam: every other calculation, mark, and encoding in the spec
references the *internal* canonical names (`Category`, `MeasureA`, etc.),
never a raw incoming field directly. Vega-Lite field references are
static strings baked in at edit time — there's no way to make `"field"`
itself dynamic/config-driven (we looked; Config can't do it either, since
it's a fixed-schema style object, not a general parameter store) — so
this adapter block is the practical equivalent: the *one* place to touch,
instead of hunting down every occurrence of a field name scattered across
the spec.

**`Category`'s line is doing real, confirmed-necessary work, and got
misdiagnosed twice before landing here.** First assumption: a field
parameter's label/value column cannot be renamed per-visual, so whatever
it's named in the model is what Deneb sees — true, but incomplete. Second
assumption, also wrong: since the `MappedParameter` table's label column
is itself named `Category` (confirmed via the Fields pane), no adapter
change should be needed. Confirmed live (via Deneb's own "view as table"
and a plain Power BI table visual, both showing the same raw column
header): when Power BI resolves a field-parameter-driven role, the data
Deneb actually receives is keyed by the **real underlying column's own
name** — whichever of `EngagementCodeList`/`AlphaCode`/
`CustodianDisplayName`/`EFileName` is currently selected — not by the
field parameter's own label column name at all. Confirmed by the visual
rendering a single `"undefined"` category for every row (i.e. the legend
collapsing to one value, and the center label reading "1 categories")
until this was fixed.

Since that real key changes with the live slicer selection, no single
static line can cover it — but since there are only 4 known candidates
(matching `dax/category-field-parameter.dax`'s definition), the adapter
enumerates them the same way the DAX `SWITCH` does: check each possible
field name in order via `isValid()`, and use whichever one actually has
data. This framework assumes **one** such field parameter per report.
`MeasureA`/`MeasureB` don't have this dynamic-naming problem (ordinary
measures rename freely to a fixed name), so they don't need an equivalent
enumeration.

**If a second, independent Deneb visual needs its own swappable Category**
(different candidate fields, swapping independently of the first): field
parameters must be uniquely named model-wide, so that second visual needs
its own spec copy. The update is contained entirely to the adapter's
first `calculate` — swap in that field parameter's own candidate list
(the same `isValid()` cascade pattern, just with different field names):
```json
{
  "calculate": "isValid(datum.SomeColumn) ? datum.SomeColumn : isValid(datum.AnotherColumn) ? datum.AnotherColumn : datum.LastCandidateColumn",
  "as": "Category"
}
```
Nothing else in the spec changes — everything downstream already reads
the internal canonical name `Category`.

## Color

Deneb doesn't let a spec add new native Format-pane cards (a per-category
"Data colors" swatch picker with paint-bucket icons, like a bar/pie chart
gets) — that UI comes from a visual's `capabilities.json`, which is fixed by
Deneb's own developer. The arc mark's `color` encoding uses
`"scale": {"range": {"field": "ColorOverride"}}` — each category's color is
pulled directly from its own row's `ColorOverride` value, computed however
you like via a DAX measure.

**Calculated gradient (magnitude-based coloring):**
[`dax/measure-a-color-gradient.dax`](dax/measure-a-color-gradient.dax) is a
DAX measure that linearly interpolates a hex color between `#263449` (at
the lowest **category-level total** of `MeasureA`) and `#4555D6` (at the
highest) — categories with a bigger `MeasureA` render in a brighter blue.
Min/max are computed over a category-level summary table
(`ADDCOLUMNS`/`ALLSELECTED` + `MINX`/`MAXX`), not over the underlying fact
table's raw rows — those differ whenever a category spans more than one
row (e.g. a category with rows of 5 and 3 has a category total of 8; the
gradient needs to compare that 8 against other categories' totals, not
against the raw per-row values 5 and 3). It also parses/rebuilds hex
manually (DAX has no native hex/decimal conversion) — replace the
placeholder table/field references with your actual model before using it.

**Pinned/fixed colors per category** instead of a gradient — same field,
different measure:
```dax
Category Colour =
SWITCH (
    SELECTEDVALUE ( 'Table'[Category] ),
    "Awareness", "#4c78a8",
    "Consideration", "#f58518",
    "Acquisition", "#e45756",
    "#999999" // default/fallback
)
```

**Matching the live report theme instead of a calculated color:** Deneb also
ships a built-in `pbiColorNominal` Vega color scheme that automatically
reflects the report's actual theme colors, confirmed from
[deneb.guide's Theme Colors & Schemes docs](https://deneb.guide/docs/1.7/schemes)
(there's also a `pbiColor(index)` expression function, and a "Report Theme
Integration" section in the Format pane with a conditional-formatting-aware
"Discrete Ordinal Colors" property). Swap the arc mark's `color.scale` to
`{"scheme": "pbiColorNominal"}` and drop `ColorOverride` if you want that
instead — no field required.

Local preview tooling (`tools/render-preview.mjs`, `tools/validate.mjs`)
registers a stand-in for `pbiColorNominal` using Power BI's default theme
palette, since that scheme only really exists at runtime inside Deneb/Power
BI — see `tools/deneb-shims.mjs`. Always verify actual colors inside Deneb
in Power BI Desktop, since the real report theme could differ from the
stand-in.

### Title / subtitle / sublabel color hierarchy

Every piece of text in this visual falls into one of two tiers, matching
Power BI's own title/subtitle vocabulary:

- **Title tier** (`#FCFCFD` near-white by default) — the primary, most
  prominent text: the wedge's category name, and the center label's title
  (`"Overview"`, or the clicked category's name).
- **Subtitle tier** (`#A5B4CB` muted blue-gray by default) — secondary,
  supporting text: the wedge's MeasureA/MeasureB/percent line(s) (called
  "sublabels" — a wedge-level subtitle), the center label's secondary
  line(s) (category count/totals, or the selected wedge's stats), and the
  legend (both its title and entries).

Concretely, that's **6 independent color params**, not 2 — there's no
single master "title color"/"subtitle color" switch, each context is its
own param so they can diverge if you want:

| Tier | Param(s) |
|---|---|
| Wedge title (category name) | `labelColorOutside`/`labelColorInside` (or per-category via `LabelColorOverride` — see below) |
| Wedge sublabel (metrics line) | `labelSubtitleColor` |
| Center title (both views) | `centerFontColor` (or per-wedge via `centerSelectedColorFromWedge` — selected view only) |
| Center subtitle (both views) | `centerSubtitleColor` |
| Legend | `legendLabelColor`, `legendTitleColor` |

**Wedge label color** — yes, the *title* tier can be measure-driven, the
same way wedge fill color is: bind the optional `LabelColorOverride` field
(see "Fields expected") to a DAX measure and it drives each wedge's
category-name color per-category (e.g. to match each label to its own
wedge, or flag categories some other way). When it's not bound (or a given
row has no value), it falls back to the static `labelColorOutside`/
`labelColorInside` params, chosen by `labelPosition` (`"outside"` or
`"inside"`). The *sublabel* tier (`labelSubtitleColor`) is always the one
static color — there's no per-category override for it.

**Wedge label font size** auto-scales with the wedge's own angle when
`labelAutoSize` is `true` (`false`/off by default) — narrow wedges get
smaller label text (`labelFontSizeMin`), wide wedges get larger text (up to
`labelFontSizeMax`), scaling linearly as the wedge's angle grows from 0° to
`labelAutoSizeAngleCap`. With `labelAutoSize: false` (the default), both the
title and sublabel lines render at one static size, `labelFontSize`
(**points**, converted to pixels via `labelFontSizePx`). Note that *line
spacing* (`lineHeight`) can't follow per-wedge auto-sizing — Vega-Lite
doesn't allow `lineHeight` as a per-datum encoding (see "Implementation
notes") — so if you do turn `labelAutoSize` back on, multi-line wedge
labels use one uniform spacing regardless of each row's actual auto-scaled
size. That imprecision doesn't apply at all when `labelAutoSize` is off
(the default), since every row then shares the exact same font size anyway.

**Center label color** is different from the wedge title tier — there's no
per-category *field* for the default (no-selection) view, because it's an
aggregate across every category, so there's no single row's color to pull
from. What exists instead:
- `centerFontColor`/`centerSubtitleColor` — static title/subtitle colors,
  used in both the default view and the selected-wedge view, unless
  overridden by the next param.
- `centerSelectedColorFromWedge` (`false` by default) — when `true`, the
  **selected-wedge view's title only** switches to that wedge's own
  `ColorOverride` value instead of `centerFontColor` (the selected view's
  underlying data is that one filtered row, which already carries
  `ColorOverride`, so this is just picking it up rather than a new DAX
  measure). The subtitle line and the default view are unaffected.

**Legend text color** (`legendLabelColor`/`legendTitleColor`) is also a
static pair of params, not measure-driven — Vega-Lite legends don't support
per-entry text color, only per-entry *symbol* color (which already comes
from `ColorOverride` via the color scale).

**Dark-canvas defaults:** the background and wedge `borderColor` both
default to `#182231` (so wedges read as separated by a thin gap matching
the page, not a bright stroke), and every title-tier param defaults to
`#FCFCFD` / every subtitle-tier param to `#A5B4CB` — matching a dark report
canvas/theme (like the one this visual is actually deployed on). If you
ever use this spec on a **light** canvas instead, flip the background/
border to a light color and the text tiers to dark ones — `labelColorInside`
stays `#ffffff` either way, since inside labels sit on a colored wedge
fill, not the page background.

## Legend position and chart centering

`showLegend` and `legendOrient` control the legend together, and — because
of how Vega-Lite lays out legends — the same two params also double as the
"center the aster on the whole visual frame vs. center it in the remaining
plot area" toggle the cosmetics pass asked for:

- **`legendOrient` set to an edge value** (`"left"`/`"right"`/`"top"`/
  `"bottom"`) reserves dedicated layout space for the legend along that
  edge, and the donut centers itself within whatever plot area is *left
  over* — this is Vega-Lite's default legend behavior, and is why a legend
  with many entries visibly pushes the donut off-center within the overall
  visual frame (confirmed live with a 10-category TopN view).
- **`legendOrient` set to a corner value** (`"top-left"`/`"top-right"`/
  `"bottom-left"`/`"bottom-right"`) renders the legend as an overlay *inside*
  the plot area instead — confirmed by rendering both ways and checking the
  compiled view's `width`/`height` signals plus the actual arc positions:
  corner orients reserve zero layout space, so the donut centers on the
  **entire** visual frame regardless of how many legend entries there are.
  The tradeoff is the legend can visually overlap whichever wedge/label
  happens to fall in that corner.
- **`showLegend: false`** hides the legend entirely (and, via
  `legendOrientEff` forcing `orient` to `"none"`, reserves no space either)
  — the donut always centers on the full frame in this case. Implemented by
  zeroing `symbolSize`/`labelFontSize`/`titleFontSize` on the legend rather
  than an opacity `encode` override — see "Implementation notes" for why.

Pick edge-orient if you want a conventional, always-fully-visible legend
and don't mind the donut shifting slightly off the frame's true center;
pick a corner orient (or hide the legend) if you want the donut always
dead-centered in the frame and can tolerate an overlay.

## Setup in Deneb

Deneb's visual editor has three separate tabs — worth knowing which is which,
confirmed from
[deneb.guide's Visual Editor docs](https://deneb.guide/docs/1.0/visual-editor):

- **Specification** — the actual Vega-Lite spec (everything in
  `spec/aster-plot.vl.json`, including our `params`).
- **Config** — a separate Vega config JSON, intended for cosmetic/theming
  properties, kept apart from the spec on purpose ("the intention here is to
  separate the cosmetic aspects of the visual away from its logical ones").
  This spec deliberately has **no top-level `config` key** so it never
  conflicts with whatever you have in this tab. Use
  [`config/base-config.json`](../../config/base-config.json) at the repo
  root as a starting point if you want one — it's Deneb's own bundled
  default, corrected for a **light** report canvas (the bundled default's
  `#CCD5E3` text/legend colors are nearly invisible on white). If your
  report canvas is dark instead (as the actual dashboard this visual is
  deployed on is), that light-canvas correction is the wrong direction —
  either skip the Config tab's text-color settings entirely (see below for
  why they mostly don't matter here) or write a dark-appropriate version.
- **Settings** — Deneb's own fixed Format-pane controls, under
  **Vega > Power BI Interactivity**: a Tooltip Handler toggle, a
  **Cross-Filtering (Selection) of Data Points** toggle, and a
  **Resolve Data Points in Context Menu** toggle.

**Does the Config tab still do anything for this spec?** Only for the few
properties this spec doesn't already hardcode. In Vega's cascade, an
explicit mark/encoding value in the Specification *always* wins over a
Config default for that same property — Config only fills in gaps. As of
this version, this spec explicitly sets almost everything Config could
otherwise control: view background (top-level `"background"` param, always
wins over Config's `view.fill`), wedge fill/stroke, both marks' font
families (`labelFontFamily`/`centerFontFamily`), all title/subtitle/legend
text colors and sizes (see "Color"), even the legend's `symbolSize`/font
sizes. **The one real gap left:** the wedge-label marks' font *weight*
isn't set by any param (only the center-label marks set `fontWeight`) — it
still falls through to whatever Config's `text` mark config specifies (or
Vega's own plain "normal" default if Config doesn't set it either). If you
want the wedge label weight controlled regardless of the Config tab's
contents, that would need a `labelFontWeight` param the same way the center
label already has `centerFontWeight` — ask if you want that added.

None of our `params` (in the Specification) can collide with anything in
Config — Vega-Lite's config schema (`view`, `axis`, `legend`, mark-type
defaults, etc.) has no `params` concept at all; the two are entirely
separate parts of the spec's JSON.

Steps:

1. Add a Deneb visual, drop `Category`, `MeasureA`, `MeasureB` in as above
   (`ColorOverride` only if you're using the pinned-color option — see
   "Color" above).
2. Paste `spec/aster-plot.vl.json` into the **Specification** tab.
3. The chart auto-sizes to the visual's container and recomputes the
   inner/outer radius from the container size on every resize
   (`innerRadius = 0.35 * outerRadius`, matching the D3 original's
   `radius * 0.35`).
4. Tune the `params` block at the top of the spec (see reference table
   below) to control borders, labels, the center summary, and formatting.
   These are edited directly in the JSON — Deneb doesn't expose them as
   Format-pane controls (see "Color" above for why).
5. Worth trying: enabling **Cross-Filtering (Selection) of Data Points** in
   the Settings tab may bridge our existing `wedgeSelect` click-to-select
   param straight to Power BI's native cross-filtering, with no spec changes
   — this hasn't been verified live (no Power BI available in this
   environment), so treat it as a promising next thing to test rather than a
   confirmed behavior.

## Params reference

All adjustable settings live in the spec's top-level `params` array, edited
directly in the JSON (see "Setup in Deneb" for why). The table below is
every param meant to be hand-edited. A handful of *other* params also exist
in that same array (`outerRadius`, `innerRadius`, `labelFontSizePx`,
`labelLineHeight`, `legendOrientEff`, `legendXEff`, `legendYEff`,
`centerFontSizePx`, `centerTitleWrapped`, `centerTitleWrappedLineCount`,
`centerDefaultTitleLineCount`, `centerDefaultLineCount`,
`centerDefaultFontSize`, `centerDefaultLineHeightEff`,
`centerDefaultTitleDy`, `centerDefaultSubtitleDy`,
`centerSelectedTitleLineCount`, `centerSelectedLineCount`,
`centerSelectedFontSize`, `centerSelectedLineHeightEff`,
`centerSelectedTitleDy`, `centerSelectedSubtitleDy`) — these are all
`expr`-computed from the params below and internal wiring (pt→px
conversion, auto-shrink math, the legend hide/show mechanism, wrapped-line
counting, title/subtitle vertical stacking); don't hand-edit their values
directly, edit the params that feed them instead.

**Font sizes are specified in points** (`labelFontSize`, `centerFontSize`),
matching how Word/PowerPoint/Power BI's own Format pane think about font
size, not raw pixels — a `*_Px` derived param (`labelFontSizePx`/
`centerFontSizePx`) does the pt→px conversion (`pt * 4 / 3`, the standard
96-DPI conversion) before it reaches any actual text-rendering property.
The two auto-scale range params (`labelFontSizeMin`/`labelFontSizeMax`)
are the one exception — they stay in raw pixels, since they only matter
when `labelAutoSize` is switched back on (off by default).

**Wedge borders** (a frequent first customization): `borderColor` and
`borderWidth` control the border drawn between/around every wedge —
there's no separate "border thickness" param beyond `borderWidth`, and no
per-category border color (it's spec-wide, matching the D3 original's
single stroke style).

| Param | Default | Purpose |
|---|---|---|
| `innerRadiusRatio` | `0.35` | Inner hole radius as a fraction of the outer radius |
| `borderColor` | `#182231` | Wedge border/stroke color (the line between adjacent wedges, and around each wedge's outer/inner edge) — defaults to the same value as the background, so wedges read as separated by a thin gap rather than a bright stroke |
| `borderWidth` | `1.5` | Wedge border/stroke thickness, in pixels |
| `measureFormat` | `~s` | [D3 format string](https://d3js.org/d3-format) applied to `MeasureA`/`MeasureB` wherever shown on-chart — `~s` gives automatic unit scaling (1200 → "1.2k", 84000 → "84k") |
| `percentFormat` | `.0%` | D3 format string for % of grand total |
| `showLabelCategory` | `true` | Wedge label: show category name (the "title" line — see "Color") |
| `showLabelMeasureA` | `false` | Wedge label: show MeasureA actual (a "sublabel" line) |
| `showLabelMeasureB` | `false` | Wedge label: show MeasureB actual (a "sublabel" line) |
| `showLabelPercent` | `true` | Wedge label: show MeasureA's % of grand total (a "sublabel" line) |
| `labelFontFamily` | `"Segoe UI"` | Wedge label font family (both the category title line and the metric sublabel line(s)) |
| `labelFontSize` | `12` | Wedge label font size **in points**, when `labelAutoSize` is `false` |
| `labelAutoSize` | `false` | Auto-scale each wedge's label font size by its own angle instead of using a static `labelFontSize` — off by default; see "Color" section for the tradeoff with line spacing |
| `labelFontSizeMin` | `7` | Smallest auto-scaled label font size **in pixels** (narrowest wedges), when `labelAutoSize` is `true` |
| `labelFontSizeMax` | `13` | Largest auto-scaled label font size **in pixels** (widest wedges), when `labelAutoSize` is `true` |
| `labelAutoSizeAngleCap` | `40` | Wedge angle (degrees) at which auto-scaled font size reaches `labelFontSizeMax`; wider wedges are capped at `labelFontSizeMax` |
| `labelPosition` | `"outside"` | Wedge label placement: `"outside"` the wedge or `"inside"` it |
| `labelOutsideOffset` | `14` | Pixels beyond the *chart's* outer radius, when `labelPosition` is `"outside"` — all outside labels sit on one constant ring regardless of each wedge's own radius (see "Implementation notes") |
| `labelInsideRatio` | `0.5` | Fraction of the way from the inner hole to the wedge's own outer edge, when `labelPosition` is `"inside"` (`0` = at the hole, `1` = at the wedge's edge) |
| `labelColorOutside` | `#FCFCFD` | Wedge label **title** (category name) color when `labelPosition` is `"outside"` and `LabelColorOverride` isn't bound/has no value |
| `labelColorInside` | `#ffffff` | Wedge label **title** (category name) color when `labelPosition` is `"inside"` and `LabelColorOverride` isn't bound/has no value |
| `labelSubtitleColor` | `#A5B4CB` | Wedge label **sublabel** color (the MeasureA/MeasureB/percent line(s) below the category name) — always this one static color, not affected by `LabelColorOverride` |
| `labelWrapWidth` | `12` | Wedge label word-wrap width, in characters (wraps at the nearest space at/after this length) |
| `labelMinAngleDegrees` | `8` | Wedge label (and leader line) is hidden entirely for wedges narrower than this angle — the overlap-prevention mechanism; see "Known limitations" |
| `showLeaderLine` | `true` | Show a thin line from the wedge's outer edge to its label (outside labels only) |
| `leaderLineColor` | `#999999` | Leader line color |
| `leaderLineWidth` | `1` | Leader line thickness |
| `leaderLineGap` | `4` | Pixel gap between the leader line's end and the label |
| `showLegend` | `true` | Show/hide the category legend entirely — see "Legend position and chart centering" |
| `legendOrient` | `"right"` | Legend position: `"left"`/`"right"`/`"top"`/`"bottom"` (reserves space, shifts the donut) or `"top-left"`/`"top-right"`/`"bottom-left"`/`"bottom-right"`/`"none"` (overlay, donut stays centered on the full frame) — see "Legend position and chart centering" |
| `legendLabelColor` | `#A5B4CB` | Legend entry text color |
| `legendTitleColor` | `#A5B4CB` | Legend title ("Category") text color |
| `centerTitleText` | `"Overview"` | Static title shown in the center when nothing is selected |
| `centerFontFamily` | `"Segoe UI"` | Center label font family (both the title and subtitle lines, in both views) |
| `centerFontSize` | `14` | Center label font size **in points** (both views), when `centerAutoSize` is `false` |
| `centerAutoSize` | `false` | Auto-shrink the center label's font size as more lines are shown (past 4 lines) instead of holding a static `centerFontSize` — off by default |
| `centerFontWeight` | `bold` | Center label font weight (`normal`, `bold`, or `100`–`900`) — applies to the title line in both views; the subtitle line is never bold, regardless of this param |
| `centerFontColor` | `#FCFCFD` | Center label **title** color, both views — unless `centerSelectedColorFromWedge` overrides it for the selected-wedge view's title |
| `centerSubtitleColor` | `#A5B4CB` | Center label **subtitle** color, both views (the secondary line(s) below the title — category count/totals in the default view, MeasureA/MeasureB/percent in the selected-wedge view) |
| `centerSelectedColorFromWedge` | `false` | When `true`, the selected-wedge center view's **title** uses *that wedge's own* `ColorOverride` value instead of the static `centerFontColor`. No equivalent exists for the default (no-selection) view, since it's an aggregate across every category and there's no single row's color to use. |
| `centerWrapWidth` | `14` | Center label word-wrap width, in characters (applies to the title, and to the selected-wedge view's category name) |
| `showCenterTitle` | `true` | Default (no selection) view: show `centerTitleText` |
| `showCenterCount` | `true` | Default view: show distinct category count |
| `showCenterMeasureAActual` | `true` | Default view: show `sum(MeasureA)` |
| `showCenterMeasureBActual` | `false` | Default view: show `sum(MeasureB)` |
| `showCenterSelCategory` | `true` | Selected-wedge view: show the clicked category's name |
| `showCenterSelMeasureAActual` | `true` | Selected-wedge view: show that wedge's MeasureA |
| `showCenterSelMeasureAPercent` | `true` | Selected-wedge view: show that wedge's % of grand total (MeasureA) |
| `showCenterSelMeasureBActual` | `true` | Selected-wedge view: show that wedge's MeasureB |
| `showCenterSelMeasureBPercent` | `false` | Selected-wedge view: show that wedge's % of grand total (MeasureB) |

## Implementation notes / gotchas

Three real bugs were found and fixed while building this by actually
compiling+running the spec and, for the interactive piece, driving it in a
headless browser — not just by eyeballing the JSON. Worth knowing about
since they're easy to reintroduce:

- **Don't use `"stack": true` on `theta` when `radius` is also a
  quantitative field on an arc mark.** Vega-Lite's auto-stack groups by the
  *other* quantitative channel's field — since every row has a unique
  `MeasureB`, each wedge "stacked" alone from 0, producing wildly
  overlapping wedges instead of a proper pie. Fixed the same way Vega-Lite's
  own Nightingale/Coxcomb chart example does: a `window` transform manually
  computes cumulative `MeasureA_start`/`MeasureA_end`, and the arc mark
  encodes `theta`/`theta2` directly from those.
- **A `condition: {"param": ..., "empty": false}` tests the *current row*
  against the selection (`vlSelectionTest`), not "is anything selected
  globally."** The center-default layer is a single aggregated row with no
  `Category` field, so it can never match the selection tuple — meaning the
  auto-generated per-datum test never fires and the default text never
  hides. Fixed by using a raw `"test": "length(data('wedgeSelect_store')) >
  0"` expression instead, which checks the selection store's size directly
  regardless of the datum's shape.
- **`{"filter": {"param": "wedgeSelect"}}` matches *everything* when the
  selection is empty** (that's the documented Vega-Lite default — an empty
  selection is treated as "no constraint"). Without `"empty": false`, the
  center-selected layer rendered all 7 rows' text stacked on top of each
  other before any wedge was ever clicked. Fixed by adding `"empty": false`
  to the filter.
- **Multi-line text marks (`lineBreak`) don't re-center the whole block —
  only the first line sits on the "middle"-baseline anchor point, and
  subsequent lines stack below it.** A 3-line center label was rendering
  ~23px below the donut's true center as a result (confirmed by inspecting
  the compiled SVG's `translate()`/`dy` values directly, not just
  eyeballing a screenshot). Fixed with an explicit `dy` correction:
  `-(lineCount - 1) * lineHeight / 2`, where `lineCount` is computed from
  the same show/hide params driving the text content, so it stays correct
  as toggles change how many lines are actually shown. Applied to both
  center layers and the wedge label layer.
- `resolve.scale` is set to `"shared"` for `theta`/`radius`/`color` so all
  layers agree on the same scales.
- Tooltip fields keep Power BI's native number formatting (no explicit
  `"format"` in the tooltip encoding); on-chart labels use explicit
  `measureFormat`/`percentFormat` since they're synthetic combined-text
  fields, not raw Power BI fields Deneb could format natively.
- **Word wrap is a regex approximation, not real text measurement** —
  Vega/Vega-Lite have no way to measure actual rendered pixel width from a
  spec expression, so `labelWrapWidth`/`centerWrapWidth` count *characters*,
  not pixels: `replace(text, regexp('(.{1,N})(\\s+|$)', 'g'), '$1\n')`
  breaks at the nearest space at/after every N characters. Good enough in
  practice, but not pixel-perfect across very different font sizes.
- **The `rule` mark (leader lines) doesn't support `theta`/`radius` polar
  encoding at all** — Vega-Lite silently drops it (confirmed by compiling
  and inspecting the warning). The line's endpoints are computed manually
  with trigonometry instead (`x = width/2 + radius * sin(angle)`,
  `y = height/2 - radius * cos(angle)`), verified to match Vega-Lite's own
  internal theta-to-pixel conversion to the exact pixel in an isolated test.
  The `width/2`/`height/2` center offset is easy to miss — arc/text marks
  get it automatically (baked into each mark's own `translate()`), but a
  plain `x`/`y`-encoded mark like `rule` does not, so omitting it silently
  scattered every leader line across the canvas rather than erroring
  (caught by inspecting the raw SVG output, not by looking at a screenshot).
- **`dy` cannot be a per-datum encoding channel in Vega-Lite** (confirmed by
  compiling — it's silently dropped, same as the `rule` polar-encoding
  case). This matters because word-wrapped text has a *variable* line count
  per row, and the vertical-centering `dy` correction depends on exactly
  how many lines are showing. Where the wrapped text is genuinely
  data-independent (the center title, a spec-level param), the line count
  is computed as a **param expression** instead and centering stays exact.
  Where it's data-dependent (a wrapped category name), see the
  known-limitation note below.
- **Splitting the wedge label into title + sublabel marks initially
  reintroduced the exact overlap this whole `dy` limitation causes** — with
  two separate marks each needing their own vertical offset, the natural
  approach (give each mark a `dy` computed from an *assumed* 1-line title)
  visibly collided the sublabel into the title's second line for any
  wrapped 2-line category name (every "Placeholder X"-style name in the
  synthetic test data, confirmed by rendering before fixing it). The actual
  fix: stop using `theta`/`radius`/`dy` (polar + mark-level) for these two
  marks entirely, and instead manually compute each row's Cartesian anchor
  point in the transform (`LabelAnchorX`/`LabelAnchorY`, the same
  `sin`/`cos` trig already used for leader lines) plus each row's *actual*
  title/sublabel line counts as plain calculated fields
  (`LabelTitleLineCount`/`LabelSubtitleLineCount`, `.../LabelTitleY`/
  `LabelSubtitleY`). A per-datum *field* isn't restricted the way a
  per-datum *encoding channel* like `dy` is — so moving the math from "mark
  property" to "data column" sidesteps the limitation completely instead of
  approximating around it, and both marks now stack exactly regardless of
  how many lines each row's wrapped category name takes.
- **Legend `orient`/`legendX`/`legendY` can all be expr-bound to params**
  (confirmed by compiling and rendering) — this is what makes `legendOrient`
  a live, toggleable param instead of a fixed spec-time choice. But two real
  bugs turned up wiring `showLegend: false` on top of that:
  - **Pushing a legend off-canvas via `legendX`/`legendY` (e.g. `-9999`) to
    "hide" it does not work.** Vega's autosize computes the SVG's bounding
    box to include *everything*, including an off-canvas legend — this
    blew the rendered canvas up to ~20x its intended size and shifted the
    entire chart (translated to keep all content in positive coordinates)
    completely out of the visible viewport, i.e. the donut visually
    vanished. Confirmed by inspecting the compiled SVG's `viewBox` and root
    `<g transform>` directly, not just a screenshot. Fixed by keeping
    `legendX`/`legendY` on-canvas (`0`) when hidden, combined with the next
    point to make it actually invisible.
  - **A mark-level `stroke` encoding (used here for `borderColor`) makes
    Vega-Lite auto-generate a `legend.encode.symbols` block for the legend
    swatches — and that auto-generated block silently replaces, rather than
    merges with, any custom `legend.encode` also written in the spec.**
    A `legend.encode.legend`-scoped opacity toggle (the normal way to hide a
    legend without affecting layout) compiled away to nothing whenever the
    arc mark's `stroke` encoding was also present — confirmed by compiling
    both together in isolation and inspecting the output `legends[].encode`.
    Fixed by not using `encode` at all for this: `showLegend: false` instead
    zeroes the legend's own `symbolSize`/`labelFontSize`/`titleFontSize`
    properties (top-level legend properties, not part of `encode`), which
    aren't affected by the auto-generated symbols block and render nothing
    visible.

## Known limitations / roadmap

- `wedgeSelect` only drives the center label right now, not Power BI's own
  cross-filtering/cross-highlighting. Deneb's Settings tab has a
  **Cross-Filtering (Selection) of Data Points** toggle
  (Vega > Power BI Interactivity) that may bridge this automatically —
  worth testing in Power BI Desktop before building anything custom here.
- **Overlap prevention is a threshold heuristic, not true collision
  detection.** `labelMinAngleDegrees` hides the label (and leader line)
  entirely for wedges narrower than that angle — it stops the worst
  clutter from many tiny wedges, but it won't rearrange or offset labels
  that are individually wide enough to overlap their neighbors despite a
  wide-enough wedge (e.g. two adjacent wide-angle wedges with long,
  wrapped category names). A true collision-avoidance layout (like the
  two-column leader-line arrangement common in professional pie charts)
  needs an iterative layout algorithm that isn't expressible in declarative
  Vega-Lite transforms — it would need a custom Vega dataflow operator,
  which isn't available inside Deneb.
- ~~Wedge labels aren't perfectly vertically centered when word-wrapped~~
  **Fixed**: wedge title/sublabel positioning now uses a per-datum,
  manually-computed Cartesian anchor (`LabelAnchorX`/`LabelAnchorY`, the
  same trig already used for leader lines) plus each row's *actual*
  title/sublabel line counts (`LabelTitleLineCount`/`LabelSubtitleLineCount`)
  instead of `dy` — since `dy` can't be a per-datum encoding but a plain
  calculated field can, this sidesteps the limitation entirely rather than
  approximating around it. See "Implementation notes" for the mechanism.
- **The selected-wedge center view's category name still isn't perfectly
  centered when it wraps** — unlike the wedge label fix above, the center
  label was never positioned via `theta`/`radius` in the first place (it
  relies on Vega-Lite's automatic view-center placement for unencoded
  marks), so the same per-datum-Cartesian-anchor trick doesn't directly
  apply there. Its vertical-centering `dy` still uses a toggle-based (not
  wrap-aware) line count, so a long wrapped category name may sit slightly
  off-center. The *default* (no-selection) center view doesn't have this
  issue, since its title text is a static param, not data (see
  "Implementation notes").
- No sort control exposed — wedges render in incoming data order (matching
  the D3 original's `.sort(null)`); a later version could expose a param for
  sort-by-MeasureA/MeasureB/alphabetical.
- Color/font-family/label-toggle params are edited directly in the spec
  JSON, not via native Format-pane controls (see "Color" above). If your
  Deneb version supports Vega-Lite `bind` input widgets rendering usably
  inside the visual canvas, that's a possible next step for making these
  adjustable without opening the JSON editor.
- **Outside labels now sit on one constant ring, which can make a leader
  line cross through a neighboring wedge** for a chart with a few very
  large wedges (by `MeasureB`/radius) mixed with many small ones — the
  large wedge's own outer edge can sit close to or past the label ring,
  so a small wedge's leader line may visually pass near/through it. This
  reads fine in testing with real data (41 categories, and a synthetic
  73%-share stress test) but isn't a guaranteed-no-crossing layout; true
  crossing avoidance would need path routing, not a straight `rule` mark.
- **A corner-orient legend (see "Legend position and chart centering") can
  overlap wedges/labels in that corner**, especially a large wedge — it's
  an intentional tradeoff for keeping the donut centered on the full frame
  rather than reserving dedicated space. Switch to an edge orient if that
  overlap is a problem for your data's typical largest-wedge angle.
