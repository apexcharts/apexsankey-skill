---
name: apexsankey
description: >
  AI skill for building ApexSankey flow / Sankey diagrams. Use whenever the user
  asks to create, configure, render, or troubleshoot a Sankey diagram, flow
  chart, energy/budget/traffic flow visualization, or multi-stage allocation
  diagram with `apexsankey`. Covers the `{ nodes, edges, options }` data shape,
  automatic layer layout, edge gradients, path highlighting, custom tooltips, the
  `setLicense` watermark, animated `update()`, alluvial and chord projections,
  circular links, themes and the family `--apx-*` tokens, the plugin/event API,
  the comparison split-view, and framework integration (React / Vue / Angular).
  In React / Vue / Angular projects, prefer the framework wrapper packages
  (`react-apexsankey`, `vue-apexsankey`, `ngx-apexsankey`) over the core API.
metadata:
  author: ApexCharts
  version: "1.4.2"
  library_version: "1.12.2"
  category: data-visualization
  tags: [sankey, flow, diagram, charts, svg, apexsankey]
  docs: https://apexcharts.com/docs/apexsankey/
  npm: apexsankey
  github: https://github.com/apexcharts/apexsankey
---

# ApexSankey AI Skill

> **Framework wrapper detection — check `package.json` before generating code.**
> - `react` → use **`react-apexsankey`** instead of the core API.
> - `vue` → use **`vue-apexsankey`**.
> - `@angular/core` → use **`ngx-apexsankey`**.
>
> Wrappers handle `destroy()` automatically on unmount, accept reactive props, and forward events as idiomatic framework events. Use the core API directly only when no framework is detected, or when the user explicitly asks for vanilla. See `references/framework-wrappers.md`.

## 1. Critical Rules

1. **Data is `{ nodes, edges, options }`** passed to `render(data)`, **not** to the constructor. Constructor takes `(element, options)`; `render(data)` paints.
2. **Every `edge.source` and `edge.target` must reference an existing `node.id`.** An unknown id does not drop the edge: ApexSankey creates an unlabeled phantom node for it, which still takes up space in the layout.
3. **`node.id` must be unique.** With duplicates, the last declaration overwrites the earlier ones.
4. **`edge.value` is required, numeric, and not negative.** A negative value breaks the band geometry. `0` draws an invisible zero-width flow, and a node fed only by zero flows loses its label. If every value is `0` there is nothing to size the layout by. Filter out empty flows in your data layer.
5. **`edge.type` only tells parallel edges apart.** Two edges with the same `source`, `target`, and `type` collapse into one: the later edge replaces the earlier, and the values are not summed. Merge such edges, or give each a distinct `type`. It is not shown in tooltips or used for styling. The TypeScript types declare it required; at runtime it may be omitted.
6. **`render(data)` returns a `SankeyGraphRenderer`** — keep it if you want to call `exportToSvg()` later.
7. **Use `update(data)` to change data on a live chart** (1.11+). Same-topology updates spring-animate; topology changes morph (entering flows grow out of their source, removed flows retract). `render()` is for the first paint.
8. **Cycles are supported.** Circular links (e.g. `recycle → raw`) render as dashed back-edges returning upstream (1.11+; 1.10 lays them out too, but draws the back-edge solid). Do not break or filter them.
9. **Call `destroy()`** before unmounting in React / Vue / Angular. Skipping it leaks tooltip DOM and watermark observers; since 1.12 it also releases the motion driver's rAF loop, the particle layer, drag handlers, and the a11y helper (a chart dropped mid-morph would otherwise keep ticking into detached DOM).
10. **`tooltipTemplate` is for edges; `nodeTooltipTemplate` is for nodes.** They have different `content` shapes.
11. **`height: 'auto'`** derives height from width at a ~1.6:1 ratio. Use a numeric `height` to lock it.
12. **`onNodeClick` lives in options, not on the data.** For edges (and richer payloads) use the 1.11+ event bus: `sankey.on('edge:click', handler)`.
13. **Set the license key once** at app startup with `ApexSankey.setLicense('KEY')` to remove the watermark.
14. **Same `{ nodes, edges }` model, three projections** (1.11+): layered Sankey (default), alluvial (`ApexSankey.buildAlluvialData(...)` plus `axisTitles`), chord (`type: 'chord'`). In chord mode self-loops and zero-value edges are dropped, and parallel edges are kept rather than collapsed.
15. **A node with no edges is not drawn.** The layout builds its node set from the edges and attaches the declared node data only to ids an edge created.
16. **Node placement is automatic.** Columns come from rank assignment and the vertical order from crossing minimization. There is no public option to pin the order in 1.12: an `order` array in `options` is not read. `draggableNodes: true` lets a reader reposition nodes until the next `render()` / `update()`.

