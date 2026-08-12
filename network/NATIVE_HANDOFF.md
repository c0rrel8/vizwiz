# Native Network Map — Project Handoff

Comprehensive lessons from the Deneb-based network diagram attempt
(vizwiz/network) and architecture reference from the native 3D scatter
visual (c0rrel8/3dscatter). Use this to build a native Power BI custom
visual for the network map.

## Why native, not Deneb

Vega's `force` transform does not work in Deneb's runtime environment.
After 9 iterations (v0.1.0–v0.1.9), every spec that included a `force`
transform produced marks at extreme off-screen coordinates — visible only
as scroll bars, no errors in the Deneb log output, no warnings. The
behavior is consistent and reproducible across different force
configurations (weak/strong repulsion, static/dynamic iterations,
pre-positioned nodes, post-force clamping). Without force-directed layout,
a network diagram is functionally a less-featured dendrogram.

### What works in Deneb (confirmed)

- `stratify` + `tree` transforms (hierarchical layout)
- `treelinks` for parent–child edges
- Direct edge derivation via `lookup` + `flatten`
- Circular and grid layouts (manual positioning via formulas)
- Click selection, neighbor highlighting, cross-highlighting
- All three hierarchy levels with exclusion cascade

### What does NOT work in Deneb

- `force` transform — marks render off-screen, no errors
- Draggable nodes — `mark.on` + `modify` requires a dataset name, not a
  signal name; using a signal silently breaks the entire mark (no error)
- Any physics simulation that needs iterative settling

### Deneb debugging limitations discovered

- **Data tab** shows only the raw bound Power BI data source, NOT
  intermediate Vega datasets — useless for pipeline debugging
- **Signals tab** shows signal values correctly (useful)
- **Logs tab** shows no errors even when force produces garbage positions
- `mark.on` with wrong `modify` target: silent failure, no error
- `from.transform` inline filter on marks: silently ignored — must use a
  top-level filtered dataset instead

## Data model (reuse from vizwiz)

All vizwiz visuals bind to the same Power BI model.

### Field parameters

4 candidate columns: `EngagementCodeList`, `AlphaCode`,
`CustodianDisplayName`, `EFileName`.

3 field parameter tables with exclusion cascade:
- `MappedParameter` — Level 1 (picks first valid)
- `MappedParameterTwo` — Level 2 (excludes L1 pick)
- `MappedParameterThree` — Level 3 (excludes L1 + L2 picks)

Field parameters use `isValid()` enumeration. Multiple field parameters
over the same candidates need the exclusion cascade (see
`vizwiz/patterns.md`).

### Hierarchy construction

From the 3 field parameter levels, build a 3-level hierarchy:
1. Stratify from flat records: each row has L1, L2, L3 values
2. Nodes at each level aggregate measures (count, sum, etc.)
3. Edges connect parent→child (L1→L2, L2→L3) and optionally L3→L3
   siblings based on shared attributes

### Dual data mode (designed but only hierarchy confirmed working)

- **Hierarchy mode**: derives nodes + edges from the 3-level field
  parameter model (above)
- **Edge-list mode**: separate Source/Target columns for explicit edges
  (planned, not yet tested with real data)

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

Color overrides: `isValid(override) ? override : default`. Never
reference the raw override field outside this fallback pattern.

Font size in points, converted to px via `* 4 / 3`.

"Stroke" not "border" for spec-internal naming. Raw Power BI fields keep
"Border" in their names; spec aliases use "Stroke".

## 3dscatter reference architecture

The `c0rrel8/3dscatter` project is the template for native Power BI
visuals. Key decisions and patterns to reuse:

### Toolchain

- Node.js 20, TypeScript 5.5.4
- `powerbi-visuals-api ~5.11.0`
- `powerbi-visuals-tools 7.0.3` (pbiviz CLI)
- `powerbi-visuals-utils-formattingmodel 6.0.4`
- `d3 7.9.0` (used for scales/layouts — network map will use d3-force)
- Windows-specific wrapper scripts for Node 20 runtime (npm.cmd/npx.cmd
  instead of npm/npx to avoid PowerShell execution policy issues)

### Project structure

```
visual/
  capabilities.json    — data roles, formatting objects, dataViewMappings
  pbiviz.json          — visual metadata
  package.json         — dependencies + scripts
  src/
    visual.ts          — IVisual implementation (constructor, update,
                         getFormattingModel, destroy)
    data.ts            — data extraction from DataView
    scene.ts           — math/projection utilities
    settings.ts        — FormattingSettingsModel
    webglScene.ts      — WebGL renderer
    webglScene.types.ts
  scripts/
    bootstrap.ps1      — one-command setup
    run-pbiviz.ps1     — wrapper for pbiviz commands
    run-tests.mjs      — test runner
```

### capabilities.json patterns

