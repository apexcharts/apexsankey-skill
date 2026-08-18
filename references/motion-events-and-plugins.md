# Motion, Events, Plugins & Comparison (1.11+)

Everything on this page arrived in apexsankey 1.11 (except where marked 1.12) and is fully additive: 1.10-era code keeps working unchanged.

## `update(data)`: animated data changes

Call `update(data)` on an already-rendered instance instead of re-rendering:

```js
sankey.render({ nodes, edges: edges2024, options: sankey.options });
// later, on the same instance:
sankey.update({ nodes, edges: edges2025, options: sankey.options });
```

- **Same topology** (same nodes and flows, only different values/positions): nodes and ribbons **spring** smoothly to their new places.
- **Changed topology**: the diagram **morphs** through it: entering flows grow out of their source node, survivors slide to their new places, removed flows retract and dissolve.
- With animation disabled or `prefers-reduced-motion` set, it redraws instantly.
- Returns the `SankeyGraphRenderer`, same as `render()`; output at rest is identical to a fresh `render()`.
- Signature: `update(data: GraphData, transition?: SankeyTransition)`. The optional `SankeyTransition` hint (`growFrom` / `shrinkInto`) anchors entering/leaving nodes to a point; the `drillDown` plugin uses it for its expand/collapse animation. You rarely pass it by hand.

## `destroy()`

`destroy()` tears the chart down: runs plugin teardowns, emits `destroyed`, drops all event handlers, and removes injected DOM (tooltip container). Since 1.12 it also releases everything that outlives a render: the relayout/morph motion driver, the particle layer, drag handlers, and the a11y helper. The motion driver owns a rAF loop, so a chart dropped mid-morph without `destroy()` keeps ticking and redrawing into a detached layer until its springs settle. Always call it before discarding the instance; it is idempotent.

## Events: `on` / `off`

```js
const off = sankey.on('node:click', ({ id, node, originalEvent }) => {
  console.log('clicked', id);
});
// ...later
off();                                   // or sankey.off('node:click', handler)
```

| Event | Payload | When |
|---|---|---|
| `node:click` | `{ id, node, originalEvent }` | A node is clicked. |
| `node:mouseenter` | `{ id, node, originalEvent }` | The pointer enters a node. |
| `node:mouseleave` | `{ id, node, originalEvent }` | The pointer leaves a node. |
| `edge:click` | `{ source, target, value, originalEvent }` | A flow (edge) is clicked. |
| `edge:mouseenter` | `{ source, target, value, originalEvent }` | The pointer enters a flow. |
| `edge:mouseleave` | `{ source, target, value, originalEvent }` | The pointer leaves a flow. |
| `rendered` | none | After the initial render and after each `update()` settles. |
| `destroyed` | none | On `destroy()`. |

`on` returns an unsubscribe function. Events work in chord mode too.

## Plugin API: `use(plugin)`

A `SankeyPlugin` is `{ name, install(ctx) }`. `install` receives `{ chart, on }` and may return a teardown function that runs on `destroy()`. Every subscription made through `ctx.on` is also released automatically on `destroy()`.

```js
sankey.use({
  name: 'click-logger',
  install: ({ chart, on }) => on('node:click', ({ id }) => console.log('clicked', id)),
});
```

`use()` returns the instance; the plugin's `install` runs immediately.

## Built-in plugins

All three ship with the library, importable as named exports (`pathTrace`, `timePlayback`, `drillDown`) or via `ApexSankey.plugins.*`.

### `pathTrace(options?)`

A bright pulse cascades along the connected flow path when a node is picked, ribbon by ribbon, cueing the eye to where the flow goes. Skipped under `prefers-reduced-motion`; Sankey projection only (not chord).

```js
sankey.use(pathTrace({ direction: 'downstream' }));
```

| `PathTraceOptions` | Type | Default | Description |
|---|---|---|---|
| `trigger` | `'click' \| 'hover'` | `'click'` | What starts a trace. |
| `direction` | `'downstream' \| 'upstream' \| 'both'` | `'downstream'` | Which way flow is traced from the picked node. |
| `color` | `string` | `'#ffffff'` | Pulse color. |
| `duration` | `number` | `700` | Milliseconds for a pulse to cross one ribbon. |
| `stagger` | `number` | `220` | Milliseconds added per hop, so the trace cascades outward. |

