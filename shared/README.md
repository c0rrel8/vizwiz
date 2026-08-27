# Shared gradient color measures

A reusable pair of DAX color measures — one for **fills**, one for **text
labels drawn on those fills** — plus the numeric base measure they both sit
on. Built to be retargeted at a new visual by editing a config block at the
top of each file, not by editing the logic underneath.

They replace the per-visual, hand-tuned gradient measures
(`aster/measureacolorgradient.dax`, `waffle/categorycolorgradient.dax`,
`waffle/categorycolorgradientrank.dax`), which are four near-identical copies
of the same idea with the endpoints, scale and direction baked into the body.
Those files stay where they are — nothing in the live report changes until
someone re-points a visual at these.

## The three files

| File | Returns | Bind to |
|---|---|---|
| `gradientposition.dax` | Number, `0`–`1` (or `1`–`N` rank) | Nothing, usually — it feeds the other two. Bind it directly only for native Power BI conditional formatting |
| `gradientfillcolor.dax` | `"#RRGGBB"` | `ColorOverride` / `ColorOverrideFill` role |
| `gradientlabelcolor.dax` | `"#RRGGBB"` | `LabelColorOverride` role |

Paste them in that order — each one references the one before it.

Only `gradientposition.dax` knows about the dimension and its field
parameter. The two color measures are thin wrappers, so the expensive,
repetitive field-parameter enumeration exists once instead of once per color.

## Rank or color code?

The ask allowed either. These emit **hex color codes**, because every Deneb
visual in this repo reads a per-datum color string straight off a bound field
(`isValid(override) ? override : defaultParam`) and never goes through Power
BI's native conditional formatting. Emitting hex also lets the label measure
read the *actual* fill color and pick text by real contrast, rather than
re-deriving it from a threshold.

The rank is still available: set `OutputScale = "RANK"` in
`gradientposition.dax` and it returns `1..N` for a native
"gradient by field value" rule. That setting breaks the two color measures
(they need `0..1`), so if you want both, duplicate the measure under a second
name rather than flipping this one back and forth.

## Setting one up for a new visual

1. **Rename all three measures** with the visual's prefix — e.g.
   `Waffle Gradient Position`, `Waffle Gradient Fill Color`,
   `Waffle Gradient Label Color`. Measure names are one model-wide namespace
   in this model (see `CLAUDE.md`), and this model has already had one
   measure pasted over another by name collision. Update the cross-references
   in the config blocks after renaming.
2. In `gradientposition.dax`, **CONFIG B**: point `SelectedField` at the
   visual's own field parameter, and search/replace `[Data Recieved GB]`
   with the gradient measure (5 occurrences).
3. In `gradientposition.dax`, **CONFIG A**: pick the basis, spacing and
   direction (table below).
4. In `gradientfillcolor.dax`, **CONFIG**: set `HexStart` / `HexEnd`.
5. In `gradientlabelcolor.dax`, **CONFIG**: usually nothing — `"CONTRAST"`
   mode has no threshold to tune and follows the endpoints automatically.

The field parameters and their label columns:

| Field parameter | Label column | Role |
|---|---|---|
| `MappedParameter` | `[Category]` | Level 1 / aster Category |
| `MappedParameterTwo` | `[Category2]` | Level 2 / waffle Category — confirmed live |
| `MappedParameterThree` | `[Category3]` | Level 3 — column name *not* confirmed live |

## Config reference

### `gradientposition.dax`

| Knob | Values | Meaning |
|---|---|---|
| `GradientBasis` | `"MEASURE"` \| `"SORT"` | Order members by a measure, or by a sort column |
| `Spacing` | `"EQUAL"` \| `"VALUE"` | Even steps per rank, or steps proportional to magnitude. `"SORT"` basis is always even |
| `ValueScale` | `"LINEAR"` \| `"LOG"` | Only with `Spacing = "VALUE"`. `"LOG"` for right-skewed measures |
| `SortDirection` | `"ASC"` \| `"DESC"` | Which end of the ramp the first member lands on |
| `OutputScale` | `"UNIT"` \| `"RANK"` | `0`–`1` for the color measures, `1`–`N` for standalone use |
| `LogOffset` | number | Added inside `LOG()` to survive zeros; raise it if the measure goes negative |
| `UseFieldParameter` | `TRUE` \| `FALSE` | `FALSE` = one fixed column, named in `FixedCandidate` |

To make the gradient **absolute** (fixed against the whole model, not
rescaled as slicers narrow the data), change `ALLSELECTED` to `ALL` in every
candidate block.

**Sorting by a column other than the dimension** — wrap the per-row sort key
in `CALCULATE`, in every candidate block:

```dax
"SortKeyTemp", CALCULATE ( MAX ( 'Table'[OrderCol] ) )
```

