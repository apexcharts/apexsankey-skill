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
- `type` groups edges visually and is shown in the default tooltip.
- Cycles are supported since 1.11: a circular link (e.g. `recycle → raw` upstream) is routed and rendered as a dashed back-edge returning against the flow direction, so loops read clearly. Do not treat cycles as invalid data on 1.11+. On 1.10 and earlier the graph must be a DAG: aggregate or break cycles in your data layer.

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

## Projections (1.11+): alluvial and chord

The same `{ nodes, edges }` model drives three views: the layered Sankey (default), an alluvial diagram, and a chord diagram.

### Alluvial: `buildAlluvialData(input)` + `axisTitles`

For categorical records flowing across ordered dimensions (cohorts over time, survey answers across questions), do not hand-build nodes and edges. `ApexSankey.buildAlluvialData` (also a named export `buildAlluvialData`) creates one node per (dimension, category) and one edge per adjacent-dimension transition. Each category keeps the same color across every dimension, so a cohort reads as one continuous stream. Pair with the `axisTitles` option for the dimension labels.

```js
const input = {
  dimensions: ['2019', '2022', '2025'],
  records: [
    { values: { 2019: 'Free', 2022: 'Pro',  2025: 'Pro'     } },
    { values: { 2019: 'Free', 2022: 'Free', 2025: 'Churned' } },
    { values: { 2019: 'Pro',  2022: 'Pro',  2025: 'Team'    }, value: 3 },
  ],
};

const sankey = new ApexSankey(el, { axisTitles: input.dimensions });
sankey.render({ ...ApexSankey.buildAlluvialData(input), options: sankey.options });
```

`AlluvialInput` fields:

| Field | Type | Description |
|---|---|---|
| `dimensions` | `string[]` | Ordered dimension (axis) ids, left → right. |
| `records` | `AlluvialRecord[]` | The subjects flowing across the dimensions. |
| `palette` | `string[]` | Optional category color palette, cycled per distinct category. |

Each `AlluvialRecord` is `{ values: Record<string, string>, value?: number }`: the category at each dimension (keyed by dimension id) and the weight it contributes (default `1`). A record missing a category at some dimension skips that adjacency.

`axisTitles: string[]` labels rank `i` with index `i` (drawn above each column, or beside each row when `orientation: 'vertical'`).

### Chord: `type: 'chord'`

`type: 'chord'` draws the radial projection: each node becomes an arc on a ring (span proportional to total incident flow), each edge a ribbon crossing the interior. Use it for dense many-to-many or symmetric relationships (migration, co-occurrence) where a layered Sankey turns into spaghetti. Hovering an arc or ribbon focuses its connections and dims the rest.

```js
const sankey = new ApexSankey(el, { type: 'chord' });
sankey.render({
  nodes: [
    { id: 'A', title: 'A' },
    { id: 'B', title: 'B' },
    { id: 'C', title: 'C' },
  ],
  edges: [
    { source: 'A', target: 'B', value: 12, type: 'x' },
    { source: 'B', target: 'C', value:  8, type: 'x' },
    { source: 'C', target: 'A', value:  5, type: 'x' },
  ],
  options: sankey.options,
});
```

- `arcCornerRadius` (default `6`, chord only) rounds each arc's outer corners; `0` gives sharp corners; clamped to the ring band width.
- Chord mode reuses the tooltips, node/edge click events, and color palette.
- Sankey-only features do not apply in chord mode: `orientation`, RTL mirroring, animated relayout, `draggableNodes`, `particleFlow`, and drill-down.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Edge points at a non-existent node id | Validate `source` / `target` are in `nodes` before render |
| Duplicate `node.id` | Make ids unique (use `crypto.randomUUID()` if needed) |
| `value: 0` or negative | Filter out at the data layer; positive values only |
| Breaking cycles on 1.11+ (or reporting dashed back-edges as a bug) | Cyclic links are supported and render dashed on purpose; only pre-1.11 requires a DAG |
| `sankey.render({ nodes, edges })` (missing `options`) | Always include `options: sankey.options` in the render payload |
