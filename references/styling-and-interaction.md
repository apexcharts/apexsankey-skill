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
  dimOpacity: 0.15,                // unrelated elements fade to this opacity
});
```

Hovering a node or edge keeps the connected flow at full opacity and dims everything else to `dimOpacity`. Set `highlightConnectedPath: false` to disable.

### Click callbacks

```js
new ApexSankey(el, {
  onNodeClick: (node) => {
    console.log(node.data?.id, node.value);
  },
});
```

`node.data` is the raw `NodeData` (`{ id, title, color, ... }`); `node.value` is the aggregate flow into that node.

> There is no built-in `onEdgeClick`. Wrap the chart container and use event delegation on the SVG path elements if you need it.

## Animation

```js
new ApexSankey(el, {
  animation: { enabled: true, duration: 800 },
});
```

The entrance animation plays only on the first render. It is automatically disabled when the user has `prefers-reduced-motion: reduce` set.

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
