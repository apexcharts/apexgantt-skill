# Events, Selection, Drag, Resize, Inline Edit

## Where events fire

ApexGantt dispatches `CustomEvent`s on the **container element** passed to the constructor. The chart instance is **not** an `EventTarget` — `gantt.addEventListener(...)` is a TypeError.

Use the `GanttEvents` constants (or the literal string names) to subscribe.

```js
import { ApexGantt, GanttEvents } from 'apexgantt';

const container = document.getElementById('chart');
const gantt = new ApexGantt(container, { series: tasks, enableSelection: true });
gantt.render();

container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, (e) => {
  const { taskId, updatedTask, timestamp } = e.detail;
  // persist to backend, etc.
});
```

## Event reference

| Constant | Event name | `detail` shape | Fires when |
|---|---|---|---|
| `TASK_UPDATE` | `taskUpdate` | `{ taskId, updates, updatedTask, timestamp }` | Update submitted (before validation). |
| `TASK_VALIDATION_ERROR` | `taskValidationError` | `{ taskId, errors: { field, message }[], timestamp }` | Inline-edit form failed validation. |
| `TASK_UPDATE_SUCCESS` | `taskUpdateSuccess` | `{ taskId, updatedTask, timestamp }` | Update completed without error. |
| `TASK_UPDATE_ERROR` | `taskUpdateError` | `{ taskId, error, timestamp }` | Update threw. |
| `TASK_DRAGGED` | `taskDragged` | `{ taskId, oldStartTime, oldEndTime, newStartTime, newEndTime, daysMoved, affectedChildTasks, timestamp }` | Bar dropped at a new horizontal position. |
| `TASK_RESIZED` | `taskResized` | `{ taskId, resizeHandle: 'left'\|'right', oldStartTime, oldEndTime, newStartTime, newEndTime, durationChange, timestamp }` | Bar resized via a handle. |
| `SELECTION_CHANGE` | `selectionChange` | `{ selectedTasks, selectedIds, timestamp }` | Selection set changed. |
| `DEPENDENCY_ARROW_UPDATE` | `dependencyArrowUpdate` | `{ fromId, toId, type, lag, chartInstanceId, arrowLinkInstanceId }` | Dependency arrow created / updated / removed. |

All `*Time` strings are formatted using `options.inputDateFormat`.

## Drag & resize

Both are on by default. Disable when you don't want users moving tasks:

```js
new ApexGantt(el, {
  enableTaskDrag: false,
  enableTaskResize: false,
  series: tasks,
});
```

When a parent task is dragged, all its children shift with it. Their new positions arrive in the event detail's `affectedChildTasks` array:

```js
container.addEventListener(GanttEvents.TASK_DRAGGED, (e) => {
  const { taskId, newStartTime, newEndTime, affectedChildTasks } = e.detail;
  // affectedChildTasks: [{ taskId, newStartTime, newEndTime }, …]
});
```

## Inline edit

Off by default. Opt-in with `enableTaskEdit: true`:

```js
new ApexGantt(el, {
  enableTaskEdit: true,
  series: tasks,
});
```

A click on a task row opens an inline edit form. The form fires:

1. `taskValidationError` — if the user submits invalid data
2. `taskUpdate` — when valid changes are submitted (before commit)
3. `taskUpdateSuccess` *(or `taskUpdateError`)* — on completion

## Selection

`enableSelection: true` is **required** for selection APIs and the `selectionChange` event.

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

Common pattern — listen for `TASK_UPDATE_SUCCESS` after drag / resize / edit and POST the change:

```js
container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, async (e) => {
  const { taskId, updatedTask } = e.detail;
  try {
    await fetch(`/api/tasks/${taskId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(updatedTask),
    });
  } catch (err) {
    // optimistic update — roll back via gantt.updateTask(taskId, originalState)
  }
});
```

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `gantt.addEventListener('taskDragged', h)` | `container.addEventListener(GanttEvents.TASK_DRAGGED, h)` |
| Selecting via DOM clicks while `enableSelection` is `false` | Set `enableSelection: true` before using selection APIs |
| Treating `selectedIds` array as a `Set` | It's a plain array; deduplicate yourself if needed |
| Forgetting to remove listeners on unmount | Always pair `addEventListener` with `removeEventListener` in cleanup |
