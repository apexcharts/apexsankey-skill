# Installing ApexSankey Skill for Cursor

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexsankey-skill/main/.cursorrules
```

Restart Cursor or open a new window. Cursor automatically reads `.cursorrules` files in the project root as context.

## For Windsurf

Same approach — Windsurf also reads `.cursorrules`.

## Verification

Ask Cursor to generate a Sankey diagram with custom node colors and gradient edges. It should pass `{ nodes, edges, options }` to `sankey.render()`, give each node a unique `id`, and leave node placement to the automatic layout.
