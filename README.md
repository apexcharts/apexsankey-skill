# ApexSankey AI Skill

AI coding skill for building [ApexSankey](https://apexcharts.com/docs/apexsankey/) flow / Sankey diagrams. Works with Claude Code, Cursor, GitHub Copilot, and any AI coding assistant.

## What This Does

AI models routinely get Sankey-diagram code wrong: passing data to the constructor, omitting `render()`, dangling `source` / `target` references, dropping `options` from the render payload. This skill ships structured reference files so the assistant generates correct ApexSankey code on the first try.

### Coverage

- **Data format** — `{ nodes, edges, options }`, layer ordering, edge grouping
- **Lifecycle** — `setLicense()`, construct, `render()`, `destroy()`, exporting SVG
- **Tooltips** — separate edge / node template shapes
- **Interaction** — path highlighting, click callbacks, animation, accessibility
- **Framework wrappers**: `react-apexsankey`, `vue-apexsankey`, `ngx-apexsankey`

## Installation

### Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apexsankey-skill.git
```

### Cursor / Windsurf

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexsankey-skill/main/.cursorrules
```

### GitHub Copilot

Reference `SKILL.md` in Copilot Chat: `@workspace #file:SKILL.md`, or paste the contents of `.cursorrules` into Copilot's custom instructions.

### As an npm dependency

```bash
npm install apexsankey-skill
```

```js
import { skillFile, referencePath } from 'apexsankey-skill';
import { readFile } from 'node:fs/promises';

const skill = await readFile(skillFile, 'utf8');
const data  = await readFile(referencePath('data-format.md'), 'utf8');
```

## Repository Structure

```
├── SKILL.md                           # Main entry point
├── .cursorrules                       # Self-contained Cursor / Windsurf version
├── references/
│   ├── data-format.md                 # nodes, edges, options, layer order
│   ├── styling-and-interaction.md     # tooltips, animation, a11y, callbacks
│   └── framework-wrappers.md          # React, Vue, Angular
└── install/
    ├── claude-code.md
    ├── cursor.md
    └── copilot.md
```

## Links

- [ApexSankey Documentation](https://apexcharts.com/docs/apexsankey/)
- [ApexSankey GitHub](https://github.com/apexcharts/apexsankey)
- [npm: apexsankey](https://www.npmjs.com/package/apexsankey)
- [react-apexsankey](https://www.npmjs.com/package/react-apexsankey)
- [vue-apexsankey](https://www.npmjs.com/package/vue-apexsankey)
- [ngx-apexsankey](https://www.npmjs.com/package/ngx-apexsankey)

## License

MIT
