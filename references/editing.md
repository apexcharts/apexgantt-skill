# Editing: CRUD, Undo/Redo, Calendar, Sub-Day Scheduling, Interaction Toggles

Most of this is new in 3.12.0; items marked 3.20.0+ need apexgantt 3.20.0. All
mutating APIs are recorded in the undo history and pass through an optional
synchronous veto hook. The CRUD methods (`addTask`, `deleteTask`, `moveTask`,
dependency changes) fire an event on the container; `updateTask` fires
`taskUpdate` / `taskUpdateSuccess` only from 3.20.0 (see below).

## CRUD API

```js
// Update (surgical patch). Throws if the id is unknown; runs beforeTaskUpdate.
gantt.updateTask('t1', { progress: 75 });
gantt.updateTask('t1', { progress: 60 }, { silent: true });   // 3.20.0+: no taskUpdate / taskUpdateSuccess

// Read (3.20.0+): copies with current values, edits included
gantt.getTask('t1');      // Task | null
gantt.getTasks();         // Task[], every task incl. collapsed and filtered-out, parent before children
gantt.getDependencies();  // DependencyLink[]: every link now, incl. added and drawn ones

// Expand / collapse (3.20.0+): view state only, not undoable, one render
gantt.expandTask('phase1');     // false for an unknown id or a task with no children
gantt.collapseTask('phase1');

// Add
gantt.addTask({ id: 't9', name: 'Review', startTime: '08-01-2026', endTime: '08-05-2026' });
gantt.addTask(
  { id: 'subA', name: 'Subtask', startTime: '08-01-2026', endTime: '08-03-2026' },
  { parentId: 't9' }
);                                     // → Task, or null if beforeTaskAdd vetoed. Throws on duplicate id.

// Delete
gantt.deleteTask('t9');                          // cascade: 'forbid' (default) — throws if it has children
gantt.deleteTask('t9', { cascade: 'children' }); // delete task + all descendants in one undoable transaction
gantt.deleteTask('t9', { cascade: 'orphan' });   // reparent children to t9's parent (or root), then remove t9
// Returns false if beforeTaskDelete vetoed. Dependency edges referencing removed tasks are auto-cleaned.

// Move (re-parent)
gantt.moveTask('subA', { newParentId: 't1' });
gantt.moveTask('subA', { newParentId: null });   // omit or null → move to root
// Returns false if beforeTaskMove vetoed. Throws on missing id or a cycle. Sibling reordering is not supported yet.

// Dependencies
gantt.addDependency('t1', 't2', { type: 'FS', lag: 2 });   // type default 'FS', lag default 0
gantt.removeDependency('t1', 't2');
// Both return false if beforeDependencyChange vetoed. addDependency throws on a missing task or a
// duplicate edge, and from 3.20.0 also on a self-link, a cycle and (unless allowSummaryDescendantLinks)
// a link between a task and its own ancestor or descendant.

// Dry-run before committing (used by the interactive draw UI too)
const res = gantt.canAddDependency('t1', 't2', { type: 'FS' });
if (res.ok) gantt.addDependency('t1', 't2');
else console.log(res.reason);
// reason ∈ 'self' | 'task-missing' | 'duplicate' | 'cycle' | 'summary-descendant' | 'hook-veto'
```

The `summary-descendant` rule blocks linking a summary to its own descendant;
override with `dependencies.allowSummaryDescendantLinks: true`. From 3.20.0 the
rule applies to `addDependency()` whether or not `dependencies.editable` is set
(links given in the data are not checked), and `addDependency()` throws for every
reason `canAddDependency()` reports except `'hook-veto'`, which returns `false`.
Through 3.19 it threw only for a missing task or a duplicate. Dependencies are
drawn and analysed, but never move tasks.

### `getTask` / `getTasks` / `getDependencies` (3.20.0+)

- `getTask(id)` and `getTasks()` return copies: `baseline`, `segments`,
  `assignees` and an object `dependency` are copied too, so changing a returned
  task never changes the chart (use `updateTask`). Custom fields you added are
  copied by reference.
- `getTasks()` is not a complete save. Each task carries only the `dependency`
  it was loaded with (removing that link clears it); links added with
  `addDependency()` or drawn by the user are not in it.
