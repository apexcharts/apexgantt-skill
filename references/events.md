# Events, Selection, Drag, Resize, Edit Dialog

## Where events fire

ApexGantt dispatches `CustomEvent`s on the **container element** passed to the constructor. The chart instance is **not** an `EventTarget`: `gantt.addEventListener(...)` is a TypeError.

Use the `GanttEvents` constants (or the literal string names) to subscribe.

```js
import { ApexGantt, GanttEvents } from 'apexgantt';

const container = document.getElementById('chart');
const gantt = new ApexGantt(container, { series: tasks, enableSelection: true });
gantt.render();

container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, (e) => {
  const { taskId, updatedTask, source, timestamp } = e.detail;   // source: 3.20.0+
  // persist to backend, etc.
});
```

## Event reference

| Constant | Event name | `detail` shape | Fires when |
|---|---|---|---|
| `TASK_UPDATE` | `taskUpdate` | `{ taskId, updates, updatedTask, source, timestamp }` | An edit is being committed. Through 3.19 only the edit dialog and inline cell edits fire it. From 3.20.0 it fires once per committed edit on every path. `source` is 3.20.0+. |
| `TASK_VALIDATION_ERROR` | `taskValidationError` | `{ taskId, errors: { field, message }[], timestamp }` | The edit dialog failed validation. Nothing is applied. |
| `TASK_UPDATE_SUCCESS` | `taskUpdateSuccess` | `{ taskId, updatedTask, source, timestamp }` | The edit was applied. Same coverage as `taskUpdate`. `source` is 3.20.0+. |
| `TASK_UPDATE_ERROR` | `taskUpdateError` | `{ taskId, error, timestamp }` | Applying an edit threw. From 3.20.0 on every edit path: `updateTask` still throws afterwards, interactive paths log, repaint and return. |
| `TASK_DRAGGED` | `taskDragged` | `{ taskId, oldStartTime, oldEndTime, newStartTime, newEndTime, daysMoved, affectedChildTasks, timestamp }` | Bar dropped at a new horizontal position. From 3.20.0 only after the commit is accepted. |
| `TASK_RESIZED` | `taskResized` | `{ taskId, resizeHandle: 'left'\|'right', oldStartTime, oldEndTime, newStartTime, newEndTime, durationChange, timestamp }` | Bar resized via a handle. From 3.20.0 only after the commit is accepted. |
| `TASK_PROGRESS_CHANGED` | `taskProgressChanged` | `{ taskId, oldProgress, newProgress, timestamp }` | In-bar progress handle dragged. From 3.20.0 only after the commit is accepted. |
| `SELECTION_CHANGE` | `selectionChange` | `{ selectedTasks, selectedIds, timestamp }` | Selection set changed. |
| `DEPENDENCY_ADDED` | `dependencyAdded` | `{ fromId, toId, type, lag, timestamp }` | Edge created: `addDependency()`, or a link drawn with `dependencies.editable: true`. |
| `DEPENDENCY_REMOVED` | `dependencyRemoved` | `{ fromId, toId, type, lag, timestamp }` (pre-removal values) | Edge removed: `removeDependency()`, the arrow's ✕ or Delete key, or `deleteTask()` taking the edge with it. Not fired by undo/redo. |
| `DEPENDENCY_ARROW_UPDATE` | `dependencyArrowUpdate` | `{ fromId, toId, type, lag, chartInstanceId, arrowLinkInstanceId }` | Internal redraw signal for each edge of a task that moved. Not a create/remove event; does not bubble. Do not persist from it. |

All `*Time` strings are formatted using `options.inputDateFormat`. CRUD, history, sort, filter, group and column events are listed in `references/editing.md` and `references/grid.md`.

### `source` (3.20.0+)

`taskUpdate` and `taskUpdateSuccess` carry `source: TaskUpdateSource`:

| `source` | Edit path |
|---|---|
| `'drag'` | Bar dragged |
| `'resize'` | Bar resized |
| `'progress'` | Progress handle dragged |
| `'dialog'` | Edit dialog saved |
| `'inline'` | Task-list cell edited |
| `'keyboard'` | Arrow keys on a focused bar |
| `'api'` | `gantt.updateTask()` (and `splitTask()`, which routes through it) |

