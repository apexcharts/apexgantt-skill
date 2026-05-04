# Installing ApexGantt Skill for Cursor

## Setup

1. Copy the `.cursorrules` file from this repository into the root of your project:

```bash
cp /path/to/apexgantt-skill/.cursorrules /path/to/your-project/.cursorrules
```

Or download it directly:

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexgantt-skill/main/.cursorrules
```

2. Restart Cursor or open a new Cursor window.

Cursor automatically reads `.cursorrules` files in the project root and uses them as context for AI-assisted coding.

## For Windsurf

Same approach — Windsurf also supports `.cursorrules` files in the project root.

## Verification

Ask Cursor to generate a Gantt with milestones and dependencies. It should follow the correct data format, the `render()` / `destroy()` lifecycle, and listen for events on the container element rather than the chart instance.