`ALLSELECTED(col)` is a one-column table, so any other column is not in row
context and cannot be referenced bare. The shipped default needs no
`CALCULATE` only because it names the column that is already in the table.

### `gradientfillcolor.dax`

| Knob | Meaning |
|---|---|
| `HexStart` / `HexEnd` | The two ends. Accepts `#RRGGBB`, `#RGB`, `RRGGBB`, or `#RRGGBBAA` (alpha dropped) |
| `UseMidStop` / `HexMid` / `MidStopAt` | Optional third stop for a diverging ramp |
| `HexFallback` | Used when the member has no value at all |

### `gradientlabelcolor.dax`

| Knob | Meaning |
|---|---|
| `LightHex` / `DarkHex` | The two candidate text colors |
| `SwitchMode` | `"CONTRAST"` (best WCAG ratio — recommended), `"LUMINANCE"` (fixed luminance threshold), `"POSITION"` (fixed % along the ramp) |
| `LuminanceThreshold` | `"LUMINANCE"` mode only. `0.179` is the standard crossover |
| `PositionThreshold` / `TextOnStartSide` / `TextOnEndSide` | `"POSITION"` mode only |
| `FillHexSource` | Any measure returning a hex string — it does not have to be the gradient |

## Worked example — the default ramp

`#A5C1FF` → `#4555D6`, text `#FCFCFD` / `#182231`. Computed by mirroring the
DAX arithmetic in a scratch script (the color math, not the DAX runtime — see
Verification below):

| Position | Fill | Luminance | `"CONTRAST"` | `"LUMINANCE"` | `"POSITION"` (0.55) |
|---|---|---|---|---|---|
| 0.0 | `#A5C1FF` | 0.534 | dark | dark | dark |
| 0.3 | `#88A1F3` | 0.372 | dark | dark | dark |
| 0.6 | `#6B80E6` | 0.243 | dark | dark | light |
| 0.7 | `#6275E2` | 0.208 | light | dark | light |
| 0.8 | `#586BDE` | 0.179 | light | light | light |
| 1.0 | `#4555D6` | 0.126 | light | light | light |

The three modes genuinely disagree in the middle of the ramp, which is the
point of having them — `"CONTRAST"` flips wherever the two text colors are
equally readable, the other two flip where you tell them to.

**Accessibility note on this particular ramp:** the weakest point is around
position 0.7, where the better of the two text colors still only reaches
**≈3.9:1**. That clears WCAG AA for large/bold text (3.0:1) but is under AA
for body text (4.5:1). Pulling `HexStart` darker or `HexEnd` darker fixes it;
`"CONTRAST"` mode will follow the change without any threshold retuning.

## Performance

`gradientposition.dax` ranks with `RANKX`, not the
`COUNTROWS(FILTER(...))` "count how many rows beat me" pattern used by the
older measures. That pattern is O(N²) and is the known cause of the
high-cardinality crash logged as open item 1 in `HANDOFF.md`; `RANKX` does one
pass. The trade-off is that ties now share a rank — and therefore share a
color — instead of being broken alphabetically. For a value-driven gradient
that is the better behavior: two members with identical values should look
identical.

The `SKIP` tie mode leaves gaps after a tie (`1, 1, 3`), so the rank still
spans `1..N` and the even spacing still lands on `0` and `1` at the ends.

## Naming gotchas

DAX rejects a `VAR` named with certain reserved words, with a bare syntax
error rather than anything that points at the cause. Two are confirmed live
in this model:

| Reserved | Use instead |
|---|---|
| `Rank` | `RankAsc`, `RankOut` — collides with the built-in `RANK` window function |
| `Members` | `MemberTable` |

The suffixed forms (`Rank_AlphaCode`, `MemberTable_AlphaCode`) are unaffected
— it is the bare word that collides. This bites when collapsing the measure
down to a single dimension, where the suffix has nothing left to distinguish
and the obvious move is to drop it.

## Verification status

- **Verified:** the color arithmetic — hex parsing (including `#RGB`
  shorthand, alpha, and unparseable input), RGB interpolation, WCAG relative
  luminance and contrast ratio, and the two-segment mid-stop math — was
  reproduced outside DAX and produces the table above.
- **NOT verified:** anything requiring a live model. No Power BI, no local
  DAX engine in this repo. Specifically unconfirmed: that `RANKX` ranks a
  **text** sort key alphabetically the way `"SORT"` basis assumes (if it
  misbehaves, point `SortKeyTemp` and `SortKey_X` at a numeric order column
  instead); that `UNION(ROW(...))` accepts `VAR` references as values in this
  engine version; and the `RANKX` performance claim above. Per `CLAUDE.md`,
  none of this is "confirmed" until it runs in the report.