Neither event fires for undo/redo, or for `updateTask(id, patch, { silent: true })`.

`updatedTask` is the stored task after the rebuild (3.20.0+), so its `level`, `wbs` and summary values are current.

### Order of events in a parent drag or keyboard move (3.20.0+)

Dragging a parent drawn as an ordinary bar (`showSummaryBar: false`), or moving it with the arrow keys, commits the bar and its descendants as one batch:

1. `beforeTaskUpdate` per task. A veto of one child keeps only that child in place; later siblings are still asked and move. A veto of the parent skips all its children.
2. `taskUpdate` for every accepted task.
3. One data rebuild (summary rows, boundary lines, critical path, render).
4. `taskUpdateSuccess` for every accepted task, then `taskDragged` for a mouse drag.

All the gesture's history entries merge into one undo step. A listener that expects each `taskUpdate` to be followed by its own `taskUpdateSuccess` before the next one starts must not assume that order here. Descendants with no `startTime` (undated summary rows) are skipped, so they get no events.

## Drag & resize

Both are on by default. `enableTaskDrag` reschedules a bar horizontally (new dates, same row); it does not reorder rows, and it does not draw links (that is `dependencies.editable`). Disable when you don't want users moving tasks:

```js
new ApexGantt(el, {
  enableTaskDrag: false,
  enableTaskResize: false,
  series: tasks,
});
```

From 3.20.0 a focused bar also answers the keyboard: ArrowLeft / ArrowRight move it one snap unit (`enableTaskDrag`), Shift+ArrowLeft / Shift+ArrowRight resize its end (`enableTaskResize`, not milestones). Both are mirrored in RTL and go through the same commit path as dragging (`beforeTaskUpdate`, undo, events with `source: 'keyboard'`; no `taskDragged` / `taskResized`).

From 3.20.0 every accepted edit (drag, resize, progress drag, dialog, inline, `updateTask`) refreshes the bar's `aria-label`, `aria-valuenow` and `aria-valuetext`, and under `locale.direction: 'rtl'` the progress handle and its percent label stay centred on the fill edge after a progress drag, `updateTask({ progress })` and undo / redo.

Summary bars (the default for any parent) cannot be dragged. A parent drawn as an ordinary bar (`showSummaryBar: false`) can, and its children shift with it. Their new positions arrive in the event detail's `affectedChildTasks` array:

```js
container.addEventListener(GanttEvents.TASK_DRAGGED, (e) => {
  const { taskId, newStartTime, newEndTime, affectedChildTasks } = e.detail;
  // affectedChildTasks: [{ taskId, newStartTime, newEndTime }, ...]
});
```

Dragging never moves linked tasks: dependencies are not enforced.

## Edit dialog and inline edit

Off by default. Opt in with `enableTaskEdit: true`:

```js
new ApexGantt(el, {
  enableTaskEdit: true,
  series: tasks,
});
```

Double-clicking a task bar (or pressing Enter on a focused row) opens the edit dialog. `enableTaskEdit` also turns on `enableInlineEdit` (double-click a Name / Start / End / Duration / Progress cell) unless you set `enableInlineEdit: false`. The dialog fires:

1. `taskValidationError` if the user submits invalid data (nothing is applied)
2. `taskUpdate` when valid changes are committed
3. `taskUpdateSuccess` *(or `taskUpdateError`)* on completion

`beforeTaskUpdate` can veto the change. Through 3.19 it runs for inline edits and `updateTask()` only; from 3.20.0 it also runs for the dialog, drag, resize, progress drag and keyboard moves.

**Dialog fixes in 3.20.0.** The date fields read ISO and other formats (through 3.19 a date not in `inputDateFormat` showed an empty field); a date that cannot be read shows an empty field and never writes "Invalid Date"; a format with time uses a `datetime-local` input. A save vetoed by `beforeTaskUpdate` keeps the dialog open, and a submit with no changes commits nothing and fires no events. Inline editors behave the same way: one left without a change commits nothing (see `references/editing.md`).

## Selection