```json
{
  "dataRoles": [
    { "name": "x", "kind": "Measure" },
    { "name": "category", "kind": "Grouping" },
    { "name": "tooltip", "kind": "GroupingOrMeasure" }
  ],
  "dataViewMappings": [{
    "categorical": {
      "categories": {
        "for": { "in": "category" },
        "dataReductionAlgorithm": { "window": { "count": 10000 } }
      },
      "values": {
        "select": [{ "bind": { "to": "x" } }],
        "dataReductionAlgorithm": { "window": { "count": 10000 } }
      }
    }
  }],
  "supportsHighlight": true,
  "supportsKeyboardFocus": true,
  "supportsMultiVisualSelection": true,
  "supportsLandingPage": true,
  "tooltips": {
    "supportedTypes": { "default": true, "canvas": true },
    "supportEnhancedTooltips": true
  }
}
```

### IVisual implementation patterns (from visual.ts)

**Constructor setup:**
- `host.createSelectionManager()` — selection + context menu
- `host.tooltipService` — tooltip integration
- `host.colorPalette` — category colors + high-contrast mode
- `host.createLocalizationManager()` — i18n
- Create root container element, attach pointer/wheel/keyboard listeners

**update() method:**
- Guard on `options.type` — handle `Data`, `Resize`, `Style` differently
- Extract data from `options.dataViews[0].categorical`
- Build `selectionId` per data point via `host.createSelectionIdBuilder()`
  with `.withCategory(categories, index).createSelectionId()`
- Apply conditional formatting via `dataViewWildcard.createDataViewWildcardSelector()`
- Handle `fetchMoreData` for large datasets: check
  `dataView.metadata.segment`, call `host.fetchMoreData(true)`
- Re-render on every update cycle

**Selection / interaction:**
- `selectionManager.select(selectionId, multiSelect)` — multiSelect when
  ctrl/meta key held
- `selectionManager.registerOnSelectCallback()` — respond to external
  selections
- `selectionManager.showContextMenu(selectionId, event)` — right-click
- `selectionManager.clear()` on background click (configurable)

**Tooltip integration:**
- `host.tooltipService.show({ coordinates, isTouchEvent, dataItems, identities })`
- `host.tooltipService.hide({ immediately, isTouchEvent })`
- Data items: `{ displayName, value }` arrays

**Category colors:**
- `host.colorPalette.getColor(categoryValue).value` — gets consistent
  color per category
- Conditional color formatting: use `dataViewWildcard` selectors in
  `getFormattingModel()` so users can override per-instance colors

**Animation loop:**
- `requestAnimationFrame` for continuous rendering (auto-rotate, force
  simulation settling)
- Cancel via stored animation frame ID in `destroy()`

**Performance:**
- `limitScenePoints()` — downsample with highlight priority (highlighted
  points never dropped)
- `maxRenderedPoints` user-configurable cap
- `downsampleDenseScenes` toggle

### scene.ts patterns

- `summarizeScene()` — normalize all point positions to [-1, 1] range
- `projectPoint()` — 3D→2D with orbit camera (yaw, pitch, distance),
  perspective division
- `limitScenePoints()` — stride-based downsampling that preserves
  highlighted points
- `extent()` — compute [min, max] range
- `normalize()` — map value into range, handle degenerate (min === max)

### Formatting settings

Use `FormattingSettingsService` from
`powerbi-visuals-utils-formattingmodel`. Define setting classes extending
`FormattingSettingsModel` with slices for each property. The
`getFormattingModel()` method returns the model for Power BI's format pane.

## Network map — recommended architecture

### Data roles

```
Source Category (Grouping)     — node identifier for edge source
Target Category (Grouping)     — node identifier for edge target
Node Label (Grouping)          — display name (optional, falls back to ID)
Edge Weight (Measure)          — numeric weight for edges
Node Size (Measure)            — numeric size for nodes
Tooltip (GroupingOrMeasure)    — additional tooltip fields
```

For hierarchy mode, also support field parameters via:
```
Level 1 (Grouping)
Level 2 (Grouping)
Level 3 (Grouping)
Measure A (Measure)            — node sizing / tooltip value
```

### Rendering approach

Use **Canvas 2D** (not WebGL) — network diagrams are 2D, don't need 3D
projection, and Canvas 2D is simpler to implement and debug. Use
**d3-force** for layout simulation (it runs in JS, not in Vega's
transform pipeline, so Deneb's incompatibility doesn't apply).

d3-force layout:
```typescript
import { forceSimulation, forceLink, forceManyBody, forceCenter, forceCollide }
  from "d3-force";

const simulation = forceSimulation(nodes)
  .force("link", forceLink(edges).id(d => d.id).distance(linkDistance))
  .force("charge", forceManyBody().strength(repulsionStrength))
  .force("center", forceCenter(width / 2, height / 2))
  .force("collide", forceCollide(nodeRadius));
```

Run simulation ticks in `requestAnimationFrame` loop for animated
settling, or call `simulation.tick(300)` for instant static layout.

### Formatting objects (capabilities.json)

