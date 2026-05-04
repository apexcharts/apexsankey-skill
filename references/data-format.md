# Data Format — Nodes, Edges, Options

## Top-level shape

```ts
interface GraphData {
  nodes: SankeyGraphNode[];
  edges: SankeyGraphEdge[];
  options: SankeyOptions;       // pass `sankey.options` back in
}
```

## Nodes

```ts
interface SankeyGraphNode {
  id: string;        // REQUIRED, unique
  title: string;     // REQUIRED, display label
  color?: string;    // override the auto-assigned palette color
}
```

- `id` must be unique. Duplicates silently collide.
- `title` is rendered beside / inside the node and used in default tooltips.
- `color` overrides the palette for this node (and tints the gradient on edges that touch it when `edgeGradientFill: true`).

## Edges

```ts
interface SankeyGraphEdge {
  source: string;    // REQUIRED, must match a node.id
  target: string;    // REQUIRED, must match a node.id
  value: number;     // REQUIRED, > 0
  type: string;      // category label — use the same string for edges that group together
}
```

- Width of the band is proportional to `value`.
- `type` groups edges visually and is used by `options.alignLinkTypes` and the default tooltip.
- Cycles (e.g. `a → b → a`) are not supported. Aggregate or break them in your data layer.

## Layer & band ordering

By default ApexSankey assigns nodes to layers automatically using a longest-path algorithm. Pin layout with `options.order`:

```ts
order: string[][][];     // layer → band → node-id[]
```

- **Layer**: a vertical column.
- **Band**: a contiguous group of nodes inside a layer (used for visual separation).

```js
sankey.render({
  nodes, edges,
  options: {
    ...sankey.options,
    order: [
      [['source-a', 'source-b']],         // layer 0, one band, two nodes top-to-bottom
      [['hub']],                          // layer 1
      [['sink-1', 'sink-2'], ['sink-3']], // layer 2, two bands (gap between)
    ],
  },
});
```

When `order` is set, ApexSankey skips both the rank assignment and the ordering algorithm.

## Worked example — energy flow

```js
const data = {
  nodes: [
    { id: 'coal',     title: 'Coal'     },
    { id: 'gas',      title: 'Natural Gas' },
    { id: 'solar',    title: 'Solar'    },
    { id: 'electric', title: 'Electric Grid' },
    { id: 'home',     title: 'Residential' },
    { id: 'industry', title: 'Industry' },
    { id: 'loss',     title: 'Loss', color: '#999' },
  ],
  edges: [
    { source: 'coal',     target: 'electric', value: 32, type: 'fossil' },
    { source: 'gas',      target: 'electric', value: 25, type: 'fossil' },
    { source: 'solar',    target: 'electric', value: 12, type: 'renew'  },
    { source: 'electric', target: 'home',     value: 28, type: 'use'    },
    { source: 'electric', target: 'industry', value: 32, type: 'use'    },
    { source: 'electric', target: 'loss',     value:  9, type: 'loss'   },
  ],
  options: sankey.options,
};

sankey.render(data);
```

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Edge points at a non-existent node id | Validate `source` / `target` are in `nodes` before render |
| Duplicate `node.id` | Make ids unique (use `crypto.randomUUID()` if needed) |
| `value: 0` or negative | Filter out at the data layer; positive values only |
| Cycles | Aggregate or break before constructing `GraphData` |
| `sankey.render({ nodes, edges })` (missing `options`) | Always include `options: sankey.options` in the render payload |
