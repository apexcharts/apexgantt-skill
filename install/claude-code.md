# Installing ApexGantt Skill for Claude Code

## Installation

```bash
# Navigate to your project's Claude config
mkdir -p .claude/skills
cd .claude/skills

# Clone the skill
git clone https://github.com/apexcharts/apexgantt-skill.git
```

Claude Code will automatically detect `SKILL.md` and load it when working on ApexGantt code.

## Verification

Ask Claude to build a Gantt chart:

> Create an ApexGantt chart with three tasks, one milestone, and a Finish-to-Start dependency.

Claude should generate code that:
- Calls `new ApexGantt(el, options)` followed by `gantt.render()`
- Uses the correct task data format (`id`, `name`, `startTime`, `endTime`, `progress`)
- Sets `inputDateFormat` to match the date strings in the data
- Returns `gantt.destroy()` from the cleanup hook in React / Vue / Angular