---

## 2. Data Format

```ts
interface GraphData {
  nodes: { id: string; title: string; color?: string }[];
  edges: { source: string; target: string; value: number; type: string }[];
  options: SankeyOptions;            // typically just `sankey.options`
}
```

Minimal example:

```js
import { ApexSankey } from 'apexsankey';

const sankey = new ApexSankey(document.getElementById('chart'), {
  width: 800,
  height: 500,
  nodeWidth: 24,
  spacing: 60,
});

sankey.render({
  nodes: [
    { id: 'a', title: 'Source A' },
    { id: 'b', title: 'Source B' },
    { id: 'c', title: 'Hub'      },
    { id: 'd', title: 'Sink'     },
  ],
  edges: [
    { source: 'a', target: 'c', value: 10, type: 'flow' },
    { source: 'b', target: 'c', value:  6, type: 'flow' },
    { source: 'c', target: 'd', value: 16, type: 'flow' },
  ],
  options: sankey.options,           // pass the resolved options back in
});
```

### Layer layout

ApexSankey places nodes automatically: each node's column comes from rank assignment, and the order within a column from crossing minimization. 1.12 has no public option to pin that order. An `order` array passed in `options` is ignored, so do not generate one.

---

## 3. Top-Level Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `width` | `number \| string` | `'100%'` | Canvas width. |
| `height` | `number \| string` | `'auto'` | `'auto'` ≈ 1.6:1 ratio from width. |
| `type` | `'sankey' \| 'chord'` | `'sankey'` | Projection (1.11+). `'chord'` draws a radial diagram: nodes as arcs on a ring, flows as ribbons across the interior. See `references/data-format.md`. |
| `orientation` | `'horizontal' \| 'vertical'` | `'horizontal'` | Flow direction (1.11+). `'horizontal'`: ranks in columns, flows left→right. `'vertical'`: ranks in rows, flows top→bottom. |
| `axisTitles` | `string[]` | none | Per-rank axis titles (1.11+), drawn above each column (or beside each row when vertical). Index `i` labels rank `i`. Pair with `buildAlluvialData` for alluvial diagrams. |
| `arcCornerRadius` | `number` | `6` | Chord only (1.11+). Corner radius (px) of each node arc's outer corners; `0` gives sharp corners; clamped to the ring band width. |
| `theme` | `string \| SankeyTheme` | none | Named built-in (`'light'`, `'dark'`, `'midnight'`, `'mint'`, `'sunset'`), a name registered via `ApexSankey.registerTheme`, or an inline `SankeyTheme` (1.11+). Seeds visual defaults; explicit options still win. |
| `nodePalette` | `string[]` | none | Ordered fills cycled across nodes without their own `color` (1.11+). Overrides the built-in palette; a `theme` sets this for you. |
| `viewPortWidth` / `viewPortHeight` | `number` | `800` / `500` | Internal SVG viewport. |
| `spacing` | `number` | `20` | Horizontal gap between node columns (px). |
| `nodeWidth` | `number` | `20` | Width of node rectangles (px). |
| `nodeBorderWidth` | `number` | `1` | Border width (px). |
| `nodeBorderColor` | `string \| null` | `null` | `null` disables the border. |
| `whitespace` | `number` | `0.18` | Fraction (0–1) of vertical space used as margins. Lower = taller nodes. |
| `edgeOpacity` | `number` | `0.4` | Edge fill opacity (0–1). |
| `edgeGradientFill` | `boolean` | `true` | Gradient between source/target colors. |
| `edgeGap` | `number` | `0` | Gap between adjacent edges at a connection point (px). |
| `draggableNodes` | `boolean` | `false` | Reposition nodes by dragging (mouse, touch, or pen) (1.11+). Connected flows follow live; the manual position holds until the next `render()`/`update()`. |
| `particleFlow` | `boolean` | `false` | Animate particles along each ribbon, density proportional to value (1.11+). Decorative; skipped under `prefers-reduced-motion`. |
| `enableTooltip` | `boolean` | `true` | Edge hover tooltips. |
| `enableToolbar` | `boolean` | `true` | Zoom/pan/export toolbar. |
| `highlightConnectedPath` | `boolean` | `true` | Hover gives a one-hop highlight preview; since 1.11 clicking a node or flow pins an isolate of its full upstream/downstream path (click again, or another element, to change/release). |
| `dimOpacity` | `number` | `0.2` | Opacity for dimmed (unrelated) elements when path highlighting is active. |
| `animation` | `{ enabled, duration }` | `{ true, 800 }` | Entrance animation. Disabled when `prefers-reduced-motion` is set. |
| `tooltipTemplate` | `(c: TooltipContent) => string` | built-in | Edge tooltip HTML. |
| `nodeTooltipTemplate` | `(c: NodeTooltipContent) => string` | built-in | Node tooltip HTML. |
| `tooltipTheme` | `'light' \| 'dark'` | — | Preset overrides for tooltip BG/border/font. |
| `onNodeClick` | `(node: SankeyNode) => void` | — | Click callback. |
| `a11y` | `{ enabled, diagramLabel, description }` | `{ enabled: true }` | WCAG 2.1 AA. |
| `locale` | `{ direction, messages }` | `{ direction: 'ltr' }` | Localization / RTL. `direction: 'ltr' \| 'rtl' \| 'auto'` mirrors the diagram and sets `dir="rtl"`; `messages` is a `Partial<SankeyMessages>` overriding the ARIA strings. See `references/styling-and-interaction.md`. |