Reuse the pattern from 3dscatter — group related properties:

- **Layout**: algorithm (force/circular/grid/hierarchical), force presets,
  link distance, repulsion, iterations
- **Nodes**: default color, size mode (fixed/metric/degree), min/max
  radius, opacity, label visibility
- **Edges**: color, thickness mode (fixed/weighted), opacity, style
  (straight/curved), arrowheads (directed/undirected)
- **Selection**: selected opacity, unselected opacity, highlight opacity,
  clear on background click
- **Legend**: show/hide, position, colors
- **Performance**: max rendered nodes, max rendered edges, downsample
  toggle
- **Camera** (if pan/zoom): zoom sensitivity, pan sensitivity, fit-to-view

### Interaction features to implement

1. **Click-to-select** with `SelectionManager` — highlight clicked node
   and its neighbors, dim everything else
2. **Cross-filtering** — selected nodes filter other visuals via
   `selectionManager.select()`
3. **Cross-highlighting** — respond to highlights from other visuals via
   `__highlight` values
4. **Draggable nodes** — pointer drag pins a node position, release unpins
   (d3-force supports this natively via `fx`/`fy` properties)
5. **Tooltips** on both nodes and edges via `host.tooltipService`
6. **Context menu** via `selectionManager.showContextMenu()`
7. **Zoom/pan** — wheel zoom + pointer drag on background
8. **Keyboard navigation** — tab between nodes, enter to select
   (`supportsKeyboardFocus: true`)

### Node sizing modes

- **Fixed**: all nodes same radius (user-configurable)
- **Metric**: radius proportional to Measure A value (normalize to
  min/max radius range)
- **Degree**: radius proportional to connection count (compute from edge
  list)

### Edge weight channels

Three independent toggles, each driven by Edge Weight measure:
- **Thickness**: strokeWidth proportional to weight
- **Opacity**: alpha proportional to weight
- **Color**: sequential color scale mapped to weight

### Multi-edge handling

When multiple edges connect the same node pair, auto-curve with
perpendicular offset to prevent overlap:
```
offset = (edgeIndex - (edgeCount - 1) / 2) * spacing
```
Apply offset perpendicular to the edge midpoint.

## Process lessons from 3dscatter (PROCESS_REFINEMENT.md)

- Separate project-specific lessons from general process corrections
- On Windows PowerShell, use `where.exe node`, not `where node`
- Use `npm.cmd` and `npx.cmd` to avoid execution policy failures
- Move repo commands behind explicit wrapper scripts that invoke the
  intended runtime directly
- When a platform has an official extensibility pattern, prefer that over
  custom UI — for Power BI conditional formatting, align with Microsoft's
  wildcard-selector and `ConstantOrRule` model
- Consolidate host validation into one script so the user isn't a
  copy/paste transport layer

## DAX naming

DAX measure names are model-wide. Prefix network measures with "Network"
to avoid collision with existing visuals:
- `NetworkLevelFixedColorFill`
- `NetworkLevelFixedColorStroke`
- etc.

Don't name a VAR after a DAX function. First line of DAX file must be
the `Name =` assignment (comments after, never before). Use MAX, not
SELECTEDVALUE.

## Files to reference

| File | Purpose |
|---|---|
| `vizwiz/network/network.vg.json` | Deneb spec (v0.1.9, broken — force doesn't work). v0.1.7/v0.1.8 are the last working versions (circular/hierarchical layout, no force). Read for data pipeline + interaction logic. |
| `vizwiz/network/CHANGELOG.md` | Version history of Deneb attempts |
| `vizwiz/patterns.md` | Vega/Deneb architectural patterns, exclusion cascade, field parameter model |
| `vizwiz/designsystem.md` | Colors, fonts, sizing conventions |
| `vizwiz/HANDOFF.md` | Session history for the broader vizwiz library |
| `c0rrel8/3dscatter/visual/src/visual.ts` | Reference native visual implementation (~1200 lines) |
| `c0rrel8/3dscatter/visual/src/scene.ts` | 3D math utilities (normalize, project, downsample) |
| `c0rrel8/3dscatter/visual/capabilities.json` | Reference capabilities structure |
| `c0rrel8/3dscatter/visual/package.json` | Toolchain versions and scripts |
| `c0rrel8/3dscatter/PROCESS_REFINEMENT.md` | Windows/sandbox process lessons |

## Summary

Build a native Power BI custom visual using `powerbi-visuals-api ~5.11.0`,
Canvas 2D rendering, and `d3-force` for layout. Follow the 3dscatter
architecture for project structure, formatting settings, selection,
tooltips, and data handling. Reuse the vizwiz data model (3-level field
parameters with exclusion cascade) for hierarchy mode, and add explicit
Source/Target roles for edge-list mode. The Deneb spec (v0.1.7/v0.1.8)
contains validated interaction logic (selection, neighbor highlighting,
cross-highlighting, edge weight channels) that can be ported to
TypeScript.