- `getDependencies()` returns every link as it is now, as `DependencyLink`
  copies `{ fromId, toId, type, lag }`: the ones loaded with the tasks, plus
  added and drawn ones, minus removed ones. Undo and redo are reflected.

Save both, and restore the links after loading:

```js
// Save
await saveChart({ tasks: gantt.getTasks(), links: gantt.getDependencies() });

// Load
const { tasks, links } = await loadChart();
const gantt = new ApexGantt(el, { series: tasks });
gantt.render();
for (const { fromId, toId, type, lag } of links) {
  // the links the tasks carry are already there, so canAddDependency skips them as duplicates
  if (gantt.canAddDependency(fromId, toId).ok) gantt.addDependency(fromId, toId, { type, lag });
}
gantt.clearHistory();   // so the restore cannot be undone
```

Each re-added link runs `beforeDependencyChange` and fires `dependencyAdded`.
To keep a server in step instead, record links from `dependencyAdded` and
`dependencyRemoved` and re-add them after loading the same way.

### `updateTask` details

- Through 3.19, `updateTask()` fires no events. From 3.20.0 it fires `taskUpdate`
  and `taskUpdateSuccess` with `source: 'api'`, unless `{ silent: true }` is
  passed (use that for rollbacks and server echoes so they are not saved back).
- From 3.20.0 a change to a task whose row is not rendered (under a collapsed
  parent, filtered out, or outside the virtual window past 50 rows) is stored
  like any other. Through 3.19 it is lost, though an undo step is still recorded,
  which breaks server-driven progress updates for hidden rows.
- `updateTask(id, { collapsed })` re-renders from 3.20.0 so rows appear or
  disappear at once. Through 3.19 it changes the model without re-rendering;
  prefer `expandTask` / `collapseTask` on 3.20.0+.
- `update()` is not a substitute: through 3.19 it rebuilds from the `series` you
  last passed and discards these edits. From 3.20.0 edits survive any `update()`
  that does not pass `series` or `parsing`.
- The patch is `Partial<TaskInput>` (3.20.0 typing), so `type: 'milestone'`
  compiles. From 3.20.0 the `updatedTask` in `taskUpdateSuccess` is the stored
  task after the rebuild, carrying its `level`, `wbs` and summary values, and the
  edit keeps the task's tree level. A `segments` patch with one entry turns a
  split task back into a plain one.
- An `updateTask`, `addTask` or `update({ series })` that throws leaves the
  previous data (and, for `update()`, the previous options) in place from
  3.20.0. Through 3.19 the chart is left half-built.

### Tree level (3.20.0+)

A committed edit (drag, `updateTask`, dialog, inline, keyboard) keeps the
task's tree level: `getTask().level`, `taskUpdateSuccess.updatedTask.level`, the
row indent and `aria-level` stay right after later renders such as
`update({ theme })` or collapse / expand. `moveTask` sets the moved subtree's
new depth, and `addTask` under a parent starts at that parent's depth plus one.

### CRUD events

Fire on the container element. All carry a `timestamp`.

| Constant | Event name | `detail` shape |
|---|---|---|
| `TASK_ADDED` | `taskAdded` | `{ taskId, task, parentId?, timestamp }` (`parentId`: the parent it landed under; 3.20.0+ also when given as `input.parentId`) |
| `TASK_DELETED` | `taskDeleted` | `{ taskId, task, removedDescendantIds, timestamp }` (one event per removed task on cascade) |
| `TASK_MOVED` | `taskMoved` | `{ taskId, oldParentId?, newParentId?, timestamp }` |
| `DEPENDENCY_ADDED` | `dependencyAdded` | `{ fromId, toId, type, lag, timestamp }` (also for a link the user draws) |
| `DEPENDENCY_REMOVED` | `dependencyRemoved` | `{ fromId, toId, type, lag, timestamp }` (captured pre-removal values; also when a deleted task takes its links with it) |
| `TASK_PROGRESS_CHANGED` | `taskProgressChanged` | `{ taskId, oldProgress, newProgress, timestamp }` |

### Veto hooks

Synchronous, and each takes **one context object** (not positional arguments).
Return `false` to cancel the operation; any other return value lets it proceed.
A returned `Promise` is not awaited and is not a veto, so confirm with the user
before calling the API, not inside the hook.