---

## 4. Lifecycle

```js
import { ApexSankey } from 'apexsankey';

ApexSankey.setLicense('KEY');         // optional, once at app startup

const sankey = new ApexSankey(el, {   // 1. construct (sets dimensions)
  width: 800, height: 500,
});

const graph = sankey.render(data);    // 2. paint — REQUIRED

graph.exportToSvg();                  // 3. (optional) export from the renderer

sankey.update(nextData);              // 4. (optional) animate to new data

sankey.destroy();                     // 5. tear down before unmount
```

**Data changes** (1.11+): call `sankey.update(data)` on the live instance. Same topology (same nodes and flows, different values): nodes and ribbons spring to their new places. Different topology: entering flows grow out of their source node, survivors slide, removed flows retract and dissolve. With animation disabled or `prefers-reduced-motion` it redraws instantly. On 1.10 and earlier there is no `update()`: re-call `sankey.render(data)`.

---

## 5. Public API

| Method | Description |
|---|---|
| `new ApexSankey(el, options?)` | Construct; applies dimensions to `el`. |
| `render(data)` | Paint. Returns a `SankeyGraphRenderer`. **Required** to display anything. |
| `update(data, transition?)` | (1.11+) Animate an already-rendered instance to new data. Returns the renderer. See `references/motion-events-and-plugins.md`. |
| `on(event, handler)` | (1.11+) Subscribe to a typed instance event (`node:*`, `edge:*`, `rendered`, `destroyed`). Returns an unsubscribe function. |
| `off(event, handler)` | (1.11+) Remove a handler registered with `on`. |
| `use(plugin)` | (1.11+) Install a `SankeyPlugin`; its teardown runs on `destroy()`. Returns the instance. |
| `destroy()` | Run plugin teardowns, emit `destroyed`, drop handlers, free DOM/observers/tooltip container; since 1.12 also stops the motion driver, particle layer, drag handlers, and a11y helper. |
| `getInstanceId()` | Unique chart instance id. |
| `ApexSankey.setLicense(key)` | Static; call once at app startup. |
| `ApexSankey.registerTheme(name, theme)` | Static (1.11+); register a named `SankeyTheme` for the `theme` option. |
| `ApexSankey.buildAlluvialData(input)` | Static (1.11+); turn records-across-dimensions into `{ nodes, edges }` (also a named export `buildAlluvialData`). |
| `ApexSankey.collapseGroups(data, groups, collapsed)` | Static (1.11+); pure transform collapsing groups into super-nodes (also a named export). |
| `ApexSankey.compare(el, config)` | Static (1.11+); before/after split view with structural diff and synced hover. Returns `{ before, after, diff, destroy() }`. |
| `ApexSankey.plugins` | Static (1.11+); `{ pathTrace, timePlayback, drillDown }` (also named exports). |
| `graph.exportToSvg()` | (Returned from `render`) Export as SVG file download. |

