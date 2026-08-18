# Styling, Tooltips, Interaction & Accessibility

## Visual options at a glance

| Option | Default | Notes |
|---|---|---|
| `edgeOpacity` | `0.4` | Edge band fill opacity. |
| `edgeGradientFill` | `true` | Gradient between source / target colors. |
| `edgeGap` | `0` | Gap (px) between adjacent edges at a node connection. |
| `nodeWidth` | `20` | Width of node rectangles (px). |
| `nodeBorderColor` | `null` | `null` disables. |
| `nodeBorderWidth` | `1` | px |
| `nodePalette` | none | (1.11+) Ordered fills cycled across nodes without their own `color`. Overrides the built-in palette; a `theme` sets this for you. |
| `theme` | none | (1.11+) Named or inline theme seeding coordinated defaults. See [Themes](#themes-111) below. |
| `orientation` | `'horizontal'` | (1.11+) `'vertical'` lays ranks in rows, flows top→bottom. |
| `whitespace` | `0.18` | Vertical-margin fraction. Lower = taller nodes. |
| `spacing` | `20` | Horizontal gap between layers (px). |
| `viewPortWidth` / `viewPortHeight` | `800` / `500` | Internal SVG viewport. |
| `fontFamily` / `fontSize` / `fontWeight` / `fontColor` | system / `'14px'` / `'400'` / `'#212121'` | Node label typography. |

## Tooltips

Edge and node tooltips have **different `content` shapes** — don't mix them.

### Edge tooltip — `tooltipTemplate({ source, target, value })`

```js
const sankey = new ApexSankey(el, {
  tooltipTemplate: ({ source, target, value }) => `
    <div style="display:flex;align-items:center;gap:8px;padding:6px 10px;">
      <span style="width:10px;height:10px;background:${source.color};display:inline-block;border-radius:2px;"></span>
      <strong>${source.title}</strong>
      <span>→</span>
      <span style="width:10px;height:10px;background:${target.color};display:inline-block;border-radius:2px;"></span>
      <strong>${target.title}</strong>
      <span style="margin-left:8px;">${value.toLocaleString()}</span>
    </div>`,
});
```

### Node tooltip — `nodeTooltipTemplate({ node, value })`

```js
new ApexSankey(el, {
  nodeTooltipTemplate: ({ node, value }) => `
    <div style="padding:6px 10px;">
      <strong>${node.title}</strong>
      <div style="font-size:12px;opacity:0.7;">Total: ${value.toLocaleString()}</div>
    </div>`,
});
```

### `tooltipTheme` — quick presets

```js
new ApexSankey(el, { tooltipTheme: 'dark' });
```

Equivalent to setting `tooltipBGColor`, `tooltipBorderColor`, `tooltipFontColor` to the dark preset. Use the explicit fields for full control.

### Disabling tooltips

```js
new ApexSankey(el, { enableTooltip: false });
```

## Interaction

### Path highlighting on hover

```js
new ApexSankey(el, {
  highlightConnectedPath: true,    // default
  dimOpacity: 0.2,                 // unrelated elements fade to this opacity
});
```

Hovering a node or edge keeps the connected flow at full opacity and dims everything else to `dimOpacity`. Set `highlightConnectedPath: false` to disable.

Since 1.11 hover gives a one-hop preview, and **clicking** a node or flow pins an isolate of its full upstream and downstream path (cycle-guarded). Click the same element again, or click another, to release or move the isolate.

### Draggable nodes (1.11+)

```js
new ApexSankey(el, { draggableNodes: true });   // default false
```

Nodes can be repositioned with a pointer (mouse, touch, or pen); connected flows follow the node live. The manual position holds until the next `render()`/`update()` recomputes the layout. Sankey projection only (not chord).

### Particle flow (1.11+)

```js
new ApexSankey(el, { particleFlow: true });     // default false
```

Animates particles drifting along each flow ribbon, with density proportional to the ribbon's value, to show direction and volume. Purely decorative; automatically skipped under `prefers-reduced-motion`. Sankey projection only.

### Click callbacks

```js
new ApexSankey(el, {
  onNodeClick: (node) => {
    console.log(node.data?.id, node.value);
  },
});
```

`node.data` is the raw `NodeData` (`{ id, title, color, ... }`); `node.value` is the aggregate flow into that node.

> There is no `onEdgeClick` option. Since 1.11, use the typed event bus instead: `sankey.on('edge:click', ({ source, target, value, originalEvent }) => { ... })`. See `references/motion-events-and-plugins.md`. On 1.10 and earlier, wrap the chart container and use event delegation on the SVG path elements.

## Animation

```js
new ApexSankey(el, {
  animation: { enabled: true, duration: 800 },
});
```

The entrance animation plays only on the first render. It is automatically disabled when the user has `prefers-reduced-motion: reduce` set.

Data-change animation is separate: since 1.11, `sankey.update(data)` springs or morphs the live diagram to new data. See `references/motion-events-and-plugins.md`.

## Themes (1.11+)

Pass `theme` to seed a coordinated set of visual defaults in one shot. Built-ins: `'light'` (the default look), `'dark'`, `'midnight'`, `'mint'`, `'sunset'`. A theme sits between the built-in defaults and your explicit options, so anything you set yourself still wins.

```js
const sankey = new ApexSankey(el, { theme: 'dark' });
```

Register a brand preset once with the static `ApexSankey.registerTheme(name, theme)`, then reference it by name:

```js
ApexSankey.registerTheme('acme', {
  nodePalette: ['#ff5a5f', '#087f8c', '#5d2e8c'],
  fontColor: '#1a1a1a',
  canvasStyle: 'background: #faf7f2; box-sizing: border-box;',
});
const sankey = new ApexSankey(el, { theme: 'acme' });
```

`theme` also accepts an inline `SankeyTheme` object. Its fields (all optional; omitted fields keep the defaults):

| `SankeyTheme` field | Type | Description |
|---|---|---|
| `nodePalette` | `string[]` | Ordered node fill colors, cycled across nodes without their own `color`. |
| `fontColor` | `string` | CSS color for node labels. |
| `edgeOpacity` | `number` | Opacity of the flow ribbons (0-1). |
| `edgeGradientFill` | `boolean` | Fill ribbons with a source→target gradient. |
| `nodeBorderColor` | `string \| null` | CSS color for the node border (`null` disables). |
| `canvasStyle` | `string` | CSS on the SVG root container, typically background and border. |

## CSS variables and family `--apx-*` tokens

Options can also be driven from CSS custom properties, resolved per render pass. Precedence, highest first:

```
product CSS variable  >  explicit option  >  --apx-* token  >  built-in default
```

An option set to a value equal to its built-in default is indistinguishable from one left alone, and the token wins there. Set a product variable if you need a value pinned regardless.

**Product variables** (`--apex-sankey-*`) pin individual Sankey options from CSS, above everything else: `--apex-sankey-node-width`, `--apex-sankey-node-border-color`, `--apex-sankey-node-border-width`, `--apex-sankey-edge-opacity`, `--apex-sankey-edge-gradient`, `--apex-sankey-font-family`, `--apex-sankey-label-color`, `--apex-sankey-label-font-size`, `--apex-sankey-tooltip-bg`, `--apex-sankey-tooltip-border-color`, `--apex-sankey-tooltip-font-color`, `--apex-sankey-tooltip-theme`.

**Family tokens** (`--apx-*`, added in 1.12) are shared by every chart in the ApexCharts family, so a page states its brand once and trees, flow diagrams, Gantt charts and plots all follow:

| Token | Role |
|---|---|
| `--apx-accent` | The color that means interactive or selected. |
| `--apx-fore` | Text and anything that must stay legible on the surface. |
| `--apx-grid` | Hairlines: borders, gridlines, connectors. |
| `--apx-surface` | The plane content sits on. |
| `--apx-series-1` … `--apx-series-N` | An ordered categorical palette (1-based, stops at the first gap). |

```css
:root {
  --apx-accent: #5b21b6;
  --apx-fore: #101828;
  --apx-grid: #e4e7ec;
  --apx-surface: #ffffff;
}

@media (prefers-color-scheme: dark) {
  :root {
    --apx-fore: #f8fafc;
    --apx-grid: #334155;
    --apx-surface: #0f172a;
  }
}
```

Custom properties inherit, so declaring them on `:root` reaches every chart on the page. They resolve below anything configured explicitly, so adopting them cannot change a chart that was already themed.

**Family named themes**: `registerTheme` from `@apex/commons` records a named set of tokens on a registry shared by the whole family; a named family theme's tokens sit one layer below the CSS `--apx-*` tokens. This registry is complementary to `ApexSankey.registerTheme`: a Sankey theme carries diagram-specific defaults (palette, edge opacity, canvas style), while a family theme carries the cross-product token roles. Both are referenced through the same `theme` option.

## Accessibility (`a11y`)

```js
new ApexSankey(el, {
  a11y: {
    enabled: true,                 // default
    diagramLabel: 'Q1 2026 budget allocation',
    description: 'Sankey diagram showing how funds flow from departments to cost categories.',
  },
});
```

When enabled, ApexSankey applies `role="img"` semantics, an `aria-label`, and a `<desc>` element to the SVG root for screen readers (WCAG 2.1 AA target).

Since 1.9.0 the auto-generated ARIA strings are localizable through `locale.messages` (see below). The `a11y.diagramLabel` string still takes precedence: it hard-overrides the SVG root `aria-label`, while `locale.messages.diagramLabel` supplies the localized default when `a11y.diagramLabel` is not set.

## Localization & RTL (`locale`)

Added in 1.9.0. The `locale` option controls text direction and the screen-reader strings the diagram generates. It is additive: the default is `{ direction: 'ltr' }`, so charts that omit `locale` render identically to earlier builds.

```js
const sankey = new ApexSankey(el, {
  locale: {
    direction: 'rtl',
    messages: { nodesGroupLabel: 'العقد' },
  },
});
```

### `locale.direction` — `'ltr' | 'rtl' | 'auto'` (default `'ltr'`)

- `'ltr'`: left-to-right (default, unchanged behavior).
- `'rtl'`: mirrors the diagram horizontally so flows read right-to-left, and sets `dir="rtl"` on the container.
- `'auto'`: defers to the document/element direction.

### `locale.messages` — `Partial<SankeyMessages>`

Overrides any subset of the accessibility strings. Unset keys keep their English defaults, exported as the `DEFAULT_SANKEY_MESSAGES` constant, so you only translate what you need.

| `SankeyMessages` key | Type | Description |
|---|---|---|
| `diagramLabel` | `(ctx: SankeyDiagramLabelContext) => string` | SVG root aria-label summary (node/flow counts plus the largest flow). |
| `nodeAriaLabel` | `(ctx: SankeyNodeLabelContext) => string` | Per-node aria-label (incoming/outgoing flow summary). |
| `edgeAriaLabel` | `(ctx: SankeyEdgeLabelContext) => string` | Per-edge aria-label. Default: `Flow from {source} to {target}: {value} units`. |
| `nodesGroupLabel` | `string` | aria-label for the `<g>` wrapping all nodes. Default: `'Sankey nodes'`. |

The three callback messages receive a typed context object:

- `SankeyDiagramLabelContext`: `{ nodeCount, flowCount, largestFlow?: { source, target, value } }`.
- `SankeyNodeLabelContext`: `{ name, incoming?: { total, sources[] }, outgoing?: { total, targets[] } }`.
- `SankeyEdgeLabelContext`: `{ source, target, value }`.

```js
new ApexSankey(el, {
  locale: {
    messages: {
      edgeAriaLabel: ({ source, target, value }) =>
        `Flujo de ${source} a ${target}: ${value} unidades`,
      nodesGroupLabel: 'Nodos del Sankey',
    },
  },
});
```

New exported types for this feature: `LocaleOptions`, `SankeyMessages`, `TextDirection` (`'ltr' | 'rtl' | 'auto'`), the label-context types (`SankeyDiagramLabelContext`, `SankeyNodeLabelContext`, `SankeyEdgeLabelContext`), and the `DEFAULT_SANKEY_MESSAGES` constant.

> `locale.messages` localizes only the ARIA (screen-reader) strings. Visible tooltips are localized separately through the existing `tooltipTemplate` / `nodeTooltipTemplate` callbacks, so you keep full control of their markup.

## Toolbar

```js
new ApexSankey(el, { enableToolbar: true });   // default
```

The built-in toolbar has zoom-in / zoom-out / reset / export-SVG buttons. Disable by setting `enableToolbar: false` — you can still call `graph.exportToSvg()` manually:

```js
const graph = sankey.render(data);
exportButton.addEventListener('click', () => graph.exportToSvg());
```

## Custom CSS

Inject arbitrary CSS onto the SVG root via `canvasStyle`:

```js
new ApexSankey(el, {
  canvasStyle: 'border: 1px solid #e2e8f0; border-radius: 8px; background: #f8fafc;',
});
```
