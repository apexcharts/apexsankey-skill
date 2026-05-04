# Installing ApexSankey Skill for Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apexsankey-skill.git
```

Claude Code will automatically detect `SKILL.md` and load it when working on ApexSankey code.

## Verification

> Build an ApexSankey diagram showing budget flow from three departments through two hubs to four cost categories.

Claude should:
- Construct with `(element, options)` and call `sankey.render({ nodes, edges, options: sankey.options })`
- Use unique `node.id` values referenced consistently in `edges`
- Return `sankey.destroy()` from the cleanup hook in React / Vue / Angular