---

## 6. Projections, Themes, Plugins (1.11+)

One `{ nodes, edges }` model, three views:

```js
// Alluvial: categorical records across dimensions
const input = {
  dimensions: ['2019', '2022', '2025'],
  records: [
    { values: { 2019: 'Free', 2022: 'Pro', 2025: 'Pro' } },
    { values: { 2019: 'Pro', 2022: 'Pro', 2025: 'Team' }, value: 3 },
  ],
};
const sankey = new ApexSankey(el, { axisTitles: input.dimensions });
sankey.render({ ...ApexSankey.buildAlluvialData(input), options: sankey.options });

// Chord: dense many-to-many relationships
const chord = new ApexSankey(el, { type: 'chord' });
chord.render({ nodes, edges, options: chord.options });

// Themes
new ApexSankey(el, { theme: 'dark' });   // light | dark | midnight | mint | sunset
ApexSankey.registerTheme('acme', { nodePalette: ['#ff5a5f', '#087f8c'] });

// Events + plugins
sankey.on('edge:click', ({ source, target, value }) => { /* ... */ });
sankey.use(ApexSankey.plugins.pathTrace({ direction: 'downstream' }));
```

Chord mode reuses tooltips, node/edge events, and the palette; Sankey-only features (orientation, RTL mirror, animated relayout, node dragging, particle flow, drill-down) do not apply to it. Details: `references/data-format.md` (projections), `references/styling-and-interaction.md` (themes, tokens), `references/motion-events-and-plugins.md` (update, events, plugins, compare).

---

## 7. Tooltips

### Edge tooltip — `tooltipTemplate({ source, target, value })`

```js
new ApexSankey(el, {
  tooltipTemplate: ({ source, target, value }) => `
    <div style="display:flex;align-items:center;gap:6px;">
      <span style="width:10px;height:10px;background:${source.color};display:inline-block;"></span>
      <strong>${source.title}</strong> → <strong>${target.title}</strong>
      <span>: ${value.toLocaleString()}</span>
    </div>`,
});
```

### Node tooltip — `nodeTooltipTemplate({ node, value })`

```js
new ApexSankey(el, {
  nodeTooltipTemplate: ({ node, value }) =>
    `<strong>${node.title}</strong><br/>Total: ${value.toLocaleString()}`,
});
```

### `tooltipTheme` shortcut

```js
new ApexSankey(el, { tooltipTheme: 'dark' });
```

Equivalent to setting `tooltipBGColor`, `tooltipBorderColor`, `tooltipFontColor` to the dark preset.

---

## 8. Pitfalls: ❌ Wrong vs ✅ Correct

