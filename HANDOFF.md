# Handoff notes — Power BI Deneb Visuals library

Read this first, then `CLAUDE.md`, `patterns.md`, and `designsystem.md`.

## Repository: c0rrel8/vizwiz (GitHub)

All work is on `main`. No sync blockers — GitHub push works.

## What's built (6 visual directories)

| Dir | Engine | Status | Notes |
|---|---|---|---|
| `aster/` | Vega-Lite | Mature, confirmed live | Radial wedge chart (angle + radius) |
| `waffle/` | Vega-Lite | Mature, confirmed live | Grid-of-shapes proportional chart |
| `circlepacking/` | Raw Vega | Confirmed live | Nested circles, 3-level hierarchy |
| `sunburst/` | Raw Vega | Confirmed live | Concentric arc rings, 3-level hierarchy |
| `dendrogram/` | Raw Vega | Confirmed live | Tree diagram, 3-level hierarchy |
| `network/` | Raw Vega | **Dead end** | Force transform incompatible with Deneb — see `network/NATIVE_HANDOFF.md` |

### Network diagram — key finding

Vega's `force` transform does not work in Deneb. After 9 iterations
(v0.1.0–v0.1.9), every spec with a force transform produced marks at
off-screen coordinates with no errors. Without force layout, a network
diagram is just a less-featured dendrogram. The `network/` directory
contains:
- The broken spec (v0.1.9) as a reference for data pipeline and
  interaction logic
- `NATIVE_HANDOFF.md` — comprehensive handoff for building a native
  Power BI custom visual instead, referencing the `c0rrel8/3dscatter`
  project architecture

## Shared data model

All visuals bind to the same Power BI model.

4 candidate columns for field parameters: `EngagementCodeList`,
`AlphaCode`, `CustodianDisplayName`, `EFileName`.

3 field parameter tables with exclusion cascade:
- `MappedParameter` — Level 1 / aster Category
- `MappedParameterTwo` — Level 2 / waffle Category
- `MappedParameterThree` — Level 3

## Open items

1. **Rank-based color gradient DAX crashes at high cardinality** —
   O(N²) rank pattern. Needs RANKX or calculated column approach.
   Circle packing sidesteps this with solid per-level colors.
2. **`MappedParameterThree`** may still need to be created in the
   report model — DAX template at
   `dendrogram/categorythreefieldparameter.dax`.

## Key conventions (see patterns.md + designsystem.md for full versions)

- Signals/params hold every adjustable value. Deneb Config tab is empty.
- Field-mapping adapter at top of transform pipeline maps Power BI names
  to canonical names. All downstream refs use canonical names only.
- Color overrides: `isValid(override) ? override : defaultParam`.
- "Stroke" not "border" for spec-internal naming.
- Font size in points, converted to px via `* 4 / 3`.
- DAX: first line = `Name =` assignment. Use MAX not SELECTEDVALUE.
  Don't name a VAR after a DAX function. Prefix measures per visual.
- Every spec/DAX change bumps version + adds dated CHANGELOG entry.
- Nothing claimed "confirmed" without actual verification.

## Deneb gotchas (accumulated across all sessions)

- **Provider is fixed at visual creation** — cannot switch Vega-Lite ↔
  Vega after creating the Deneb visual. Create a new visual with the
  correct Provider before pasting.
- **`force` transform**: marks render off-screen, no errors. Unusable.
- **`mark.on` + `modify`**: expects a dataset name, not a signal name.
  Using a signal silently breaks the entire mark.
- **`from.transform` inline filter**: silently ignored on marks. Use a
  top-level filtered dataset instead.
- **Data tab**: shows only raw bound data, NOT intermediate datasets.
- **Symbol mark `size`** for circles = `4 * r²` (bounding square area),
  NOT `π * r²` like Vega-Lite.
- **Text marks on interactive marks** intercept clicks — set
  `interactive: false` on labels.
- **Partition transform**: non-leaf nodes must contribute
  `LayoutValue: 0` to avoid double-counting angular proportions.
- **Event streams**: `@markName:click` (no trailing `!`).

## Design system (dark canvas)

| Role | Hex |
|---|---|
| Background / stroke default | `#182231` |
| Title tier text | `#FCFCFD` |
| Subtitle tier text | `#A5B4CB` |
| Level 1 fill default | `#2D3B68` |
| Level 2 fill default | `#4555D6` |
| Level 3 fill default | `#8C97E8` |
| Font | Segoe UI |

## Related projects

- **c0rrel8/3dscatter** — Native Power BI custom visual (WebGL 3D
  scatter). Reference architecture for building native visuals. See
  `network/NATIVE_HANDOFF.md` for how to use it as a template for the
  network map.