`enableSelection: true` is **required** for selection APIs and the `selectionChange` event. Without it the methods do nothing (from 3.20.0 `getSelectedTasks`, `setSelectedTasks`, `clearSelection` and `setState({ selected })` also log a one-time warning).

```js
const gantt = new ApexGantt(el, {
  enableSelection: true,
  showCheckboxColumn: true,        // checkbox column for multi-select; default true
  series: tasks,
});
gantt.render();

container.addEventListener(GanttEvents.SELECTION_CHANGE, (e) => {
  const { selectedIds, selectedTasks } = e.detail;
});

// Programmatic API
gantt.getSelectedTasks();          // Task[]
gantt.setSelectedTasks(['t1','t3']);
gantt.clearSelection();
```

Selection supports `Click`, `Ctrl+Click` (toggle), `Shift+Click` (range), and arrow-key keyboard navigation.

## Persisting changes to a backend

**3.20.0+.** Every committed edit (drag, resize, progress, dialog, inline, keyboard, `updateTask`) fires one `taskUpdateSuccess`, so one listener covers them all. Do not also save from `taskDragged` / `taskResized` / `taskProgressChanged`, or each drag is saved twice. Roll a failed save back with `updateTask(..., { silent: true })`: a silent update fires no `taskUpdateSuccess`, so the rollback does not loop back into your save handler.

```js
// Last state the server confirmed, per task.
const confirmed = new Map(tasks.map((t) => [t.id, t]));

container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, async (e) => {
  const { taskId, updatedTask, source } = e.detail;
  try {
    const res = await fetch(`/api/tasks/${taskId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(updatedTask),
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    confirmed.set(taskId, updatedTask);
  } catch (err) {
    const prev = confirmed.get(taskId);
    if (prev) {
      const { startTime, endTime, progress, name } = prev;
      gantt.updateTask(taskId, { startTime, endTime, progress, name }, { silent: true });
    }
  }
});
```

Use `{ silent: true }` the same way when applying a server echo or a push from another client, so it is not saved back. `gantt.getTask(id)` (3.20.0+) returns the current stored values if you need them.

Links are not task edits: persist them from `dependencyAdded` / `dependencyRemoved` (they fire for drawn links, arrow deletes and links a deleted task takes with it, but not for undo / redo), or save `gantt.getDependencies()` (3.20.0+) next to `gantt.getTasks()`. `getTasks()` alone holds only the links each task was loaded with. See `references/editing.md` for the reload recipe.

**Through 3.19.** `taskUpdateSuccess` fires only for dialog and inline edits, and `updateTask()` fires nothing. Save drags, resizes and progress drags from `taskDragged` / `taskResized` / `taskProgressChanged`, and dialog / inline edits from `taskUpdateSuccess`. A plain `gantt.updateTask(taskId, prev)` is already silent there. Also note that through 3.19 a change to a task whose row is not rendered (under a collapsed parent, filtered out, or outside the virtual window past 50 rows) is lost.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `gantt.addEventListener('taskDragged', h)` | `container.addEventListener(GanttEvents.TASK_DRAGGED, h)` |
| Saving from both `taskDragged` and `taskUpdateSuccess` on 3.20.0+ | Save from `taskUpdateSuccess` alone; branch on `source` if needed |
| Rolling back with a plain `updateTask()` on 3.20.0+ (fires another save) | `updateTask(id, prev, { silent: true })` |
| Assuming each `taskUpdate` is followed by its own `taskUpdateSuccess` during a parent drag (3.20.0+) | All `taskUpdate` events come first; read the final values in `taskUpdateSuccess` |
| Returning a `Promise` from `beforeTaskUpdate` to veto | Hooks are synchronous; return `false` |
| Expecting `enableTaskDrag` to reorder rows | It reschedules horizontally; rows follow `sortBy` |
| Selecting via DOM clicks while `enableSelection` is `false` | Set `enableSelection: true` before using selection APIs |
| Treating `selectedIds` array as a `Set` | It's a plain array; deduplicate yourself if needed |
| Forgetting to remove listeners on unmount | Always pair `addEventListener` with `removeEventListener` in cleanup |
