# Dependencies & Critical Path

## Dependency types

ApexGantt supports the four standard PMI dependency types:

| Type | Meaning |
|---|---|
| `'FS'` | Finish-to-Start. Successor starts when predecessor finishes. *(Default.)* |
| `'SS'` | Start-to-Start. Successor starts when predecessor starts. |
| `'FF'` | Finish-to-Finish. Successor finishes when predecessor finishes. |
| `'SF'` | Start-to-Finish. Successor finishes when predecessor starts. *(Rare.)* |

## Two ways to declare a dependency

### A) Plain id string (shortest)

Treated as `FS` with `lag: 0`.

```js
{ id: 't2', name: 'Develop', startTime: '01-16-2026', endTime: '02-15-2026', dependency: 't1' }
```

### B) `TaskDependency` object (full control)

```ts
interface TaskDependency {
  taskId: string;                          // REQUIRED, id of predecessor
  type?: 'FS' | 'SS' | 'FF' | 'SF';        // default 'FS'
  lag?: number;                            // days; positive = lag, negative = lead
  lagUnit?: 'working' | 'calendar';        // how lag counts days; see below
}
```

### `lagUnit` (3.12.0)

Controls how `lag` counts days. When a working calendar is configured
(`GanttUserOptions.calendar`), the default is `'working'` — lag skips weekends
and holidays. Without a calendar, working and calendar days coincide so the
setting has no effect. Force raw calendar-day lag with `'calendar'`.

```js
{ id: 't2', name: 'Develop', startTime: '01-16-2026', endTime: '02-15-2026',
  dependency: { taskId: 't1', type: 'FS', lag: 2, lagUnit: 'calendar' } }
```

```js
{
  id: 't2', name: 'Develop', startTime: '01-16-2026', endTime: '02-15-2026',
  dependency: { taskId: 't1', type: 'SS', lag: 2 }       // SS + 2-day delay
}
```

```js
{
  id: 't3', name: 'Polish',  startTime: '02-15-2026', endTime: '02-25-2026',
  dependency: { taskId: 't2', type: 'FS', lag: -1 }      // FS + 1-day overlap
}
```

## Critical path

The critical path is the longest chain of dependent tasks — slippage on any of these tasks delays the project. Enable with `enableCriticalPath: true`:

```js
const gantt = new ApexGantt(el, {
  enableCriticalPath: true,
  criticalBarColor: '#e53935',     // bar fill on the critical path
  criticalArrowColor: '#e53935',   // arrow stroke on the critical path
  series: tasks,
});
gantt.render();
```

ApexGantt computes the path automatically using each task's start/end and dependency graph and re-runs the calculation on every render or update. Since 3.18.3 it is also recomputed after every edit that can move it: `addDependency()` / `removeDependency()`, a link drawn between two bars, a date change through `updateTask()`, a drag or the edit dialog, and undo/redo of any of these. The `bar-critical` and `critical-arrow` classes move to match, so there is no need to re-render after an edit. Edits that cannot change the path (progress, name, colour) skip the recompute.

## Editing dependencies interactively

Off by default. Set `dependencies: { editable: true }` to let users draw and delete arrows:

- **Draw**: hovering a bar shows a circle at its start and finish edges. Drag from one onto another bar's start or finish edge. The pair of edges sets the type: finish to start is `FS`, start to start `SS`, finish to finish `FF`, start to finish `SF`. A drawn link has `lag: 0`.
- **Delete**: click an arrow to select it, then click the ✕ at its head (or press Delete on a focused arrow).

`editable: true` also widens each arrow's hit area to at least 12 px (`hitWidth`) so the thin stroke can be clicked. Links between a summary task and its own descendants are refused unless `allowSummaryDescendantLinks: true`.

A drawn link goes through `addDependency()` and a deleted one through `removeDependency()`, so both are undoable, pass the `beforeDependencyChange` veto hook, and fire the same events as the API calls. Persist from those:

```js
import { GanttEvents } from 'apexgantt';

const gantt = new ApexGantt(container, { series: tasks, dependencies: { editable: true } });
gantt.render();

container.addEventListener(GanttEvents.DEPENDENCY_ADDED, (e) => {
  const { fromId, toId, type, lag } = e.detail;   // drawn links: type from the edges, lag 0
  // create on the backend
});

container.addEventListener(GanttEvents.DEPENDENCY_REMOVED, (e) => {
  const { fromId, toId } = e.detail;
  // delete on the backend
});
```

`type` is one of `'FF' | 'FS' | 'SF' | 'SS'`. `lag` is in days. `deleteTask()` also fires `DEPENDENCY_REMOVED` once for each edge it takes with the task.

Undo and redo do **not** fire these two events. They fire only `historyChange` (`kind: 'undo'` or `'redo'`), and its detail does not name the edge, so a backend synced from the two events alone drifts when a user undoes a link. If the backend must mirror every edge, turn history off with `history: { enabled: false }`.

Do not persist on `DEPENDENCY_ARROW_UPDATE`. It is an internal redraw signal, fired for every edge of a task that moves, so its `detail` names edges that were neither created nor removed, and it does not bubble.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `dependency: { id: 't1' }` | `dependency: { taskId: 't1' }` (key is `taskId`, not `id`) |
| Negative lag for "delay" | Negative lag is a **lead** (overlap). Positive lag is delay. |
| Self-dependency: `dependency: 't1'` on task `t1` | Always reference a different task. Self-deps create a cycle and are ignored. |
| Dependency cycle (A→B→A) | Tasks render but the arrow draw is suppressed for the cycle edge. Audit your data. |
| Critical path not highlighted | Set `enableCriticalPath: true`. The default is `false`. |

## Worked example

```js
import { ApexGantt } from 'apexgantt';

const tasks = [
  { id: 'design',  name: 'Design',     startTime: '01-01-2026', endTime: '01-15-2026', progress: 100 },
  { id: 'backend', name: 'Backend',    startTime: '01-16-2026', endTime: '02-10-2026', progress: 60,
    dependency: 'design' },
  { id: 'frontend',name: 'Frontend',   startTime: '01-20-2026', endTime: '02-15-2026', progress: 40,
    dependency: { taskId: 'design', type: 'FS', lag: 4 } },     // 4-day buffer
  { id: 'qa',      name: 'QA',         startTime: '02-16-2026', endTime: '03-01-2026', progress: 0,
    dependency: { taskId: 'frontend', type: 'FS' } },   // one predecessor per task
  { id: 'launch',  name: 'Launch',     startTime: '03-05-2026', type: 'milestone',
    dependency: 'qa' },
];

const gantt = new ApexGantt(document.getElementById('chart'), {
  series: tasks,
  enableCriticalPath: true,
});
gantt.render();
```

> Note: a task takes **one** `dependency` — either a task-ID string (Finish-to-Start, 0 lag) or a `TaskDependency` object `{ taskId, type, lag }`. The field type is `string | TaskDependency`, so arrays are **not** supported (an array value is read as a single dependency); give each task a single predecessor.
