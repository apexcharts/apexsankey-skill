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

- `id` must be unique. With duplicates, the last declaration overwrites the earlier ones.
- A node no edge references is not drawn: the layout builds its node set from the edges.
- `title` is rendered beside / inside the node and used in default tooltips.
- `color` overrides the palette for this node (and tints the gradient on edges that touch it when `edgeGradientFill: true`).

## Edges

```ts
interface SankeyGraphEdge {
  source: string;    // REQUIRED, must match a node.id
  target: string;    // REQUIRED, must match a node.id
  value: number;     // REQUIRED, not negative (0 draws an invisible flow)
  type: string;      // tells parallel edges between the same two nodes apart
}
```

- Width of the band is proportional to `value`. A negative value breaks the band geometry; a graph where every value is `0` cannot be sized.
- An endpoint that matches no `node.id` is not dropped: ApexSankey creates an unlabeled phantom node for it.
- `type` is the key that tells parallel edges apart. Two edges with the same `source`, `target`, and `type` collapse into one, the later replacing the earlier (values are not summed). It is not shown in the default tooltip and does not affect styling. The TypeScript types declare it required; at runtime it may be omitted, which counts as one shared empty type.
- Cycles are supported: a circular link (e.g. `recycle → raw` upstream) is routed and rendered as a dashed back-edge returning against the flow direction, so loops read clearly (1.11+; 1.10 lays them out but draws the back-edge solid). Do not treat cycles as invalid data.

## Layer layout

ApexSankey places nodes automatically. Each node's column (rank) comes from rank assignment over the edges, and the order of nodes within a column from a crossing-minimization pass. 1.12 has no public option to pin either: an `order` array in `options` is not read, so earlier examples that passed one were laying out automatically all along. For manual adjustment, `draggableNodes: true` lets a reader drag nodes; the positions hold until the next `render()` / `update()`.

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
| Edge points at a non-existent node id (it becomes an unlabeled phantom node) | Validate `source` / `target` are in `nodes` before render |
| Duplicate `node.id` (the last declaration wins) | Make ids unique (use `crypto.randomUUID()` if needed) |
| `value: 0` (invisible flow) or negative (broken geometry) | Filter out at the data layer; positive values only |
| Two edges with the same `source`, `target`, and `type` (only the last survives) | Merge them into one edge, or give each a distinct `type` |
| A node with no edges (not drawn) | Remove it, or add the flow it represents |
| Passing `options.order` to pin the layout (it is not read) | Let the layout place nodes; use `draggableNodes` for manual adjustment |
| Breaking cycles (or reporting dashed back-edges as a bug) | Cyclic links are supported and render dashed on purpose |
| `sankey.render({ nodes, edges })` (missing `options`) | Always include `options: sankey.options` in the render payload |
