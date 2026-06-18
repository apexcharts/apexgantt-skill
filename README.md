# ApexGantt AI Skill

AI coding skill for building [ApexGantt](https://apexcharts.com/docs/apexgantt/) timeline / Gantt charts. Works with Claude Code, Cursor, GitHub Copilot, and any AI coding assistant that can read project context.

> **Separate skill — one of the ApexCharts ecosystem skills.** This is the dedicated skill for **ApexGantt** (`apexgantt`), shipped as its own `apexgantt-skill` package and repo — distinct from the core `apexcharts-skill` and the other product skills. Each product has its own library and skill; use the one that matches yours:
>
> | Product | npm library | Skill package & repo |
> |---|---|---|
> | ApexCharts — charts | `apexcharts` | [`apexcharts-skill`](https://github.com/apexcharts/apexcharts-skill) |
> | **ApexGantt** — Gantt / timeline · *this skill* | `apexgantt` | `apexgantt-skill` |
> | ApexTree — hierarchy / org charts | `apextree` | [`apextree-skill`](https://github.com/apexcharts/apextree-skill) |
> | ApexSankey — flow / Sankey | `apexsankey` | [`apexsankey-skill`](https://github.com/apexcharts/apexsankey-skill) |
> | Apex Grid — data grid | `apex-grid` | [`apexgrid-skill`](https://github.com/apexcharts/apexgrid-skill) |

## What This Does

AI models routinely get Gantt-chart code wrong: bad date formats, missing `render()` calls, listening for events on the wrong target, mutating `series` in place, treating `progress` as a fraction. This skill ships structured reference files so the assistant generates correct ApexGantt code on the first try.

### Coverage

- **Task data format** — `id`, `name`, `startTime`, `endTime`, `progress`, hierarchy, milestones, baseline
- **Dependencies** — `FS` / `SS` / `FF` / `SF` types with lag/lead, critical path
- **Lifecycle** — `render()`, `update()`, `updateTask()`, `destroy()`, license key
- **Events** — full `GanttEventMap` with `CustomEvent.detail` shapes
- **Selection, drag, resize, inline edit** with the right opt-in flags
- **Custom toolbar items**, columns, parsing for non-standard data shapes
- **Framework wrappers**: `react-apexgantt`, `vue-apexgantt`, `ngx-apexgantt`

## Installation

### Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apexgantt-skill.git
```

### Cursor / Windsurf

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexgantt-skill/main/.cursorrules
```

### GitHub Copilot

Reference `SKILL.md` in Copilot Chat: `@workspace #file:SKILL.md`, or paste the contents of `.cursorrules` into Copilot's custom instructions.

### Generic AI Assistant

Paste the contents of `SKILL.md` into the system prompt or attach it as context.

### As an npm dependency

For tools that build on top of this skill (MCP servers, custom AI agents):

```bash
npm install apexgantt-skill
```

```js
import { skillFile, referencesDir, referencePath } from 'apexgantt-skill';
import { readFile } from 'node:fs/promises';

const skill = await readFile(skillFile, 'utf8');
const deps = await readFile(referencePath('dependencies.md'), 'utf8');
```

## Repository Structure

```
├── SKILL.md                       # Main entry point — read this first
├── .cursorrules                   # Self-contained version for Cursor / Windsurf
├── references/
│   ├── data-format.md             # tasks, hierarchy, milestones, baseline
│   ├── dependencies.md            # FS/SS/FF/SF, lag/lead, critical path
│   ├── columns-and-toolbar.md     # columnConfig, toolbarItems, parsing
│   ├── events.md                  # GanttEventMap, selection, drag, resize
│   └── framework-wrappers.md      # React, Vue, Angular
└── install/
    ├── claude-code.md
    ├── cursor.md
    └── copilot.md
```

## Links

- [ApexGantt Documentation](https://apexcharts.com/docs/apexgantt/)
- [ApexGantt GitHub](https://github.com/apexcharts/apexgantt)
- [npm: apexgantt](https://www.npmjs.com/package/apexgantt)
- [react-apexgantt](https://www.npmjs.com/package/react-apexgantt)
- [vue-apexgantt](https://www.npmjs.com/package/vue-apexgantt)
- [ngx-apexgantt](https://www.npmjs.com/package/ngx-apexgantt)

## License

MIT
