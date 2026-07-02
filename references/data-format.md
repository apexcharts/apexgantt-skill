# Data Format — Tasks, Hierarchy, Milestones, Baseline

## TaskInput shape

Every entry in `series` is a `TaskInput`:

```ts
interface TaskInput {
  id: string;                              // REQUIRED, unique, stable across renders
  name: string;                            // REQUIRED, display label
  startTime: string;                       // REQUIRED, parsed by inputDateFormat
  endTime?: string;                        // omit for milestones
  progress?: number;                       // 0–100, default 0
  type?: 'task' | 'milestone';             // default 'task'
  parentId?: string;                       // for hierarchical nesting
  dependency?: string | TaskDependency;    // see references/dependencies.md
  baseline?: { start: string; end: string };
  barBackgroundColor?: string;             // per-task override
  rowBackgroundColor?: string;             // per-row override
  collapsed?: boolean;                     // hide children initially
  showSummaryBar?: boolean;                // default true; see below (summary/group rows)
  assignees?: Assignee[];                  // for the renderers.avatars column; see columns-and-toolbar.md
}
```

## Date parsing

Date strings are parsed with [dayjs](https://day.js.org/docs/en/parse/string-format) using `options.inputDateFormat` (default `'MM-DD-YYYY'`).

| inputDateFormat | startTime example |
|---|---|
| `'MM-DD-YYYY'` *(default)* | `'01-15-2026'` |
| `'YYYY-MM-DD'` | `'2026-01-15'` |
| `'DD/MM/YYYY'` | `'15/01/2026'` |
| `'YYYY-MM-DD HH:mm'` | `'2026-01-15 09:30'` |

If the string doesn't match the format, the bar fails to render with no console error. Always set `inputDateFormat` to match your data.

## Hierarchy via `parentId`

```js
series: [
  { id: 'phase1', name: 'Phase 1',                               startTime: '01-01-2026', endTime: '02-01-2026' },
  { id: 't1.1',  name: 'Subtask 1.1',  parentId: 'phase1',       startTime: '01-01-2026', endTime: '01-15-2026' },
  { id: 't1.2',  name: 'Subtask 1.2',  parentId: 'phase1',       startTime: '01-16-2026', endTime: '02-01-2026' },
  { id: 'phase2', name: 'Phase 2',                               startTime: '02-02-2026', endTime: '03-15-2026' },
  { id: 't2.1',  name: 'Subtask 2.1',  parentId: 'phase2', collapsed: true,
                                                                 startTime: '02-02-2026', endTime: '03-01-2026' },
]
```

**Rules:**
- `parentId` must reference an existing task `id` in the same `series`. Orphans are silently flattened to top-level.
- Parent rows render an expand/collapse caret automatically.
- `collapsed: true` on the parent hides its children initially.

## Milestones

Milestones render as a diamond marker at `startTime`. Omit `endTime`:

```js
{ id: 'launch', name: 'Launch', startTime: '06-15-2026', type: 'milestone' }
```

Use the `TaskType` enum if you prefer:

```js
import { TaskType } from 'apexgantt';
{ id: 'launch', name: 'Launch', startTime: '06-15-2026', type: TaskType.Milestone }
```

## Baseline (planned vs. actual)

Enable globally, then attach `baseline: { start, end }` to any task you want to compare against its plan:

```js
const gantt = new ApexGantt(el, {
  baseline: { enabled: true, color: '#b0b8c1' },
  inputDateFormat: 'YYYY-MM-DD',
  series: [
    { id: 't1', name: 'Design',
      startTime: '2026-01-01', endTime: '2026-01-20',           // actual
      baseline:  { start: '2026-01-01', end: '2026-01-15' } }   // planned
  ]
});
```

A thin secondary bar is rendered below the actual bar so schedule slip is immediately visible.

## Summary (group) bars

A parent task (one that has children via `parentId`) renders as a summary bar
by default. Its date range is computed automatically from the earliest start
and latest end of all descendants — the parent's own `startTime`/`endTime` are
ignored (a warning is logged if both are set). Summary bars are always
read-only: drag, resize, and progress are disabled.

Set `showSummaryBar: false` on the parent to suppress the derived-span bar for
that row.

```js
{ id: 'phase1', name: 'Phase 1', startTime: '01-01-2026', showSummaryBar: true }  // default
```

## Assignees

Attach people/teams to a task for the built-in `renderers.avatars` column. The
library does not interpret these directly — they only render when an
avatar-style column is configured via `columnConfig` (see
`references/columns-and-toolbar.md`).

```js
{
  id: 't1', name: 'Design', startTime: '01-01-2026', endTime: '01-15-2026',
  assignees: [
    { name: 'Ada Lovelace', avatarUrl: '/u/ada.png' },
    { name: 'Alan Turing', initials: 'AT', color: '#6366F1' },  // initials fallback when no avatarUrl
  ],
}
```

`Assignee`: `{ name; avatarUrl?; initials?; color? }`.

## Per-task styling overrides

```js
{
  id: 't1', name: 'Critical Task',
  startTime: '01-01-2026', endTime: '01-31-2026',
  barBackgroundColor: '#e53935',     // override task bar color
  rowBackgroundColor: '#FFF3E0',     // override row background
}
```

## Data Parsing for non-standard shapes

When your API returns data with different field names, use `parsing` instead of a manual `.map()`:

```js
const apiData = [
  { task_id: 'T1', task_name: 'Design', start_date: '01-01-2026',
    end_date: '01-15-2026', completion: 0.75 }
];

new ApexGantt(el, {
  series: apiData,
  parsing: {
    id: 'task_id',
    name: 'task_name',
    startTime: 'start_date',
    endTime: 'end_date',
    progress: { key: 'completion', transform: (v) => v * 100 },
  }
});
```

`parsing` accepts dot-notation paths for nested keys (`'project.task.id'`) and per-field `transform` functions for type coercion.

**Supported parse keys:** `id`, `name`, `startTime`, `endTime`, `progress`, `type`, `parentId`, `dependency`, `barBackgroundColor`, `rowBackgroundColor`, `collapsed`.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `startTime: '2026-06-01'` with default format | Set `inputDateFormat: 'YYYY-MM-DD'` |
| `progress: 0.75` | `progress: 75` |
| Milestone with `endTime` | Omit `endTime`, set `type: 'milestone'` |
| Re-using `id` across renders | Keep ids stable; selection / dependencies break otherwise |
| Mutating `series` directly | Always go through `gantt.update({ series: [...] })` |