```js
new ApexGantt(el, {
  series: tasks,
  beforeTaskAdd:    ({ input, parentId }) => input.name.trim().length > 0,
  beforeTaskUpdate: ({ task, updates }) => !task.locked,
  beforeTaskMove:   ({ task, oldParentId, newParentId }) => true,
  beforeTaskDelete: ({ task, descendantIds, cascade }) => confirm(`Delete ${task.name}?`),
  beforeDependencyChange: ({ change, fromId, toId, type, lag }) => change === 'add',
});
```

`beforeTaskUpdate` receives `{ task, updates }`, where `task` is the task as
stored and `updates` the partial change. Which paths run it depends on the version:

- **Through 3.19**: only `updateTask()` and inline cell edits. Drags, resizes,
  progress drags and dialog saves bypass it.
- **From 3.20.0**: every edit path (drag, resize, progress drag, dialog save,
  inline edit, keyboard move/resize, `updateTask()`). Returning `false` vetoes
  the change and the bar is repainted from the model. `taskDragged`,
  `taskResized` and `taskProgressChanged` fire only after the commit is accepted.
  During a parent drag or keyboard move the hook is asked once per moved task: a
  veto of one child keeps only that child in place, while later siblings are
  still asked and move (and appear in `taskDragged.affectedChildTasks`). A veto
  of the parent skips all its children. Descendants with no `startTime`
  (undated summary rows) are not edited and are not asked.

`beforeTaskAdd` receives `{ input, parentId }` (the `TaskInput` about to be
inserted), not a `Task`. From 3.20.0 `parentId` is the parent the task lands
under: `addTask()`'s `parentId` option, else `input.parentId`, and `undefined`
for an id that matches no task. Through 3.19 `input.parentId` is ignored there,
so a hook that checks the parent is bypassed and `taskAdded` reports a root task.

## Undo / redo (`history`)

