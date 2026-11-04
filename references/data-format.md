# Data Format: Tasks, Hierarchy, Milestones, Baseline

## TaskInput shape

Every entry in `series` is a `TaskInput`:

```ts
interface TaskInput {
  id: string;                              // REQUIRED, unique, stable across renders
  name: string;                            // REQUIRED, display label
  startTime: string;                       // REQUIRED except on parent rows (optional in the 3.20.0 type)
  endTime?: string;                        // required with startTime, except on milestones and (3.20.0+) parents
  progress?: number;                       // percentage 0 to 100, default 0, not clamped
  type?: 'task' | 'milestone';             // default 'task' (3.20.0+: 'milestone' when endTime is missing and there are no children); exact lowercase
  parentId?: string;                       // for hierarchical nesting; matched by value
  dependency?: string | TaskDependency;    // ONE predecessor; see references/dependencies.md
  baseline?: { start: string; end: string };
  barBackgroundColor?: string;             // per-task override
  barTextColor?: string;                   // 3.20.0+: per-task colour of the label inside the bar
  rowBackgroundColor?: string;             // per-row override
  collapsed?: boolean;                     // hide children initially
  showSummaryBar?: boolean;                // parents get a summary bar by default; see below
  assignees?: Assignee[];                  // for the renderers.avatars column; see columns-and-toolbar.md
  segments?: TaskSegment[];                // split task; see below
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

Each string is first parsed strictly against `inputDateFormat`. When that fails, ApexGantt falls back to dayjs's lenient parser, so ISO strings such as `'2026-01-15'` still draw under the default format. That fallback is not a format detector: a string it cannot read either (such as `'15/01/2026'` under the default) gives an invalid date and a misplaced or missing bar, and how lenient parsing treats non-ISO strings varies by browser. A mismatched format also goes wrong even when the bars draw: dragged and edited dates are written back in `inputDateFormat`, mixing formats in your data. Through 3.19 the edit dialog shows empty date fields too; from 3.20.0 it reads ISO and other formats, shows an empty field only for a date it cannot read (never "Invalid Date"), and an untouched dialog or inline editor no longer rewrites the stored date. Always set `inputDateFormat` to match your data.

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
- `parentId` must reference an existing task `id` in the same `series`. Through 3.19 a task with an unknown `parentId` is not rendered at all, in the list or the timeline; from 3.20.0 it is attached to the root with a console warning.
- `parentId` and dependency ids are compared by value, so a task whose `id` is the number `1` is found by `parentId: '1'`. Use strings all the same: the types say `string`.
- A `parentId` that makes a cycle (a task naming itself, or a under b under a) overflows the stack through 3.19. From 3.20.0 the task is shown at the root with the warning `[ApexGantt] Task "<id>" has parentId "<p>", which makes a cycle of parents. It is shown at the root.`
- A parent row may omit `startTime` and `endTime`: it derives its span from its children. From 3.20.0 it may also give only one of them and still draws a summary bar over its children (through 3.19 that throws "Task must have an id, start, and end date"). A start-only parent with no `type` is typed `'task'`; the same row without children is a milestone.
- A task with no `startTime` and no children throws. From 3.20.0 the message is `Task "<id>" has no startTime and no children. Provide dates or add child tasks.` (through 3.19 it read "has no startTime/endTime and no children"), and it also covers a leaf with only an `endTime` and a `type: 'milestone'` with no start, which used to throw other messages.
- Any other task needs both dates. Through 3.19 a `startTime` without an `endTime` throws "Task must have an id, start, and end date" unless `type` is `'milestone'`. From 3.20.0 a task with no `endTime`, no `type` and no children is drawn as a milestone; an explicit `type: 'task'` with no `endTime` still throws.
- Parent rows render an expand/collapse caret automatically.
- `collapsed: true` on the parent hides its children initially. At runtime use `gantt.expandTask(id)` / `gantt.collapseTask(id)` (3.20.0+); through 3.19 `updateTask(id, { collapsed })` changes the model without re-rendering.
- Rows are sorted by start date by default (siblings within each parent), so the display order is not the array order. Pass `sortBy: []` to keep the order of `series`.

## Milestones

Milestones render as a diamond marker at `startTime` and need no `endTime`. The `type` must be the exact lowercase `'milestone'`: `'Milestone'` is treated as an ordinary task. An `endTime` on a milestone is ignored for drawing but still widens the timeline and is announced as a range, so omit it:

```js
{ id: 'launch', name: 'Launch', startTime: '06-15-2026', type: 'milestone' }
```

From 3.20.0 a task with no `endTime`, no `type` and no children is also drawn as a milestone (end = start); with children it is a summary bar instead. Through 3.19 the same row throws. Set `type: 'milestone'` explicitly: it is the clearest and works on every version.

Use the `TaskType` enum if you prefer:

```js
import { TaskType } from 'apexgantt';
{ id: 'launch', name: 'Launch', startTime: '06-15-2026', type: TaskType.Milestone }
```

## Split tasks (worked segments)

A task can be split into several **worked segments** separated by gaps on a single row (work Mon–Wed, pause, resume Fri). Pass `segments`, an array of `TaskSegment` objects. **Each segment uses `start` / `end`, not `startTime` / `endTime`:**

```js
{
  id: 't1', name: 'Build',
  segments: [
    { start: '2026-06-01', end: '2026-06-04' },
    { start: '2026-06-08', end: '2026-06-12' },
  ],
}
```

`TaskSegment`: `{ start: string; end: string }` (both parsed with `inputDateFormat`). An entry missing either is dropped with a warning.

A split needs two or more segments. From 3.20.0 a task given exactly one segment is stored as a plain task: the segment sets `startTime` / `endTime` and `segments` is not kept, so `updateTask(id, { segments: [one] })` turns a split task back into a normal one.

The task's own `startTime` / `endTime` are the **envelope** (first segment start → last segment end) and are derived automatically, so summary rollups, dependencies, the timeline header, and the Duration column all keep working. The bar draws one filled piece per segment joined by a thin connector; progress fills the worked spans left to right. Dragging moves all segments together; resizing adjusts the outer segment.

Split (or unsplit) at runtime:

```js
gantt.splitTask('t1', '2026-06-08', { resumeAt: '2026-06-12' });  // cut at a date, resume later
gantt.updateTask('t1', { segments: [] });                         // back to one contiguous bar
gantt.isSplit('t1');                                              // → boolean
```

`splitTask()` routes through `updateTask`, so it is validated and undoable (from 3.20.0 it also emits `taskUpdate` / `taskUpdateSuccess` with `source: 'api'`; through 3.19 `updateTask` emits no events). It is a no-op on milestones, summary bars, or when the split date does not fall strictly inside a worked span.

`splitTask()` keeps the task's finish date: the gap comes out of the existing span, so the task does not slip. To show a slip, pass the new `segments` through `updateTask`. From 3.20.0 `resumeAt` must fall after `at` and no later than the segment's last day with a day-only format (the end day is worked, so a one-day remainder is fine), or before the segment's end with a format that carries time; anything else is rejected with a console warning and the task is left unchanged. Through 3.19 a `resumeAt` past the segment's end produces a backwards segment, and one that is not after `at` overlapping segments. The default `resumeAt` is the day after `at`, which leaves no visible gap with day-only formats: pass a later date to open one. (Deferred: interactive split gesture, per-segment drag/resize, gap-aware critical path.)

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

Any parent task (one that has children via `parentId`) renders as a summary bar
by default; you do not need to set `showSummaryBar`. Its date range is computed
from the earliest start and latest end of all descendants, and the parent's own
`startTime`/`endTime` are ignored for drawing (silently: nothing is logged).
Summary bars are always read-only: drag, resize, and progress are disabled.
There is no `'summary'` task type: `type` is only `'task'` or `'milestone'`.

```js
{ id: 'phase1', name: 'Phase 1' },                                   // summary bar, span from children
{ id: 'a', name: 'Spec', parentId: 'phase1', startTime: '01-01-2026', endTime: '01-10-2026' },
```

Set `showSummaryBar: false` on a parent to draw it as an ordinary bar from its
own `startTime`/`endTime` instead (it then needs both dates).

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

**Supported parse keys:** `id`, `name`, `startTime`, `endTime`, `progress`, `type`, `parentId`, `dependency`, `barBackgroundColor`, `rowBackgroundColor`, `collapsed`. From 3.20.0 also `showSummaryBar`, `baseline`, `segments`, `assignees` and `barTextColor`; through 3.19 those fields are dropped from parsed rows.

Through 3.19 every row whose mapped `startTime` is empty is skipped with a console warning, so a parent row with no dates of its own disappears along with the hierarchy under it. From 3.20.0 a row with no start date is kept only when it is the parent of a dated row, directly or further down; it takes its span from its children (a parent with only an end date included). Any other row with no start date (an unscheduled backlog item, a row with only an end date, null dates) is skipped with the warning `DataParser: Skipping item at index N: it has no start date and no dated child rows`. Ids are matched by value here too, so a numeric `id` and a string `parentId` meet. A row whose `parentId` matches no row is shown at the root with a warning.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `startTime: '2026-06-01'` with default format | Set `inputDateFormat: 'YYYY-MM-DD'` |
| `progress: 0.75` | `progress: 75` |
| `type: 'Milestone'` (treated as a task, throws without `endTime`) | `type: 'milestone'`, lowercase |
| No `type` and no `endTime` (throws through 3.19, a milestone from 3.20.0) | Say what you mean: `type: 'milestone'`, or add `endTime` |
| An unscheduled row with no dates (throws in `series`; skipped with a warning under `parsing`) | Give it dates, or leave it out until it is scheduled |
| `parentId` loop (`a` under `b` under `a`) | Fix the data; 3.20.0 shows the task at the root with a warning, 3.19 overflows the stack |
| Milestone with `endTime` (widens the timeline) | Omit `endTime` |
| `type: 'summary'` (not a type) | Leave `type` unset; any parent gets a summary bar |
| `dependency: ['a', 'b']` (arrays draw nothing) | One predecessor in the data; add more with `gantt.addDependency()` |
| Re-using `id` across renders | Keep ids stable; selection / dependencies break otherwise |
| Expecting rows in array order | Rows sort by start date; pass `sortBy: []` |
| Mutating `series` directly | `gantt.addTask()` / `updateTask()` / `deleteTask()` for single changes; `gantt.update({ series: [...] })` to replace the whole dataset |
