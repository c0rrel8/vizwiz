# Visual design system

The shared look-and-feel for this library's Deneb visuals, established
while building the aster plot (`visuals/aster-plot/`) for a dark-canvas
Power BI report. Start every new visual from these values instead of
re-deriving them — copy the relevant params verbatim, then adjust only if
the new visual has a genuine reason to diverge (e.g. a different report's
canvas is light, not dark).

This file documents *values and rules*. For *why the spec is structured
the way it is* (params vs. Config, field-mapping adapter, etc.), see
`docs/patterns.md`.

## Color palette

| Role | Hex | Used for |
|---|---|---|
| Background / border | `#182231` | The visual's own background, and any border/stroke between adjacent chart elements (set both to the *same* value, so elements read as separated by a thin gap matching the page, not a bright stroke) |
| Title tier (primary text) | `#FCFCFD` | The most prominent text in any given context — see "Typography hierarchy" below |
| Subtitle tier (secondary text) | `#A5B4CB` | Secondary/supporting text, legends — see "Typography hierarchy" below |

These three colors are chosen for a **dark** report canvas. If a future
report/visual uses a light canvas instead, invert the direction (dark
background/border, dark title text, a mid-gray subtitle tier) — don't reuse
these exact hex values on a light canvas, they were picked for contrast
against `#182231` specifically.

Category/series fill colors are a separate concern from this palette —
those come from the data itself (a `ColorOverride`-style field driven by a
DAX measure, or a theme-matching Vega color scheme) so they can vary
per-category. See `docs/patterns.md`'s "Optional field with fallback"
pattern and the aster plot README's "Color" section for the two approaches
in full.

## Typography hierarchy

Every text-bearing element in a visual falls into one of two tiers. This
is a **content/emphasis** hierarchy, not a literal "font size" hierarchy —
both tiers can share the same font size; what differs is color (and
optionally weight):

- **Title tier** — the primary, most prominent piece of text in a given
  context: a data point's own label/category name, a center/summary
  label's headline, an axis or chart title. Color: `#FCFCFD`.
- **Subtitle tier** — secondary, supporting text attached to a title:
  a metric/percentage shown below a category name, a count or aggregate
  shown below a headline, legend text (both the legend's own title and its
  entries). Color: `#A5B4CB`.

**Font:** Segoe UI, for both tiers, everywhere. Matches Power BI/Office's
own default UI font rather than a generic `sans-serif` fallback.

**Font size units: points, not pixels.** Any user-facing font-size param
should be authored and documented in **points** (matching how Word/
PowerPoint/Power BI's own Format pane think about font size), then
converted to pixels internally via a derived param:
```json
{ "name": "labelFontSize", "value": 12 },
{ "name": "labelFontSizePx", "expr": "labelFontSize * 4 / 3" }
```
`4 / 3` is the standard 96-DPI pt→px conversion (`96/72`). Use the `*Px`
derived param everywhere an actual rendering property needs a pixel value
(`size` encodings, `lineHeight` math) — never feed the raw point value
directly into a pixel-space calculation.

**Reference sizes** (established for the aster plot; reuse for anything at
a similar visual weight/prominence):
- A data-point label (wedge label, bar label, etc.): **12pt**, `Segoe UI`,
  not bold.
- A center/summary/headline label: **14pt**, `Segoe UI`, **bold** (title
  line only — the subtitle line under a headline is never bold, regardless
  of the headline's own weight param).

## Auto-scaling: build it, default it off

Where a visual has room to auto-scale text (e.g. font size shrinking as
more lines are shown, or growing/shrinking per-datum by some other
dimension), implement it as an opt-in toggle, **defaulting to `false`**:
```json
{ "name": "labelAutoSize", "value": false },
{ "name": "labelFontSizeMin", "value": 7 },
{ "name": "labelFontSizeMax", "value": 13 }
```
Static, predictable sizing is the default; auto-scaling is an option for
someone who explicitly wants it, not a surprise behavior. This was a
direct, explicit design decision (not an oversight) — earlier revisions of
the aster plot had auto-scaling on by default and it was deliberately
flipped off.

## Title/subtitle text blocks: split into two marks, not one

When a single logical "label" contains both title-tier and subtitle-tier
content (e.g. a category name plus a metric below it), render it as **two
separate text marks** stacked vertically — one per tier — rather than one
combined multi-line text block with one color. A single Vega-Lite text
mark can only have one fill color for its entire (possibly multi-line)
string; there's no declarative way to color individual lines within one
mark.

**The hard part is vertical stacking**, especially when the title tier's
own line count is data-dependent (e.g. a wrapped category name can be 1 or
2+ lines depending on its length) — see `docs/patterns.md`'s "dy can't be
per-datum" pattern for the general technique, and the aster plot's
`LabelAnchorX`/`LabelAnchorY`/`LabelTitleY`/`LabelSubtitleY` transform
fields for a concrete, working implementation to copy from.

## Borders / gaps between elements

Default border/stroke color between adjacent chart elements (wedge
borders, bar gaps, etc.) to the **same value as the background**
(`#182231`), not a contrasting color like white. This reads as clean
negative-space separation rather than a drawn line, and looks correct
regardless of what fill colors the data ends up using.

## Legend

- Legend title and entry text both use the subtitle tier (`#A5B4CB`).
- Expose a `showLegend` (boolean) and `legendOrient` (string) param pair.
  `legendOrient` accepts Vega-Lite's edge orients (`"left"`/`"right"`/
  `"top"`/`"bottom"` — reserve layout space) and corner orients
  (`"top-left"`/`"top-right"`/`"bottom-left"`/`"bottom-right"` — render as
  a non-space-reserving overlay instead). This single param pair doubles as
  a chart-centering control: corner orients (or `showLegend: false`) keep
  the chart centered on the full visual frame; edge orients shift it to
  center within the remaining space. See the aster plot README's "Legend
  position and chart centering" section for the full mechanism and the two
  real Vega-Lite bugs found implementing `showLegend: false` (pushing a
  legend off-canvas breaks autosize; a mark `stroke` encoding silently
  clobbers a custom `legend.encode` block) — both documented there so they
  aren't rediscovered.

## Config tab: keep the spec self-contained

Don't rely on Deneb's separate Config tab for anything this system
controls (background, text colors, legend colors, fonts). Set every one of
these values explicitly via spec `params`, and leave the spec's top-level
`config` key absent entirely. This keeps the whole visual's look defined
in one place (the Specification tab, version-controlled with the spec) and
immune to whatever happens to be pasted into a given report's Config tab.
See `docs/patterns.md` for the fuller params-vs-Config rationale.
