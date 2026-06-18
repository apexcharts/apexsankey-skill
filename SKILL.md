---
name: apexsankey
description: >
  AI skill for building ApexSankey flow / Sankey diagrams. Use whenever the user
  asks to create, configure, render, or troubleshoot a Sankey diagram, flow
  chart, energy/budget/traffic flow visualization, or multi-stage allocation
  diagram with `apexsankey`. Covers the `{ nodes, edges, options }` data shape,
  layer ordering, edge gradients, path highlighting, custom tooltips, the
  `setLicense` watermark, and framework integration (React / Vue / Angular).
  In React / Vue / Angular projects, prefer the framework wrapper packages
  (`react-apexsankey`, `vue-apexsankey`, `ngx-apexsankey`) over the core API.
metadata:
  author: ApexCharts
  version: "1.0.0"
  library_version: "1.8.0"
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
2. **Every `edge.source` and `edge.target` must reference an existing `node.id`.** Dangling references silently drop the edge.
3. **`node.id` must be unique.** Duplicates collide and one node is silently dropped.
4. **`edge.value` is required and must be > 0.** Zero or negative values produce zero-width bands.
5. **`edge.type` is a category label** (used for grouping & tooltips). Use the same string across edges that should group together.
6. **`render(data)` returns a `SankeyGraphRenderer`** — keep it if you want to call `exportToSvg()` later.
7. **Cycles are not supported.** ApexSankey is a layered DAG; cycles cause layout to fail or produce odd routing.
8. **Call `destroy()`** before unmounting in React / Vue / Angular. Skipping it leaks tooltip DOM and watermark observers.
9. **`tooltipTemplate` is for edges; `nodeTooltipTemplate` is for nodes.** They have different `content` shapes.
10. **`height: 'auto'`** derives height from width at a ~1.6:1 ratio. Use a numeric `height` to lock it.
11. **`onNodeClick` lives in options, not on the data.** It's a `SankeyOptions` field.
12. **Set the license key once** at app startup with `ApexSankey.setLicense('KEY')` to remove the watermark.

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

### Manual layer ordering

By default ApexSankey assigns nodes to layers automatically. To pin layout, pass `options.order` — a list of layers, each a list of bands, each a list of node ids:

```js
sankey.render({
  nodes, edges,
  options: {
    ...sankey.options,
    order: [
      [['a', 'b']],     // layer 0, one band, two nodes
      [['c']],          // layer 1
      [['d']],          // layer 2
    ],
  },
});
```

---

## 3. Top-Level Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `width` | `number \| string` | `'100%'` | Canvas width. |
| `height` | `number \| string` | `'auto'` | `'auto'` ≈ 1.6:1 ratio from width. |
| `viewPortWidth` / `viewPortHeight` | `number` | `800` / `500` | Internal SVG viewport. |
| `spacing` | `number` | `20` | Horizontal gap between node columns (px). |
| `nodeWidth` | `number` | `20` | Width of node rectangles (px). |
| `nodeBorderWidth` | `number` | `1` | Border width (px). |
| `nodeBorderColor` | `string \| null` | `null` | `null` disables the border. |
| `whitespace` | `number` | `0.18` | Fraction (0–1) of vertical space used as margins. Lower = taller nodes. |
| `edgeOpacity` | `number` | `0.4` | Edge fill opacity (0–1). |
| `edgeGradientFill` | `boolean` | `true` | Gradient between source/target colors. |
| `edgeGap` | `number` | `0` | Gap between adjacent edges at a connection point (px). |
| `enableTooltip` | `boolean` | `true` | Edge hover tooltips. |
| `enableToolbar` | `boolean` | `true` | Zoom/pan/export toolbar. |
| `highlightConnectedPath` | `boolean` | `true` | Hover highlights the connected flow. |
| `dimOpacity` | `number` | `0.15` | Opacity for dimmed elements during highlight. |
| `animation` | `{ enabled, duration }` | `{ true, 800 }` | Entrance animation. Disabled when `prefers-reduced-motion` is set. |
| `tooltipTemplate` | `(c: TooltipContent) => string` | built-in | Edge tooltip HTML. |
| `nodeTooltipTemplate` | `(c: NodeTooltipContent) => string` | built-in | Node tooltip HTML. |
| `tooltipTheme` | `'light' \| 'dark'` | — | Preset overrides for tooltip BG/border/font. |
| `onNodeClick` | `(node: SankeyNode) => void` | — | Click callback. |
| `a11y` | `{ enabled, diagramLabel, description }` | `{ enabled: true }` | WCAG 2.1 AA. |

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

sankey.destroy();                     // 4. tear down before unmount
```

**Re-renders**: re-call `sankey.render(data)` with new data. There is no incremental `update()` API — pass a fresh `GraphData` each time.

---

## 5. Public API

| Method | Description |
|---|---|
| `new ApexSankey(el, options?)` | Construct; applies dimensions to `el`. |
| `render(data)` | Paint. Returns a `SankeyGraphRenderer`. **Required** to display anything. |
| `destroy()` | Free DOM, observers, tooltip container. |
| `getInstanceId()` | Unique chart instance id. |
| `ApexSankey.setLicense(key)` | Static; call once at app startup. |
| `graph.exportToSvg()` | (Returned from `render`) Export as SVG file download. |

---

## 6. Tooltips

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

## 7. Pitfalls — ❌ Wrong vs ✅ Correct

### 1. Passing data to the constructor
❌ `new ApexSankey(el, { nodes, edges, ... })` — data is ignored.
✅ `new ApexSankey(el, options); sankey.render({ nodes, edges, options: sankey.options })`.

### 2. Forgetting `render()`
❌ `new ApexSankey(el, opts)` — nothing paints.
✅ Always call `sankey.render(data)`.

### 3. Dangling `source` / `target`
❌ `edges: [{ source: 'x', target: 'y', value: 5 }]` when `'x'` / `'y'` aren't in `nodes`.
✅ Every edge endpoint must match a `node.id`. Dangling edges silently drop.

### 4. Duplicate `node.id`
❌ Two nodes with `id: 'a'` — second is silently dropped, edges may target the wrong one.
✅ Ensure ids are unique.

### 5. `value: 0` or negative
❌ `{ source: 'a', target: 'b', value: 0 }` — zero-width band.
✅ Use a positive number; filter out zero-flow edges in your data layer.

### 6. Cycles
❌ `a → b → a` — layered Sankey can't resolve cycles; layout breaks.
✅ Pre-aggregate or break cycles in your data.

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

---

## 8. Reference Routing Table

| Topic | Reference File |
|---|---|
| Data shape, ordering, custom colors, edge grouping | `references/data-format.md` |
| Tooltips, accessibility, animation, interaction | `references/styling-and-interaction.md` |
| React / Vue / Angular wrappers | `references/framework-wrappers.md` |
