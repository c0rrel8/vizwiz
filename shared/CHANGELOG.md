# Changelog — Shared gradient measures

## 0.2.0 — 2026-08-27

**Ties on the measure value no longer share a color, and no longer break the
ramp.** New `TieBreak` config (`"SORT"` default, `"NONE"` for the old
behavior).

The reason ties shared a color in `0.1.x` was that `RANKX` ranks one
expression and has no secondary tie-break — the thing the O(N²)
`COUNTROWS(FILTER(...))` pattern was really buying. The fix keeps the single
pass and folds the tie-break into the ranked expression: value and sort key
are packed into one zero-padded sortable text key, ranked once. Three
constraints come with that (padding width, negative values, numeric sort
columns needing their own padding) — all three are documented in the measure
and in the README, and the text ordering was verified outside DAX.

**Also fixes a real defect in `0.1.x`:** `SKIP` ranks divided by member count
meant a tie *at the maximum* silently compressed the ramp — 5/10/20/20 topped
out at position 0.67 and `HexEnd` was never used. Ranking is now `DENSE` and
the spacing divides by the distinct *key* count, so the last rank always lands
on 1.0 in both tie modes.

Raises the risk on the one unverified assumption: the packed key is text in
every mode, so `RANKX` ranking text is now load-bearing everywhere, not just
for `"SORT"` basis. Test it on a throwaway table before wiring it up — see the
README's "Verification status".

Structural: one ranking key replaces the separate measure-rank and sort-rank
variables, so the engine collapses from two rank `SWITCH`es to one.

## 0.1.2 — 2026-08-27

`Members_X` renamed to `MemberTable_X`. **Confirmed live by the report
author: `Members` is reserved** — a bare `VAR Members =` is rejected. Same
class of finding as the bare `VAR Rank =` collision found earlier (see
`waffle/categorycolorgradientrank.dax`), and it bites in the same place: when
this measure is collapsed to a single dimension, the per-candidate suffix has
nothing left to distinguish and the obvious move is to drop it, landing on the
reserved bare word. Both are now tabled in the README under "Naming gotchas".

## 0.1.1 — 2026-08-27

Docs only. Documented that a sort key pointing at a column other than the
dimension must be wrapped in `CALCULATE` — `ALLSELECTED(col)` is a one-column
table, so any other column is not in row context and cannot be referenced
bare. The shipped default sorts by the dimension column itself, which is why
it needs no `CALCULATE`; anyone repointing it at an explicit order column
would have hit this.

## 0.1.0 — 2026-08-26

Initial release of the parameterized gradient measure trio
(`gradientposition.dax`, `gradientfillcolor.dax`, `gradientlabelcolor.dax`).

**Why:** the four existing gradient measures (`aster/measureacolorgradient`,
`waffle/categorycolorgradient`, `waffle/categorycolorgradientrank`) are
near-identical copies of one idea with the endpoints, scale, direction and
rank strategy hard-coded into the body of each. Retargeting one at a new
visual meant editing interpolation math and rank logic by hand, in four
places, with the field-parameter enumeration duplicated in every copy. These
three isolate every adjustable value into a config block at the top of each
file and leave the logic beneath it generic.

- **`gradientposition.dax`** — the only file that knows about the dimension
  and its field parameter. Counts the dimension's distinct members in the
  current filter context, ranks the current member `1..N`, returns a `0..1`
  position (or the raw rank, via `OutputScale = "RANK"`, for native Power BI
  conditional formatting). Knobs: measure-driven vs sort-column-driven,
  rank-spaced vs magnitude-spaced, linear vs log, ascending vs descending,
  field parameter vs fixed column.
- **`gradientfillcolor.dax`** — hex fill from a 2- or 3-stop ramp. Accepts
  `#RRGGBB`, `#RGB`, bare, or `#RRGGBBAA` endpoints; unparseable input
  degrades to a dark color rather than erroring.
- **`gradientlabelcolor.dax`** — light/dark text color, by best WCAG contrast
  against the actual fill (default), by a fixed luminance threshold, or by a
  fixed % along the ramp. The two color-based modes take any hex-returning
  measure, so they work over fixed-color measures too, not just this gradient.

**Addresses `HANDOFF.md` open item 1** (rank gradient crashes at high
cardinality): ranking uses `RANKX` instead of the O(N²)
`COUNTROWS(FILTER(...))` pattern. Consequence: ties share a rank and a color
instead of being broken alphabetically. Not yet verified live — the crash was
reported from the report, and only the report can confirm the fix.

Color arithmetic verified by reproducing it outside DAX; everything requiring
a live model (`RANKX` over a text sort key, `UNION(ROW(...))` with `VAR`
values, the performance claim) is unverified. See the README's "Verification
status".
