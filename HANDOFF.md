# Handoff notes — Power BI Deneb Visuals library

Picking this up in a new Claude session/account. Read this first, then the
repo's own `README.md`/`docs/patterns.md`/`docs/design-system.md` for the
full architecture.

## Where the actual work lives

Everything is committed to git, branch `claude/local-git-github-sync-pzb8iz`,
currently at commit `f45c283`. **Nothing has reached GitHub yet** — see
"The GitHub sync blocker" below. To restore this exact state in a new
environment:

```
git clone softcrylic-dataviz.bundle softcrylic-dataviz
cd softcrylic-dataviz
git checkout claude/local-git-github-sync-pzb8iz
npm install
node tools/validate.mjs   # sanity check — should print "OK" for all 3 visuals
```

## The GitHub sync blocker

`git push` to `c0re71/softcrylic-dataviz` fails with a GitHub-side 403:
*"Permission to C0re71/softcrylic-dataviz.git denied to C0re71"* — this is
a real GitHub permission/App-installation issue, not a network fault, and
not something retrying fixes. **This may or may not persist under a
different Claude account** — it depends on whether that account's
Claude-GitHub integration has been separately granted write access to this
repo. Worth checking early in the new session (attempt a push, or check
the repo's GitHub App installation permissions) rather than assuming a
fresh account automatically resolves it.

## What's built (3 visuals, all in `visuals/`)

1. **`aster-plot`** (Vega-Lite) — mature, stable. Wedge angle/radius
   chart. Recently backported the waffle chart's fill/stroke/gap feature
   set (see its own CHANGELOG v0.11.0) and the "stroke," not "border,"
   naming convention.
2. **`waffle-chart`** (Vega-Lite) — mature. Grid-of-shapes chart, with an
   opt-in `useRowBreaks` staircase layout (a deliberate design choice to
   convey category grouping without using color, since this report can't
   use color for set identity), cross-highlight-in support, and small
   multiples via `SeriesCategory`.
3. **`circle-packing`** (raw Vega, `.vg.json`) — v0.20.2, **confirmed
   rendering live against real report data**. The library's first
   raw-Vega spec (true circle packing needs Vega's `stratify`/`pack`
   hierarchy transforms, which Vega-Lite doesn't have — see
   `docs/patterns.md`'s "When to use raw Vega instead of Vega-Lite").
   3-level hierarchy (Level1/Level2/Level3), each level its own field
   parameter (`MappedParameter`/`MappedParameterTwo`/`MappedParameterThree`
   — the first two are *reused* from the aster plot/waffle chart, not new;
   see the README's "Fields expected" for the shared-slicer implication).

## Two open next-step items (not blockers)

Both surfaced during circle packing's live confirmation, both documented
in the relevant READMEs/CHANGELOGs but not fixed yet:

1. **The rank-based color gradient DAX
   (`visuals/waffle-chart/dax/category-color-gradient-rank.dax`) crashes
   at high cardinality** — thousands of distinct leaf values against an
   O(N²) rank pattern. Prominent warning header now on the file itself;
   needs re-engineering (drop the alphabetical tie-break in favor of
   RANKX/`RANK` window function accepting ties, or precompute the sort
   into a calculated column on the fact table) before per-category rank
   coloring is safe to bind on circle packing. Circle packing currently
   uses solid per-level colors, which sidesteps this entirely.
2. **`MappedParameterThree` still needs to be created in the report's
   model** — the DAX for it is at
   `visuals/circle-packing/dax/category-three-field-parameter.dax`, not
   yet pasted live. Level 1/2 reuse the aster plot's and waffle chart's
   existing field parameters directly (see the circle packing README's
   "Fields expected"); only Level 3 is a genuinely new field parameter.

## Learned this session (documented in docs/patterns.md)

- **Deneb's Provider setting is fixed at visual creation, not switchable
  later.** The UI toggle in Settings changes, but the actual compiler
  doesn't swap — a repurposed Vega-Lite Deneb visual will keep failing
  Vega-Lite validation on a raw Vega spec even after Provider is set to
  Vega and ▶ Run is pressed. Fix: create a brand-new Deneb visual and
  set Provider to Vega *before* pasting the spec.

## Working conventions established across this whole session (see docs/patterns.md and docs/design-system.md for the full versions)

- Params (or raw-Vega `signals`) hold every adjustable value; Deneb's
  Config tab is deliberately left empty.
- A field-mapping adapter (`calculate`/`formula` steps at the top of the
  transform pipeline) isolates every place a Power BI field rename could
  require a spec change, mapping raw incoming field names to fixed
  internal canonical names.
- Every color override field has an `isValid(override) ? override :
  defaultParam` fallback — never reference the raw override field
  directly outside that one fallback calculation.
- Every commit that changes a visual's spec or DAX bumps the root
  `package.json` version and adds a dated entry to that visual's own
  CHANGELOG.md explaining *why*, not just *what*.
- Nothing is claimed "confirmed" unless it was actually verified — either
  locally via `tools/validate.mjs`/`render-preview.mjs`/
  `test-interaction.mjs` (compiles, renders, or an actual simulated click
  via Playwright), or live in Power BI Desktop by the report author.
  "Verified locally" and "confirmed live" are kept explicitly distinct
  throughout every README — this repo's history includes multiple cases
  where something that compiled cleanly was still silently wrong (e.g. a
  `joinaggregate` that multiplied cell counts, an arc mark's `filled:
  false` layer losing its click hit-area, a wrong circle-size formula in
  raw Vega) — none of those threw an error, all were only caught by
  actually rendering and looking, or by a live report author's report.
