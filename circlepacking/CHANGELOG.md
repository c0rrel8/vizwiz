# Changelog — Circle Packing

## 0.1.0

Initial build. A 3-level hierarchy (Level 1 > Level 2 > Level 3), each
level driven by its own field parameter over the shared 4 candidate
columns (a 3-way exclusion cascade extending the waffle chart's 2-way
pattern), a single metric sizing leaf circles, independent per-level
fill/stroke overrides with per-level default params, click-to-select with
subtree+ancestor highlighting, and cross-highlight-in support — bringing
this visual to parity with the aster plot's and waffle chart's existing
feature set, per the original request.

**The first raw-Vega visual in this library.** True circle packing needs
Vega's `stratify`/`pack` hierarchy transforms, which have no Vega-Lite
equivalent — confirmed as a deliberate, documented exception to this
library's "Vega-Lite only" convention, not a drift from it (see the
README's "Why raw Vega" and `docs/patterns.md`'s new "When to use raw Vega
instead of Vega-Lite" section).

Four real bugs found and fixed during the build, all documented in the
README's "Implementation notes" with how each was diagnosed:

- Raw Vega symbol marks size circles by bounding-square area (`4r²`), not
  circle area (`πr²`) like this library's existing Vega-Lite point-mark
  convention — confirmed empirically (an intended radius of 30 rendered at
  ~26.6, exactly the `√π⁄2` ratio predicted by the wrong formula).
- A mark's `"from"` property silently ignores an inline `"transform"`
  array — the root node rendered as a large stray background circle
  despite a filter meant to exclude it. Fixed by using real top-level
  filtered datasets (`packedNodes`, `level{N}NodeLabels`) instead.
- Event stream selectors for named marks are `"@markName:eventType"` with
  no trailing `!` — a guessed `"@fillMark:click!"` silently never fired.
- Label text marks drawn on top of a circle intercepted clicks meant for
  that circle, since marks are interactive/pickable by default — fixed
  with `"interactive": false` on all 3 label marks.

Also fixed, found via visual inspection rather than a silent failure:
radial-path labels rendered upside-down on the far side of the pack until
a standard 180°-flip-past-90° correction was applied.

Verified locally: default render (3-tier nesting correct), `fillStrokeGap`
at a visible value (gap renders on all levels), click-to-select at every
level (leaf, mid, and top — including toggle-off and a parent-area click
correctly lighting its full subtree while dimming unrelated branches),
cross-highlight-in via a synthetic `Measure A__highlight` field (correct
per-level dimming, composing correctly with click-select), tooltip content
(level/category/value/percent-of-parent, percentages summing to 100%
within a level), and radial label rotation at every position around the
pack (readable, not upside-down, after the fix above). Not verified:
anything requiring a live Power BI report or Deneb's actual "Provider:
Vega" rendering path — see the README's "Known limitations."

Required a small, one-time addition to `tools/validate.mjs`,
`render-preview.mjs`, and `interactive-harness.html`: detect a raw-Vega
spec (`.vg.json`, or a `$schema` without "vega-lite" in it) and skip the
`vega-lite.compile()` step, via a new shared `tools/spec-compat.mjs`
helper. Every future raw-Vega visual in this library benefits from this,
not just this one.
