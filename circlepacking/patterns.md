# Reusable architectural patterns

Logical/structural patterns established while building the aster plot
(`visuals/aster-plot/`), meant to carry over to the next visual in this
library rather than being rediscovered from scratch. For colors/fonts/
sizing, see `docs/design-system.md` instead — this file is about *how the
spec is structured*, not what it looks like.

## Params, not Config, for everything the spec controls

Deneb splits a visual into three tabs: **Specification** (the Vega-Lite
spec + its `params`), **Config** (a separate Vega config JSON, meant for
cosmetic/theming), and **Settings** (Deneb's own fixed Format-pane
controls). Prefer putting every adjustable value — color, font, size,
toggle — into the Specification's `params` array, and leave the spec's
top-level `config` key absent entirely. Reasons:

- Vega-Lite's `params` have no equivalent in Config's schema, so there's
  zero risk of the two colliding.
- An explicit spec-level value always wins over a Config default for the
  same property — so once a property has a param, Config can no longer
  affect it anyway, making Config's role shrink to "whatever the spec
  doesn't bother to set" (in practice, this ends up being very little —
  see the aster plot README's "Setup in Deneb" section for the one
  concrete gap it currently has).
- A visual that's fully self-contained in its Specification is portable —
  paste it into a fresh Deneb visual and it looks right immediately,
  without also needing to remember and re-paste a matching Config JSON.

## When to use raw Vega instead of Vega-Lite

Default to Vega-Lite for every new visual. Vega-Lite is a declarative
layer that compiles down to Vega — it generates scales, axes, legends, and
(via `params`/`select`) the signals and event listeners that make
click-to-select work, all from a much smaller vocabulary than raw Vega
requires. Every visual in this library except `circle-packing` stays
entirely within that vocabulary, because the hard part of the aster plot
(wedge angle/radius math) and the waffle chart (grid cell allocation, the
row-break staircase) was always the *math*, expressed via `calculate`/
`window`/`aggregate` transforms — never a capability Vega-Lite lacked.

**Drop to raw Vega only when a spec needs something genuinely outside
Vega-Lite's compiled vocabulary** — there's no "asking Vega-Lite nicely"
for these, they simply aren't in what it compiles to:

- Hierarchy layouts (`stratify`, `pack`, `tree`, `partition`, `treemap`) —
  the trigger for `circle-packing`. Vega-Lite has no equivalent transform;
  true circle packing needs an actual packing algorithm (fit circles
  without overlap, size parents to enclose children), which isn't
  expressible in row-wise `calculate`/`window` transforms the way closed-
  form geometry (wedge angles, grid cells) was.
- Custom signal logic beyond point/interval selection — e.g. the circle
  packing visual's subtree+ancestor click-highlighting (string-prefix
  tests against a node's hierarchy path) isn't expressible through
  Vega-Lite's `param`/`select` shorthand at all; it needs a hand-written
  signal.
- Force layouts, custom projections, voronoi — same class of reason, not
  yet needed by anything in this library.

**There is no benefit to migrating an existing Vega-Lite visual to raw
Vega just because a newer one needed it.** Vega-Lite gives every existing
visual its scales/legends/`params` machinery for free; raw Vega means
hand-writing all of it, plus re-deriving and re-verifying everything
that's already tested and documented, for zero functional gain if nothing
about that visual actually needs raw Vega's extra capability. Decide per
visual, not as a blanket rule.

### What translates, and what doesn't, when a visual does need raw Vega

Most of this library's conventions carry over directly, just in raw
Vega's more verbose syntax:

| Vega-Lite | Raw Vega |
|---|---|
| `params` array | top-level `signals` array (`{"name": ..., "value": ...}` or `"update"` for a derived one) |
| `calculate` transform | `formula` transform — same expression language, same `{"expr": "...", "as": "..."}` shape |
| `aggregate: [{"op", "field", "as"}, ...]` | `aggregate` transform with **parallel arrays** instead — `"fields": [...], "ops": [...], "as": [...]` |
| `param`/`select` (click-to-highlight) | a signal with an `"on"` array of event-stream handlers (`{"events": "@markName:eventType", "update": "..."}`) — see `circle-packing`'s `packSelect` for a worked example, including the subtree/ancestor test that wouldn't be expressible in Vega-Lite at all |
| `condition` array on an encoding channel | an array value for that same `encode.update` property, evaluated in the same first-match-wins order (Vega's own native array-of-conditions support, not a Vega-Lite-only feature) |
| Legend from a `color` encoding's `legend` object | a top-level `legends` array referencing a named scale — not used yet in this library's one raw-Vega visual (`circle-packing` uses per-node label marks instead of a legend; see that visual's README for why) |
| Field-mapping adapter, color-fallback pattern, DAX conventions, versioning discipline | unchanged — none of these are Vega-Lite-specific ideas |

One genuine gotcha, not just a syntax translation: **raw Vega's `symbol`
mark sizes a `"circle"` shape by bounding-square area (`4r²`), not circle
area (`πr²`) the way this library's Vega-Lite point-mark convention
documents** (`size = π × diameter² ⁄ 4`, established in the aster plot's
and waffle chart's own READMEs). Confirmed empirically while building
`circle-packing` — see that visual's README for the exact repro. Vega-Lite
evidently applies its own conversion when compiling a point/circle mark
down to the underlying Vega symbol mark; a spec written directly against
raw Vega doesn't get that conversion for free. Re-verify circle sizing
empirically (render a known radius, measure the actual SVG output) on any
future raw-Vega visual rather than assuming the Vega-Lite formula carries
over.

A mark's `"from"` property also does **not** support an inline
`"transform"` array the way Vega-Lite's per-layer transforms might
suggest — confirmed by a real bug (see `circle-packing`'s CHANGELOG):
filtering data inline in a mark's `from` silently compiled and ran with no
error, but the filter simply never applied. Use a real top-level filtered
dataset and point the mark's `"from": {"data": ...}` at that instead.

### `.vg.json` naming convention

A raw Vega spec file uses the `.vg.json` extension (vs. `.vl.json` for
Vega-Lite), matching Vega's own conventional file extension. `tools/
validate.mjs`, `render-preview.mjs`, and `interactive-harness.html` all
detect which kind a spec is (by `$schema` — containing `vega-lite` or not
— or by trying `.vl.json` first and falling back to `.vg.json`) via the
shared `tools/spec-compat.mjs` helper, and skip the `vega-lite.compile()`
step for a raw Vega spec. This is done once, in the tooling, not
per-visual.

## Field-mapping adapter

Vega-Lite field references (`"field": "X"`) are static strings baked in at
spec-authoring time — there's no way to make a `"field"` dynamically
resolve to "whichever column the report author bound." Rather than
scattering the actual incoming Power BI field name across dozens of
`calculate`/encoding references, put a small number of `calculate` steps at
the very top of the `transform` array that map incoming field names to
fixed internal canonical names, and have every other calculation/encoding
in the spec reference only the canonical names:

```json
{ "calculate": "datum.MeasureA", "as": "MeasureA" },
{ "calculate": "datum.MeasureB", "as": "MeasureB" }
```

This isolates any future rename to these few lines instead of a
spec-wide search-and-replace. See the aster plot spec's own opening
`transform` steps and its README's "Field mapping" section for the full
pattern, including the field-parameter-specific wrinkle below.

### Field parameters need an `isValid()` enumeration, not a direct reference

If a role is driven by a Power BI **field parameter** (letting a report
viewer swap which underlying column feeds a role via a slicer), the field
parameter's own name/label column is **not** what Deneb actually receives
data under — confirmed live, after two wrong assumptions, by comparing
Deneb's "view as table" output against a plain table visual's column
headers. What Deneb actually receives is keyed by the **real underlying
column's own name** — whichever candidate is currently selected — which
changes at runtime with the live slicer selection. Since a Vega-Lite field
reference can't be dynamic, the adapter step for that role must instead
enumerate every known candidate and pick whichever one actually has data:

```json
{
  "calculate": "isValid(datum.CandidateA) ? datum.CandidateA : isValid(datum.CandidateB) ? datum.CandidateB : isValid(datum.CandidateC) ? datum.CandidateC : '(Unspecified)'",
  "as": "Category"
}
```

The trailing static fallback (`'(Unspecified)'` above) matters too — real
production data can include rows with no value in *any* candidate column;
without a fallback these render as a literal, illegible `"undefined"`.

This mirrors an identical DAX constraint: a DAX measure referencing "the
field parameter's currently-selected column" has to enumerate candidates
via `SWITCH`, for the same underlying reason. See
`visuals/aster-plot/dax/category-field-parameter.dax` for the DAX side.

**One field parameter per report is assumed by this pattern as written.**
If a second, independent visual needs its own swappable role, that field
parameter must be uniquely named model-wide, and that second visual needs
its own copy of this adapter step with its own candidate list — nothing
else in the spec needs to change, since everything downstream already
reads only the canonical name.

### Two field parameters, same candidates, one visual: use an exclusion cascade

The above assumes only *one* role in a given visual is field-parameter-
driven. The waffle chart needed **two** (a series role and a category
role, each independently swappable) — over the *same* 4 candidate
columns. Confirmed live (a "view as table" screenshot with the two field
parameters sliced to different candidates): Deneb exposes both as
separate, simultaneous keys in that case, no forced collision. But a
plain `isValid()` cascade — the single-role version above, copy-pasted
for the second role — silently breaks: since both roles check the same
candidates in the same order, whichever candidate is present and checked
first wins for **both** roles, regardless of which field parameter it
actually came from. Reproduced locally before it was understood (two
different incoming columns, both roles still resolved to the same one).

The fix is an **exclusion cascade**: resolve the first role normally, also
recording *which candidate name* it used (as a string, via the identical
cascade structure); the second role's cascade then requires each candidate
additionally not match that recorded name:

```json
{
  "calculate": "isValid(datum.CandidateA) ? 'CandidateA' : isValid(datum.CandidateB) ? 'CandidateB' : isValid(datum.CandidateC) ? 'CandidateC' : ''",
  "as": "RoleOneSource"
},
{
  "calculate": "isValid(datum.CandidateA) ? datum.CandidateA : isValid(datum.CandidateB) ? datum.CandidateB : isValid(datum.CandidateC) ? datum.CandidateC : '(Unspecified)'",
  "as": "RoleOne"
},
{
  "calculate": "(isValid(datum.CandidateA) && datum.RoleOneSource !== 'CandidateA') ? datum.CandidateA : (isValid(datum.CandidateB) && datum.RoleOneSource !== 'CandidateB') ? datum.CandidateB : (isValid(datum.CandidateC) && datum.RoleOneSource !== 'CandidateC') ? datum.CandidateC : null",
  "as": "RoleTwo"
}
```

Two details that matter and are easy to get backwards:
- The **second role's terminal fallback must be `null`, not a string like
  `'(Unspecified)'`** — `isValid('(Unspecified)')` is `true` (any normal
  string passes), so a string fallback silently defeats any downstream
  `isValid(datum.RoleTwo) ? ... : 'somethingElse'` check that depends on
  detecting "this role has nothing." Confirmed by testing `isValid()`
  directly against a string vs. `null`. The *first* role can safely use a
  string fallback if it doesn't feed a similar downstream check.
- If both field parameters are ever sliced to the **identical** column at
  the same time, the second role's cascade correctly (and safely) resolves
  to nothing, rather than duplicating the first role's value under a false
  pretense of being independent — verify this is the behavior you want
  before relying on it.

Verify any adapter built this way against all three shapes before trusting
it: both roles bound to *different* candidates, the second role's field
parameter entirely *unbound*, and both roles bound to the *same* candidate
— see `visuals/waffle-chart/spec/waffle-chart.vl.json` and its README's
"Field mapping" section for the worked, verified example.

## Optional field with `isValid()` fallback

For any value that's *usually* computed from a static param but should be
overridable per-row from an optional bound field (e.g. a per-category
label color driven by a DAX measure, with a sensible default when no such
measure is bound), use the same `isValid()` shape:

```json
{ "calculate": "datum.LabelColorOverride", "as": "LabelColorOverride" },
```
```json
{
  "calculate": "isValid(datum.LabelColorOverride) ? datum.LabelColorOverride : (labelPosition === 'outside' ? labelColorOutside : labelColorInside)",
  "as": "LabelColor"
}
```
The field-mapping adapter step for the optional field is a no-op pass-
through when nothing is bound (`datum.LabelColorOverride` is simply
`undefined`), so it's safe to always include, whether or not the report
author ever binds it.

## Chained `window` transforms each need their own identical `"sort"`

A `"sort"` on one `window` transform does **not** carry over to a later,
separate `window` transform in the same `transform` array — each one
needs its own, and a mismatch between them doesn't just misorder results,
it can silently **delete rows downstream**.

The waffle chart's cell-allocation math (a cumulative `sum` producing
`CellEnd`, then a `lag` on `CellEnd` producing `CellStart`, both meant to
walk categories largest-to-smallest) had `"sort"` on the `sum` step but
not the following `lag` step. The `lag` step silently fell back to
incoming (often alphabetical) row order, so a category's `CellStart`
became "the previous row *alphabetically*'s `CellEnd`" instead of "the
previous row *by size*'s `CellEnd`" — producing wrong, sometimes
**negative-width** ranges (`end < start`) wherever the alphabetical and
size-sorted neighbors differed enough. Feeding a negative-width range into
`sequence(start, stop)` returns an **empty array**, and `flatten` on an
empty array produces **zero rows** — so affected categories didn't
misplace, they vanished entirely, including from any legend whose domain
is built from what actually reaches the mark. This read as two unrelated
bug reports ("sort isn't working," "some categories aren't rendering")
before being traced to one root cause, by dumping the transform's output
immediately before the `flatten` step and checking for `end < start`.

**The fix and the general rule**: every `window` transform in a chain
that depends on a consistent row order needs its own explicit, identical
`"sort"` — never assume it's inherited from an earlier step. Vega-Lite
compiles a mismatched chain like this without any warning, live or
compile-time, so nothing will flag it until the output looks wrong.

## `dy` (and other per-datum-restricted properties) can't vary by row

Several Vega-Lite mark properties are **mark-level only** — they accept a
uniform value or `expr`, but not a per-datum field encoding. Confirmed by
compiling and checking for a silent-drop warning: `dy`, `lineHeight`.
`rule` marks additionally don't support `theta`/`radius` polar encoding at
all.

This matters whenever something that *should* vary per row — like how far
down a second text mark needs to sit, based on the first mark's own
wrapped line count for that specific row — needs exactly this kind of
per-datum value. **The fix is to move the calculation from the mark layer
to the data layer**: instead of a per-datum `dy`, compute a per-datum
absolute pixel position as a plain `calculate` transform field (which has
no such restriction), and encode that as `x`/`y` (with `"scale": null`)
instead of `theta`/`radius`/`dy`. Concretely, for polar-positioned marks
that need this:

```json
{ "calculate": "width / 2 + datum.SomeRadius * sin(datum.SomeAngleRad)", "as": "AnchorX" },
{ "calculate": "height / 2 - datum.SomeRadius * cos(datum.SomeAngleRad)", "as": "AnchorY" },
{ "calculate": "datum.AnchorY - (datum.TotalLineCount - 1) * lineHeight / 2", "as": "TitleY" },
{ "calculate": "datum.TitleY + datum.TitleLineCount * lineHeight", "as": "SubtitleY" }
```
then encode `"x": {"field": "AnchorX", "scale": null}`, `"y": {"field":
"TitleY"/"SubtitleY", "scale": null}` on the two marks instead of
`theta`/`radius`/`dy`. This is exactly how the aster plot's wedge
title/sublabel marks are positioned — see its transform array and
`docs/design-system.md`'s "Title/subtitle text blocks" section.

Where the *only* thing varying is a genuinely uniform (not per-row) line
count — e.g. a static center-title param whose wrap-driven line count
doesn't depend on data — the plain `dy`-as-`expr` approach is fine and
simpler; reach for the Cartesian-anchor version only when the line count
is truly per-datum.

## DAX file convention

Every `.dax` file's **first line** must be the `Name = ` assignment itself
— explanatory `//` comments go *after* that line, never before it. A
comment block preceding the name/assignment breaks Power BI's DAX editor
(confirmed live). This is a hard formatting rule for every DAX file in
this repo, not a style preference.

**Don't name a `VAR` after a DAX function, even one that looks unlikely
to collide.** Power BI DAX now has real built-in `RANK`, `ROWNUMBER`,
`OFFSET`, and `INDEX` window functions (relatively recent additions) —
`VAR Rank = ...` throws a syntax error, confirmed live, even though
nothing in this repo's older DAX predates that being a problem. Suffix or
qualify a `VAR` name if there's any chance it matches a function name
(`RankValue`, not `Rank`) — this isn't limited to the obviously-reserved
words like `SUM`/`FILTER`/`CALCULATE`; DAX's function list keeps growing.

## Deneb cross-filtering (out) vs. cross-highlighting (in): different features, different odds

Power BI's Deneb visual exposes two separate interactivity toggles in
Settings under **Vega > Power BI interactivity** — **Expose
cross-filtering values for dataset rows** (a click on this visual filtering
*other* visuals) and **Expose cross-highlight values for measures** (other
visuals' selections dimming/highlighting *this* one). They're driven by
different mechanisms, and one is far more likely to work than the other in
a spec built around the field-mapping-adapter/proportional-composition
patterns this library uses:

- **Click-to-filter out is likely to fail for any visual whose marks are
  more than one reshaping transform away from a raw dataset row.** Deneb's
  "Simple" cross-filtering mode (the only mode available for Vega-Lite —
  "Advanced," where you resolve cross-filtering yourself, is Vega-only)
  works by re-running the spec's transform pipeline headlessly and tracing
  a clicked mark's datum back to the real rows that produced it. Deneb has
  a filed, confirmed issue for this breaking across the `fold` transform
  ([deneb-viz/deneb#494](https://github.com/deneb-viz/deneb/issues/494)) —
  any transform of the same *shape* (one row becoming many, or vice versa,
  more than once in the pipeline) is a reasonable bet to break the same
  way. This library's waffle chart aggregates raw rows into one per
  category, then explodes that back into many synthetic per-cell rows via
  `sequence`/`flatten` — confirmed live, this breaks click-to-filter-out
  entirely. The aster plot's wedges are built the same general way
  (aggregate, then derive display geometry) and should be assumed to have
  the same limitation until proven otherwise. **There is no spec-level fix
  for this while a visual needs one mark per synthesized display unit
  rather than one mark per real row** — a native Power BI visual (built
  with the actual TypeScript custom-visuals SDK) sidesteps it entirely,
  since it builds `ISelectionId`s per row directly and never routes through
  Deneb's transform-lineage tracing. Worth keeping in mind as an escape
  hatch if a customer specifically needs full bidirectional cross-filtering
  out of a visual like this — not something to build speculatively ahead of
  that need.
- **Cross-highlighting in is a normal spec feature, not a Deneb-internal
  mechanism to fight.** With the toggle on, Deneb adds a
  `[measure name]__highlight` field to every *raw* row before your
  transform pipeline runs — it's just another incoming field. Read it with
  an `isValid()` fallback to the measure's own value (so behavior is
  unchanged when the toggle is off, or in any tooling that doesn't inject
  it), carry it through the same `aggregate`/`groupby` step as the measure
  itself, and encode whatever "is this in the current highlight" signal you
  derive from it (e.g. `highlightTotal > 0`) into an opacity/dimming
  channel the same way a local click-selection would. See the waffle
  chart's `MetricAHighlight` → `CategoryHighlightTotal` →
  `CategoryIsHighlighted` chain and its README's "Cross-filtering and
  cross-highlighting" section for a full worked example, including how it
  composes with an existing click-to-dim selection via a two-entry
  `condition` array (checked in order — the two behaviors don't need to be
  merged into one expression).

## Versioning discipline

Every commit that changes a visual's spec or DAX should, in the same
commit:
1. Bump `package.json`'s `version`.
2. Add a dated/numbered entry to that visual's own `CHANGELOG.md`
   (reverse-chronological, most recent first), describing *why* the
   change was made and what broke/was fixed, not just what changed
   syntactically.

## Local dev verification, before shipping any change

Before telling a report author a change is ready to try, run it through
the local toolchain in `tools/` — none of it touches Power BI, all of it
runs in plain Node/a headless browser:

- `node tools/validate.mjs` — compiles every visual's spec + sample data
  via `vega-lite`/`vega`. Catches thrown compile errors only — it will
  **not** catch a silently-wrong-but-valid spec (e.g. a field resolving to
  `undefined` everywhere renders fine, just wrong).
- `node tools/render-preview.mjs <name>` — renders to SVG for a quick
  visual sanity check (`tools/screenshot.mjs` turns that into a PNG via
  headless Chromium).
- `node tools/test-interaction.mjs <name> <x> <y>` — drives a real
  browser via Playwright and simulates an actual click, since compiling
  and rendering a static frame never executes selection/interaction logic
  at all.

For anything involving Power BI-specific runtime behavior this repo can't
reproduce locally (Deneb's own field-exposure quirks, the real report
theme, actual Format-pane behavior), say so explicitly rather than
claiming a fix is confirmed — every non-local claim in this repo's history
has needed the report author's own live test to actually confirm.

## Shared data model across visuals (for now)

As of the waffle chart, every visual in this library is being bound to the
**same underlying Power BI model/report** — not a separate model per
visual. Until told otherwise, a new visual's field-mapping adapter should
default to reusing these same real field names rather than inventing new
placeholder ones, mapping them to whatever internal canonical roles that
visual actually needs:

| Real Power BI field | Typical role |
|---|---|
| `Category1` | A grouping/series dimension (e.g. a small-multiple split) |
| `Category2` | A second grouping dimension (e.g. the primary category/legend) |
| `Measure A` | A relative-magnitude measure (note the space — needs `datum['Measure A']` bracket notation, not dot notation, in any `calculate` expression) |
| `ColorOverrideFill` | Per-category fill color (hex) |
| `ColorOverrideBorder` | Per-category stroke color (hex) — the raw field keeps this name (a live-bound Power BI field can't be renamed without re-binding it in the report); its spec-internal alias is `StrokeColorOverride`, per `docs/design-system.md`'s "border"→"stroke" naming convention |

Which of `Category1`/`Category2` maps to which *role* is genuinely
ambiguous from the names alone — they aren't labeled by role — so treat
that mapping as a documented assumption in the visual's own README (see
the waffle chart's "Fields expected" section for the pattern to copy),
not a settled fact. It's a one-line fix in the adapter if it's backwards.

This is a starting default, not a hard rule — if a new visual genuinely
needs a field these five don't cover (a second measure, a date, etc.),
just add it and document it normally. And if the user ever moves a visual
to a different model/report, this table stops applying to that visual;
update its own README's "concrete bindings" section instead of assuming
this one.

**DAX measure names must be unique across the whole shared model, not
just within one visual.** When writing a new DAX template for a new
visual, don't reuse a measure name an earlier visual's template already
uses (e.g. a generic `Category Color Gradient`), even if the new
template is conceptually "the same kind of thing" for a different
visual — Power BI measure names are model-wide, and a name collision
means whichever measure gets pasted second silently overwrites the
first in the report author's actual model (this happened for real: a new
gradient-measure template was named identically to the aster plot's
existing one, and got pasted over it). Give each new DAX template's
measure a name that's unique model-wide — prefixing with the visual or
role it's for (e.g. `Category2 Color Gradient` rather than `Category
Color Gradient`) is enough.

## Never fabricate "realistic" sample data

`spec/sample-data.json` files in this repo are for local compile/render
tooling only — they have no live connection to any report's real data.
Keep their values obviously synthetic (placeholder names, round numbers)
rather than inventing numbers that could look like real business data.
When a fix genuinely needs to be verified against real-shaped data, ask
for real values or a real (possibly redacted) sample rather than guessing.
