# vizwiz — Deneb visuals library for Power BI

## Repo structure

Each visual lives in its own directory with a `.vg.json` or `.vl.json`
spec, DAX templates, README, and CHANGELOG:

```
aster/          Vega-Lite (.vl.json) — radial wedge chart (angle + radius)
waffle/         Vega-Lite (.vl.json) — grid-of-shapes proportional chart
circlepacking/  Raw Vega  (.vg.json) — nested circles, 3-level hierarchy
sunburst/       Raw Vega  (.vg.json) — concentric arc rings, 3-level hierarchy
dendrogram/     Raw Vega  (.vg.json) — tree diagram, 3-level hierarchy
network/        Raw Vega  (.vg.json) — force-directed network, dual data mode
shared/         DAX only — reusable, parameterized color measures
```

`shared/` holds the parameterized gradient measure trio (position → fill
color → label color). Prefer retargeting those at a new visual, via their
config blocks, over writing another bespoke gradient measure — see
`shared/README.md`.

Root-level files: `patterns.md` (architectural patterns), `designsystem.md`
(colors/fonts/sizing), `HANDOFF.md` (session history). The root `README.md`
is the aster plot's own documentation (legacy flat structure, not a repo
overview).

## Vega vs Vega-Lite decision

Default to **Vega-Lite** (`.vl.json`). Drop to **raw Vega** (`.vg.json`)
only when the spec needs hierarchy layouts (`stratify`, `pack`, `partition`,
`tree`, `treemap`) or custom signal logic beyond `param`/`select`. In
Deneb, **Provider must be set to Vega before pasting** — it cannot be
switched after visual creation. Always call out Vega vs Vega-Lite when
delivering a new visual.

## Key conventions

- **Signals (Vega) / params (Vega-Lite) hold every adjustable value.**
  Deneb's Config tab is deliberately empty.
- **Field-mapping adapter** at the top of the transform pipeline maps
  incoming Power BI field names to fixed canonical names. All downstream
  references use canonical names only.
- **Field parameters** need `isValid()` enumeration, not direct reference.
  Multiple field parameters over the same candidates need an **exclusion
  cascade** (see `patterns.md`).
- **Color overrides**: `isValid(override) ? override : defaultParam`.
  Never reference the raw override field outside this fallback.
- **Stroke, not border** for naming. Raw Power BI fields keep "Border" in
  their names; spec-internal aliases use "Stroke".
- **Font size in points**, converted to px via `* 4 / 3`.
- **DAX files**: first line must be the `Name =` assignment (comments
  after, never before). Use MAX, not SELECTEDVALUE. Don't name a VAR
  after a DAX function. Reserved VAR names found live: `Rank`, `Members`.
- **Delivering a DAX change**: output the COMPLETE measure as one code
  block, never a diff or a "change these lines" summary — measures get
  pasted whole into the Power BI DAX editor, so a partial answer isn't
  usable.

## Design system values (dark canvas)

| Role | Hex |
|---|---|
| Background / stroke default | `#182231` |
| Title tier text | `#FCFCFD` |
| Subtitle tier text | `#A5B4CB` |
| Level 1 fill default | `#2D3B68` |
| Level 2 fill default | `#4555D6` |
| Level 3 fill default | `#8C97E8` |
| Font | Segoe UI |

## Shared data model

All visuals bind to the same Power BI model. 4 candidate columns for
field parameters: `EngagementCodeList`, `AlphaCode`,
`CustodianDisplayName`, `EFileName`. Three field parameter tables:
`MappedParameter` (Level 1 / aster Category), `MappedParameterTwo`
(Level 2 / waffle Category), `MappedParameterThree` (Level 3).

## Raw Vega gotchas (apply to circlepacking + sunburst)

- Symbol mark `size` for circles = `4 * r²` (bounding square area), NOT
  `π * r²` like Vega-Lite.
- Mark `from.transform` inline filter is silently ignored — use a
  top-level filtered dataset instead.
- Event streams: `@markName:click` (no trailing `!`).
- Text marks on top of interactive marks intercept clicks — set
  `interactive: false` on labels.
- Partition transform: non-leaf nodes must contribute `LayoutValue: 0` to
  avoid double-counting angular proportions (doesn't affect `pack`).

## Versioning

Every commit changing a spec or DAX bumps `package.json` version and adds
a dated CHANGELOG entry explaining *why*.

## What NOT to do

- Don't migrate existing Vega-Lite visuals to raw Vega without a reason.
- Don't fabricate realistic sample data.
- Don't claim "confirmed" without actual verification (local tooling or
  live Power BI).
- Don't reuse DAX measure names across visuals (model-wide namespace).