Every mutating call (drag, resize, progress drag, inline / dialog edit, add,
delete, move, dependency change, and the programmatic equivalents) is recorded.
Ctrl/Cmd+Z and Ctrl+Y / Ctrl/Cmd+Shift+Z work anywhere inside the chart (unless
focus is in a text input, where the browser's native undo takes over).

```js
new ApexGantt(el, {
  series: tasks,
  history: { enabled: true, maxSize: 100 },   // defaults; stack is FIFO-bounded by maxSize
});

gantt.undo();   // → true if a transaction was undone, false if the stack is empty / disabled
gantt.redo();
gantt.canUndo();          // gate your Undo button off this
gantt.canRedo();
gantt.getHistorySize();   // { undo, redo }
gantt.clearHistory();     // empty both stacks (fires historyChange kind: 'clear')
```

From 3.20.0 a parent drag or keyboard move is one undo step: the bar and its
descendants are committed as one batch, and undo / redo of any entry are applied
as one batch too (one data rebuild, not one per task).

Set `history: { enabled: false }` to opt out entirely — `undo()`/`redo()` become
no-ops and `canUndo()`/`canRedo()` return `false`. Useful when the host app runs
its own history layer. Any new mutation between an `undo()` and a `redo()`
discards the redo stack.

```js
container.addEventListener(GanttEvents.HISTORY_CHANGE, (e) => {
  const { kind, canUndo, canRedo, undoSize, redoSize, topUndoLabel } = e.detail;
  // kind ∈ 'record' | 'undo' | 'redo' | 'clear' — keep external Undo/Redo UI in sync
});
```

## Interaction toggles (user-facing editing)

The same operations, exposed through UI affordances. All default to the values below.

| Option | Default | What it enables |
|---|---|---|
| `enableInlineEdit` | `false` | Double-click a `name`/`startTime`/`endTime`/`duration`/`progress` list cell to edit inline. Auto-on when `enableTaskEdit: true` unless set `false`. Commit with Enter/blur, cancel with Escape. Summary rows are not editable, nor are a milestone's date, duration and progress cells. |
| `enableProgressDrag` | `true` | Drag the handle at the bottom of a bar (visible on hover) to set progress; snaps to whole percent. Fires `taskProgressChanged`. |
| `enableTaskEditingShortcuts` | `false` | `Delete`/`Backspace` deletes the focused row (cascade `'children'`); `Tab`/`Shift+Tab` indents/outdents. Requires task-list focus. |
| `enableTaskCRUDToolbar` | `false` | `+ Add Task` and trash-`Delete` toolbar buttons. Delete acts on the current selection (needs `enableSelection: true`) with cascade `'children'`. |
| `enableContextMenu` | `false` | Right-click menu on bars / rows: Edit, Add child, Add sibling, Indent, Outdent, Delete. Entries are capability-gated (Edit only when `enableTaskEdit`; Indent/Outdent only when legal). |
| `enableAddTaskRow` | `false` | A `+ Add task` row below the last task; clicking inserts a root-level placeholder and fires `taskAdded`. Hidden while more than 50 rows are visible (row virtualisation is on); collapsing or filtering back to 50 or fewer brings it back. |

`enableTaskEdit` (default `false`) opens the edit dialog on a bar double-click
(or Enter on a focused row). `enableTaskDrag` (default `true`) reschedules a bar
horizontally; it does not reorder rows.

**Keyboard on a focused bar (3.20.0+).** ArrowLeft / ArrowRight move the bar one
snap unit (`enableTaskDrag`); Shift+ArrowLeft / Shift+ArrowRight resize its end
(`enableTaskResize`, not milestones). Mirrored in RTL, recorded in undo, run
through `beforeTaskUpdate`, and reported with `source: 'keyboard'` (not
`taskDragged` / `taskResized`). Focus stays on the bar. With a calendar the keys
skip non-working days in the direction of travel (`dragSnapMode: 'allow'` is
honoured), a move also moves the bar's children, and each bar's
`aria-keyshortcuts` lists the keys that work for it. Escape cancels drawing a
task or a dependency wherever focus is.

**Inline editors (3.20.0+).** Typed spaces and Home / End stay in the editor.
Cancel, a veto or no change restores the cell with its expand arrow working. A
Start, End or Duration editor left without a change commits nothing: no
`taskUpdate` / `taskUpdateSuccess` and no undo step. Dates are compared by
value, so ISO-stored dates in a `MM-DD-YYYY` chart are not rewritten by an
untouched editor, and in working-day mode a task ending on a weekend (or running
only on weekend days) keeps its stored end. After every accepted edit the bar's
`aria-label`, `aria-valuenow` and `aria-valuetext` are refreshed.

## Working calendar (`calendar`)

```js
import { ApexGantt } from 'apexgantt';

new ApexGantt(el, {
  inputDateFormat: 'MM-DD-YYYY',
  series: tasks,
  calendar: {
    workingWeekdays: [1, 2, 3, 4, 5],   // 0 = Sunday ... 6 = Saturday. Default Mon-Fri.
    holidays: [
      '12-25-2026',                                   // plain date
      { date: '01-01-2027', label: 'New Year\'s Day' } // labelled: shows in holidayTooltip
    ],
    showNonWorkingStripes: true,        // default true: tinted bands over weekends/holidays
    holidayTooltip: ({ date, label }) => `<strong>${label ?? 'Holiday'}</strong>`,
    dragSnapMode: 'next',               // 'next' (default) | 'previous' | 'allow'
  },
});
```

- An active calendar makes weekends + holidays drive duration math and summary aggregation.
- **When it is active.** From 3.20.0 any `calendar` object you pass, even `{}`, turns it on with a Mon-Fri week. Through 3.19 it turns on only when `workingWeekdays` or `holidays` is set. No `calendar` option means every day is a working day.
- **Stripes and tooltip.** From 3.20.0 the stripes draw over the row backgrounds and under the bars, and `holidayTooltip` shows on hover. Through 3.19 the row backgrounds cover them and the tooltip does not show.
- **`update()`.** From 3.20.0 passing `calendar` or `inputDateFormat` to `update()` rebuilds the working calendar, and a theme change keeps it. Through 3.19 `update()` never rebuilds the calendar.
- **Holiday strings.** From 3.20.0 they are read with `inputDateFormat`, like task dates, falling back to ISO `YYYY-MM-DD`. In a `'DD-MM-YYYY'` chart `'03-04-2026'` is 3 April and `'25-12-2026'` is Christmas; through 3.19 strings go through the browser's lenient parser, so the first can read as 4 March and the second is dropped. Use `inputDateFormat` or ISO for holidays.
- **`workingWeekdays`.** From 3.20.0 numeric strings (`'1'`) count as numbers, and a list that names no weekday from 0 to 6 (`[]`, `[7]`) logs a warning and falls back to Monday to Friday. Through 3.19 such a list makes every day non-working.
- `dragSnapMode`: when a drag/resize commit lands on a non-working day, `'next'`/`'previous'` snap to the adjacent working day (drag preserves the source's working-day duration; resize snaps only the moving endpoint), `'allow'` permits ad-hoc non-working start/end.
- Dependency `lag` counts **working days** while a calendar is active and calendar days otherwise. It feeds the critical path and the arrow tooltip only; links never move tasks. `TaskDependency.lagUnit` is read by nothing, has no effect, and is deprecated in 3.20.0.
- From 3.20.0 the `BaselineVariance` column counts working days (suffix `wd`) while a calendar is active.

## Sub-day scheduling (`snapUnit` / `snapValue`)

Snap granularity for drag, resize, and inline edits. Independent of the timeline
header tier (which follows `pixelsPerDay`).

```js
new ApexGantt(el, {
  inputDateFormat: 'YYYY-MM-DD HH:mm',   // time tokens required for sub-day work
  snapUnit: 'minute',                    // 'day' (default) | 'hour' | 'minute'
  snapValue: 15,                         // multiplier: 15-minute steps
  series: [
    { id: 's1', name: 'Standup', startTime: '2026-06-01 09:00', endTime: '2026-06-01 09:15' },
  ],
});
```

- `'day'` (default): drags snap to whole days; `endTime` is inclusive (a 1-day task spans one cell).
- `'hour'` / `'minute'`: drags snap to whole hours / minutes; `endTime` is the **exclusive** end timestamp when `inputDateFormat` includes time tokens. `taskDragged.daysMoved` becomes fractional (a 6-hour move reports `0.25`).
- The Duration column shows base units from 3.20.0 (a 45-minute task reads `45 m` with `snapValue: 15`; working days, `5 wd`, when a calendar is set with day snapping), and its inline editor takes the same units with step `snapValue`. Through 3.19 the displayed number is divided by `snapValue`.
- The inline Duration editor follows the Duration column's rule for every date format (3.20.0+): with a calendar and day snapping it counts working days, so the value matches the `wd` cell; without a calendar, day snapping counts day boundaries; the new end keeps the current end's time of day.
- From 3.20.0 keyboard and mouse moves and resizes with `snapUnit: 'day'` step by calendar days. In a time zone with daylight saving a one-day step across the clock change lands on the next date; through 3.19 it can leave the date unchanged (the task stretches instead of moving) or jump two days. Hour and minute snaps still move by elapsed time.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `beforeTaskUpdate: (taskId, updates) => ...` | One object: `({ task, updates }) => ...` |
| `beforeTaskUpdate: async () => await confirm()` | Hooks are synchronous; a `Promise` is not a veto |
| Expecting `beforeTaskUpdate` to see drags on 3.19 | Only from 3.20.0; on 3.19 react to `taskDragged` instead |
| `deleteTask('parent')` on a task with children | Pass `{ cascade: 'children' }` (or `'orphan'`), else it throws. |
| `removeTask(id)` | The method is `deleteTask(id, { cascade })`. |
| Expecting `undo()` to work with `history.enabled: false` | It's a no-op; leave history enabled (the default). |
| Sub-day snap without time in `inputDateFormat` | Add `HH:mm` — `endTime` stays inclusive/day-based otherwise. |
| Building an external Undo button off polling | Listen for `historyChange` and read `canUndo`/`canRedo`. |
| `moveTask` to reorder siblings | Not supported yet; `moveTask` only re-parents. |
| `addDependency(a, b)` without checking (3.20.0+ throws on self, cycle, summary-descendant) | `if (gantt.canAddDependency(a, b).ok) gantt.addDependency(a, b)` |
| Saving only `getTasks()` and losing added or drawn links | Save `getDependencies()` too and re-add the links after loading |