### `timePlayback(options)`

Steps the diagram through a sequence of data frames, driven by the chart's own `update()`, so topology-stable frames spring smoothly from one to the next. Ships a small control bar (play/pause plus a scrubber).

```js
sankey.render({ ...frames[0], options: sankey.options });
sankey.use(timePlayback({ frames, interval: 1500, loop: true }));
```

| `TimePlaybackOptions` | Type | Default | Description |
|---|---|---|---|
| `frames` | `TimePlaybackFrame[]` | required | Ordered frames (`{ nodes, edges, label? }`). Share topology across frames for a smooth morph. |
| `interval` | `number` | `1600` | Milliseconds each frame is shown before advancing. |
| `autoplay` | `boolean` | `false` | Start playing on install. |
| `loop` | `boolean` | `false` | Loop back to the first frame after the last. |
| `controls` | `boolean` | `true` | Render the built-in control bar. |
| `mount` | `HTMLElement` | after the chart | Where to render the control bar. |

### `drillDown(options)`

Turns a large diagram into super-nodes you expand on demand: click a super-node to reveal its constituent flows, click any child to collapse back. Expanding animates children growing out of the super-node (and shrinking back on collapse). Built on the pure `ApexSankey.collapseGroups(data, groups, collapsed)` transform (also a named export `collapseGroups`), which re-routes and merges the flows of collapsed groups. Sankey projection only; chord has no drill-down in v1.

Seed the initial render with `collapseGroups` so there is no load-time morph:

```js
const detail = { nodes, edges };
const groups = [{ id: 'Fossil', title: 'Fossil', children: ['Coal', 'Gas', 'Oil'] }];
sankey.render({ ...ApexSankey.collapseGroups(detail, groups, ['Fossil']), options: sankey.options });
sankey.use(drillDown({ ...detail, groups }));
```

| `DrillDownOptions` | Type | Default | Description |
|---|---|---|---|
| `nodes` | `SankeyGraphNode[]` | required | The full, detailed leaf nodes before any collapsing. |
| `edges` | `SankeyGraphEdge[]` | required | The full, detailed flows between the leaf nodes. |
| `groups` | `DrillDownGroup[]` | required | Group definitions (`{ id, title, color?, children }`); each collapses its children into one super-node. |
| `expanded` | `string[]` | `[]` | Group ids expanded on install; every other group starts collapsed. |

## Comparison split-view: `ApexSankey.compare(el, config)`

Renders two diagrams of the same flow model side by side (a "before" and an "after"), outlines each flow by how it changed (added / removed / changed, computed by the pure `diffGraphs` transform, also a named export), draws a legend, and links the panels so hovering a node or flow highlights its twin in the other. Returns a `SankeyComparison` handle exposing `before`, `after`, the computed `diff`, and `destroy()`.

```js
const cmp = ApexSankey.compare(el, {
  before: { nodes, edges: edges2024, title: '2024' },
  after:  { nodes, edges: edges2025, title: '2025' },
});
// later: cmp.destroy();
```

| `ComparisonConfig` | Type | Default | Description |
|---|---|---|---|
| `before` | `{ nodes, edges, title? }` | required | The left panel's flow graph and optional heading. |
| `after` | `{ nodes, edges, title? }` | required | The right panel's flow graph and optional heading. |
| `options` | `Partial<SankeyOptions>` | `{}` | Base options shared by both panels (each manages its own width). |
| `syncHighlight` | `boolean` | `true` | Highlight the twin node or flow in the other panel on hover. |
| `showDiff` | `boolean` | `true` | Outline added / removed / changed flows in each panel. |
| `showLegend` | `boolean` | `true` | Render the diff color legend below the panels. |
| `diffColors` | `{ added?, removed?, changed? }` | greens/reds/ambers | Override the diff outline colors. |