### 1. Passing data to the constructor
❌ `new ApexSankey(el, { nodes, edges, ... })` — data is ignored.
✅ `new ApexSankey(el, options); sankey.render({ nodes, edges, options: sankey.options })`.

### 2. Forgetting `render()`
❌ `new ApexSankey(el, opts)` — nothing paints.
✅ Always call `sankey.render(data)`.

### 3. Dangling `source` / `target`
❌ `edges: [{ source: 'x', target: 'y', value: 5 }]` when `'x'` / `'y'` aren't in `nodes`.
✅ Every edge endpoint must match a `node.id`. An unknown id becomes an unlabeled phantom node in the diagram.

### 4. Duplicate `node.id`
❌ Two nodes with `id: 'a'`: the last declaration overwrites the earlier ones.
✅ Ensure ids are unique.

### 5. `value: 0` or negative
❌ `{ source: 'a', target: 'b', value: -5 }` breaks the band geometry; `value: 0` draws an invisible flow.
✅ Use a positive number; filter out zero-flow edges in your data layer.

### 6. Treating cycles as an error
❌ Filtering out circular links, or reporting dashed back-edges as a rendering bug.
✅ Cyclic links are supported and intentionally render as dashed ribbons returning upstream (1.11+).

### 7. Not destroying on unmount (React / Vue)
❌ — tooltip DIVs and observers leak.
✅ `useEffect(() => { const s = new ApexSankey(ref.current, opts); s.render(data); return () => s.destroy(); }, [])`.

### 8. Mixing tooltip templates
❌ Returning the edge template's HTML from `nodeTooltipTemplate` — `content` shape differs.
✅ Edge template gets `{ source, target, value }`; node template gets `{ node, value }`.

### 9. Forgetting `options` in `render(data)`
❌ `sankey.render({ nodes, edges })` — `options` is required on `GraphData`.
✅ `sankey.render({ nodes, edges, options: sankey.options })`.

### 10. License watermark in production
❌ Watermark visible.
✅ `ApexSankey.setLicense('KEY')` once before any `new ApexSankey(...)`.

### 11. Full re-render for a data change (1.11+)
❌ `sankey.destroy(); new ApexSankey(el, opts).render(next)` on every data tick: loses animation and rebuilds everything.
✅ `sankey.update(next)`: springs or morphs to the new data on the live instance.

### 12. Expecting Sankey-only features in chord mode
❌ `{ type: 'chord', draggableNodes: true, particleFlow: true, orientation: 'vertical' }` and wondering why nothing changes.
✅ Chord reuses tooltips, node/edge events, and the palette only. Orientation, RTL mirror, animated relayout, dragging, particles, and drill-down apply to the Sankey projection.

### 13. Parallel edges with the same `type`
❌ `{ source: 'a', target: 'b', value: 10, type: 'x' }` and `{ source: 'a', target: 'b', value: 1, type: 'x' }`: only the second survives, so the diagram shows 1, not 11.
✅ Merge them into one edge with `value: 11`, or give each a distinct `type` if they really are separate flows.

### 14. Pinning node order with `options.order`
❌ `options: { ...sankey.options, order: [[['a', 'b']], [['c']]] }`: the array is not read, and the layout stays automatic.
✅ Let the layout place nodes, or enable `draggableNodes` for manual adjustment.

> Since 1.10.0, license keys are ECDSA P-256 signature-verified. Unsigned (legacy) keys stay valid until 2027-07-31; re-issue a signed key before then.

---

## 9. Reference Routing Table

| Topic | Reference File |
|---|---|
| Data shape, layer layout, custom colors, parallel edges, cyclic links, alluvial and chord projections | `references/data-format.md` |
| Tooltips, accessibility, animation, interaction, themes, `--apx-*` tokens, draggable nodes, particle flow | `references/styling-and-interaction.md` |
| `update()` motion, events, plugin API, built-in plugins, comparison split-view, `destroy()` | `references/motion-events-and-plugins.md` |
| React / Vue / Angular wrappers | `references/framework-wrappers.md` |
