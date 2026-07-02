# Editing — CRUD, Undo/Redo, Calendar, Sub-Day Scheduling, Interaction Toggles

Everything here is new in 3.12.0. All mutating APIs are recorded in the undo
history, fire an event on the container, and pass through an optional
synchronous veto hook.

## CRUD API

```js
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
// Both return false if beforeDependencyChange vetoed; addDependency throws on missing task / duplicate edge.

// Dry-run before committing (used by the interactive draw UI too)
const res = gantt.canAddDependency('t1', 't2', { type: 'FS' });
if (res.ok) gantt.addDependency('t1', 't2');
else console.log(res.reason);
// reason ∈ 'self' | 'task-missing' | 'duplicate' | 'cycle' | 'summary-descendant' | 'hook-veto'
```

The `summary-descendant` rule blocks linking a summary to its own descendant;
override with `dependencies.allowSummaryDescendantLinks: true`.

### CRUD events

Fire on the container element. All carry a `timestamp`.

| Constant | Event name | `detail` shape |
|---|---|---|
| `TASK_ADDED` | `taskAdded` | `{ taskId, task, parentId?, timestamp }` |
| `TASK_DELETED` | `taskDeleted` | `{ taskId, task, removedDescendantIds, timestamp }` (one event per removed task on cascade) |
| `TASK_MOVED` | `taskMoved` | `{ taskId, oldParentId?, newParentId?, timestamp }` |
| `DEPENDENCY_ADDED` | `dependencyAdded` | `{ fromId, toId, type, lag, timestamp }` |
| `DEPENDENCY_REMOVED` | `dependencyRemoved` | `{ fromId, toId, type, lag, timestamp }` (captured pre-removal values) |
| `TASK_PROGRESS_CHANGED` | `taskProgressChanged` | `{ taskId, oldProgress, newProgress, timestamp }` |

### Veto hooks

Synchronous. Return `false` to cancel the operation; any other return value
lets it proceed.

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

`beforeTaskUpdate` fires for every update path — drag, resize, progress drag,
inline edit, dialog edit, and `updateTask()`.

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
| `enableInlineEdit` | `false` | Double-click a `name`/`startTime`/`endTime`/`duration`/`progress` list cell to edit inline. Auto-on when `enableTaskEdit: true` unless set `false`. Commit with Enter/blur, cancel with Escape. Summary rows are not editable. |
| `enableProgressDrag` | `true` | Drag the handle at the bottom of a bar (visible on hover) to set progress; snaps to whole percent. Fires `taskProgressChanged`. |
| `enableTaskEditingShortcuts` | `false` | `Delete`/`Backspace` deletes the focused row (cascade `'children'`); `Tab`/`Shift+Tab` indents/outdents. Requires task-list focus. |
| `enableTaskCRUDToolbar` | `false` | `+ Add Task` and trash-`Delete` toolbar buttons. Delete acts on the current selection (needs `enableSelection: true`) with cascade `'children'`. |
| `enableContextMenu` | `false` | Right-click menu on bars / rows: Edit, Add child, Add sibling, Indent, Outdent, Delete. Entries are capability-gated (Edit only when `enableTaskEdit`; Indent/Outdent only when legal). |
| `enableAddTaskRow` | `false` | A `+ Add task` row below the last task; clicking inserts a root-level placeholder and fires `taskAdded`. Disabled while row virtualisation is active (dataset ≥ 50 rows). |

## Working calendar (`calendar`)

```js
import { ApexGantt } from 'apexgantt';

new ApexGantt(el, {
  inputDateFormat: 'MM-DD-YYYY',
  series: tasks,
  calendar: {
    workingWeekdays: [1, 2, 3, 4, 5],   // 0 = Sunday … 6 = Saturday. Default Mon–Fri.
    holidays: [
      '12-25-2026',                                   // plain date
      { date: '01-01-2027', label: 'New Year\'s Day' } // labelled → shows in holidayTooltip
    ],
    showNonWorkingStripes: true,        // default true — hatched bands over weekends/holidays
    holidayTooltip: ({ date, label }) => `<strong>${label ?? 'Holiday'}</strong>`,
    dragSnapMode: 'next',               // 'next' (default) | 'previous' | 'allow'
  },
});
```

- A configured calendar makes weekends + holidays drive duration math and summary aggregation.
- `dragSnapMode`: when a drag/resize commit lands on a non-working day, `'next'`/`'previous'` snap to the adjacent working day (drag preserves the source's working-day duration; resize snaps only the moving endpoint), `'allow'` permits ad-hoc non-working start/end.
- Dependency `lag` is interpreted in **working days** by default once a calendar is set. Override per-edge with `lagUnit: 'calendar'` on the `TaskDependency`.

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

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `deleteTask('parent')` on a task with children | Pass `{ cascade: 'children' }` (or `'orphan'`), else it throws. |
| Expecting `undo()` to work with `history.enabled: false` | It's a no-op; leave history enabled (the default). |
| Sub-day snap without time in `inputDateFormat` | Add `HH:mm` — `endTime` stays inclusive/day-based otherwise. |
| Building an external Undo button off polling | Listen for `historyChange` and read `canUndo`/`canRedo`. |
| `moveTask` to reorder siblings | Not supported yet; `moveTask` only re-parents. |
