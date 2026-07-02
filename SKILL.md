---
name: apexgantt
description: >
  AI skill for building ApexGantt timeline and Gantt charts. Use whenever the
  user asks to create, configure, render, update, or troubleshoot a Gantt chart,
  task timeline, project schedule, milestone view, dependency arrows, critical
  path, or baseline-vs-actual visualization with `apexgantt`. Covers task data
  format, dependency types (`FS` / `SS` / `FF` / `SF`), date parsing, view
  modes, the `update()` / `updateTask()` lifecycle, custom toolbar items,
  selection, and framework integration (React / Vue / Angular). In React /
  Vue / Angular projects, prefer the framework wrapper packages
  (`react-apexgantt`, `vue-apexgantt`, `ngx-apexgantt`) over the core API.
metadata:
  author: ApexCharts
  version: "1.3.0"
  library_version: "3.12.0"
  category: data-visualization
  tags: [gantt, timeline, project-management, scheduling, charts, svg, apexgantt]
  docs: https://apexcharts.com/docs/apexgantt/
  npm: apexgantt
  github: https://github.com/apexcharts/apexgantt
---

# ApexGantt AI Skill

> **Framework wrapper detection — check `package.json` before generating code.**
> - `react` → use **`react-apexgantt`** instead of the core API.
> - `vue` → use **`vue-apexgantt`**.
> - `@angular/core` → use **`ngx-apexgantt`**.
>
> Wrappers handle `destroy()` automatically on unmount, accept reactive props, and forward events as idiomatic framework events. Use the core API directly only when no framework is detected, or when the user explicitly asks for vanilla. See `references/framework-wrappers.md`.

## 1. Critical Rules

1. **Always call `gantt.render()`** after `new ApexGantt(el, options)`. The constructor builds internal state but does not paint the DOM.
2. **Always call `gantt.destroy()`** before unmounting in React / Vue / Angular. The chart attaches `ResizeObserver`s and global event listeners — skipping `destroy()` causes memory leaks and "ghost" event handlers.
3. **`series` is the task array (required).** It is **NOT** optional and **NOT** the same shape as ApexCharts series. Each task is an object with `id`, `name`, `startTime`, and (usually) `endTime` — see the data format in §2.
4. **Date strings must match `inputDateFormat`.** The default is `'MM-DD-YYYY'`. If you pass `'2026-06-01'` (ISO), set `inputDateFormat: 'YYYY-MM-DD'` or the parse will fail silently and the bar won't render.
5. **`id` must be unique and stable across renders.** Selection, dependency arrows, and the diff-update path all key off `id`. Re-using `id`s breaks dependency drawing and selection state.
6. **`parentId` references must point to an existing task `id`.** Orphan parents are dropped silently from the tree.
7. **Milestones omit `endTime`.** A task with `type: TaskType.Milestone` (or `'milestone'`) renders as a diamond at `startTime`; do not pass `endTime` for milestones.
8. **`progress` is 0–100, not 0–1.** Values outside the range are clamped.
9. **Dependencies accept two shapes**: a plain task `id` string (treated as `FS` with `lag: 0`) or a `TaskDependency` object `{ taskId, type, lag }`. Mixing the two on the same task is fine.
10. **Use `update()` for full reconfigures**, `updateTask(id, patch)` for surgical single-task changes. `updateTask` skips a full re-layout and is much faster for streaming progress updates.
11. **For non-standard data shapes use `parsing`, not manual `.map()`.** The `parsing` config maps your fields → `TaskInput` fields and supports dot-notation paths and `transform` functions.
12. **`enableSelection: true` is required** for `getSelectedTasks()` / `setSelectedTasks()` / `clearSelection()` and the `selectionChange` event to fire.
13. **Custom toolbar items live in `toolbarItems`**, not in a separate render call. Use `position: 'left'` to put them before the built-in zoom/export controls.
14. **Events fire on the host container element**, not on the `gantt` instance. Listen via `container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, …)`.

---

## 2. Task Data Format (`series`)

`series` is an array of `TaskInput` objects:

```ts
{
  id: string;                       // REQUIRED, unique, stable
  name: string;                     // REQUIRED, display label
  startTime: string;                // REQUIRED, parsed by inputDateFormat
  endTime?: string;                 // omit for milestones
  progress?: number;                // 0–100, default 0
  type?: 'task' | 'milestone';      // default 'task'
  parentId?: string;                // hierarchical nesting
  dependency?: string | TaskDependency;
  baseline?: { start: string; end: string };
  barBackgroundColor?: string;      // per-task override
  rowBackgroundColor?: string;      // per-row override
  collapsed?: boolean;              // hide children
}
```

Minimal example:

```js
const gantt = new ApexGantt(document.getElementById('chart'), {
  series: [
    { id: 't1', name: 'Design',   startTime: '01-01-2026', endTime: '01-15-2026', progress: 60 },
    { id: 't2', name: 'Develop',  startTime: '01-16-2026', endTime: '02-15-2026', progress: 25, dependency: 't1' },
    { id: 'm1', name: 'Launch',   startTime: '02-20-2026', type: 'milestone' }
  ]
});
gantt.render();
```

### Dependency variants

| Shape | Meaning |
|---|---|
| `dependency: 't1'` | Plain id string. `FS` with 0-day lag. |
| `dependency: { taskId: 't1' }` | `FS` with 0-day lag (verbose). |
| `dependency: { taskId: 't1', type: 'SS', lag: 2 }` | Start-to-Start, 2-day lag (delay). |
| `dependency: { taskId: 't1', type: 'FS', lag: -1 }` | Finish-to-Start, 1-day lead (overlap). |

`type` values: `'FS'` (Finish-to-Start, default), `'SS'` (Start-to-Start), `'FF'` (Finish-to-Finish), `'SF'` (Start-to-Finish).

### Hierarchy

Use `parentId` to nest. Parent rows automatically aggregate children's date range and render an expand/collapse caret. Set `collapsed: true` on the parent to start collapsed.

```js
series: [
  { id: 'a',  name: 'Phase 1',                                                      startTime: '01-01-2026', endTime: '02-01-2026' },
  { id: 'a1', name: 'Subtask 1.1', parentId: 'a',                                   startTime: '01-01-2026', endTime: '01-15-2026' },
  { id: 'a2', name: 'Subtask 1.2', parentId: 'a',                                   startTime: '01-16-2026', endTime: '02-01-2026' },
]
```

### Baseline (planned vs. actual)

Enable globally, then attach `baseline: { start, end }` to any task you want to compare:

```js
const gantt = new ApexGantt(el, {
  baseline: { enabled: true, color: '#b0b8c1' },
  series: [
    { id: 't1', name: 'Design', startTime: '01-01-2026', endTime: '01-20-2026',
      baseline: { start: '01-01-2026', end: '01-15-2026' } }
  ]
});
```

---

## 3. Top-Level Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `series` | `TaskInput[]` | **required** | Task data. |
| `theme` | `'light' \| 'dark'` | `'light'` | Built-in palette. |
| `pixelsPerDay` | `number` | auto-fit | Initial zoom as pixels-per-day (continuous; header tiers auto-chosen). Reference values: Year ≈ `0.5`, Quarter ≈ `1.6`, Month ≈ `4.9`, Week ≈ `25.7`, Day = `80`. Omit to auto-fit the data range. |
| `inputDateFormat` | `string` | `'MM-DD-YYYY'` | dayjs format for `startTime`/`endTime`. |
| `snapUnit` | `'day' \| 'hour' \| 'minute'` | `'day'` | Granularity that drag / resize / inline edits snap to. `'hour'`/`'minute'` enable sub-day scheduling. See §10. |
| `snapValue` | `number` | `1` | Multiplier on `snapUnit` (e.g. `'minute'` + `15` → 15-min steps). |
| `calendar` | `CalendarOptions` | none | Working-calendar (weekends, holidays, non-working stripes, drag snap). See §10. |
| `history` | `{ enabled, maxSize }` | `{ enabled: true, maxSize: 100 }` | Undo/redo stack config. See §11. |
| `width` / `height` | `number \| string` | `'100%'` / `500` | Pixel number or CSS string. |
| `rowHeight` | `number` | `28` | px |
| `tasksContainerWidth` | `number` | `425` | Initial task-list panel width in px. |
| `columnConfig` | `ColumnListItem[]` | (all) | Authoritative; only listed columns render. |
| `enableTaskDrag` | `boolean` | `true` | Reorder rows by dragging. |
| `enableTaskResize` | `boolean` | `true` | Resize bars by dragging handles. |
| `enableTaskEdit` | `boolean` | `false` | Inline edit form on row click. |
| `enableInlineEdit` | `boolean` | `false` | Edit `name`/`startTime`/`endTime`/`duration`/`progress` directly in list cells. Auto-on when `enableTaskEdit` is `true` unless set `false`. |
| `enableProgressDrag` | `boolean` | `true` | Drag the in-bar handle to set progress. Fires `taskProgressChanged`. |
| `enableSelection` | `boolean` | `false` | Multi-row selection. Required for selection APIs. |
| `enableTaskEditingShortcuts` | `boolean` | `false` | `Delete`/`Backspace` delete focused row; `Tab`/`Shift+Tab` indent/outdent. Needs list focus. |
| `enableTaskCRUDToolbar` | `boolean` | `false` | Built-in `+ Add Task` / trash `Delete` toolbar buttons. Delete needs `enableSelection`. |
| `enableContextMenu` | `boolean` | `false` | Right-click menu: Edit, Add child/sibling, Indent, Outdent, Delete (capability-gated). |
| `enableAddTaskRow` | `boolean` | `false` | `+ Add task` row at the bottom of the list. Disabled while virtualising (≥ 50 rows). |
| `enableCriticalPath` | `boolean` | `false` | Compute & highlight CPM through dependencies. |
| `enableRollups` | `boolean` | `false` | Thin rollup markers under summary bars at each leaf's range (visible even when collapsed). |
| `enableProjectBoundary` | `boolean` | `false` | Two vertical lines at the project's earliest start / latest end. |
| `projectBoundaryColor` | `string` | `'#7C3AED'` | Stroke for the boundary lines. Falls back to `annotationBorderColor`. |
| `enableCrosshair` | `boolean` | `false` | Vertical cursor-following line + date/time label. |
| `crosshairColor` | `string` | theme accent | Crosshair line and label-background color. |
| `crosshairLabelFormat` | `(date, tier) => string` | auto | Custom crosshair label formatter; `tier` is the active header tier. |
| `barLabel` | `BarLabelOptions` | `{ position: 'right' }` | Per-task bar label: `position` `'right'\|'inside'\|'left'\|'auto'`, `field`, `render`, `className`, `leadingPadding`. |
| `columnLines` | `boolean` | `true` | Draw vertical lines between timeline columns. `false` keeps only row dividers. |
| `baseline` | `{ enabled, color }` | disabled | See §2. |
| `parsing` | `ParsingConfig` | — | Field-mapping to avoid manual transforms. |
| `toolbarItems` | `ToolbarItem[]` | `[]` | Custom toolbar buttons / selects / separators. |
| `tooltipTemplate` | `(task, fmt) => string` | built-in | HTML string returned per task. |
| `annotations` | `Annotation[]` | `[]` | Vertical/horizontal markers on the timeline. |

---

## 4. Lifecycle Pattern

```js
import { ApexGantt } from 'apexgantt';

const gantt = new ApexGantt(document.getElementById('chart'), {
  series: tasks,
  pixelsPerDay: 25.7,                        // ≈ week density; omit to auto-fit
  inputDateFormat: 'YYYY-MM-DD',
});

gantt.render();                              // REQUIRED — paint to DOM

gantt.update({ pixelsPerDay: 80 });          // merge new options + re-render (80 = day density)
gantt.updateTask('t1', { progress: 75 });    // surgical single-task patch
gantt.zoomIn();                              // raise pixelsPerDay (toward day-level detail)
gantt.zoomOut();                             // lower pixelsPerDay (toward year overview)

gantt.destroy();                             // free observers + DOM
```

`update()` preserves scroll position, collapsed/expanded state, and current zoom (`pixelsPerDay`) unless you explicitly override them.

---

## 5. Public API

### Instance methods

| Method | Description |
|---|---|
| `render(_data?)` | Paints the chart. Returns the container `HTMLElement`. Call once after construction. |
| `update(options)` | Merge new options & re-render. Unspecified keys keep current values. |
| `updateTask(id, patch)` | Surgical single-task update. Throws if `id` not found. |
| `addTask(input, { parentId? })` | Insert a task (recorded in history). Returns the `Task`, or `null` if `beforeTaskAdd` vetoed. Throws on duplicate `id`. |
| `deleteTask(id, { cascade? })` | Remove a task. `cascade`: `'forbid'` (default, throws if it has children), `'children'`, `'orphan'`. Returns `false` if `beforeTaskDelete` vetoed. |
| `moveTask(id, { newParentId? })` | Re-parent (`null`/omit → root). Returns `false` if `beforeTaskMove` vetoed. Throws on cycle / missing id. |
| `addDependency(fromId, toId, { type?, lag? })` | Create an edge. Returns `false` if `beforeDependencyChange` vetoed. Throws on missing task / duplicate. |
| `removeDependency(fromId, toId)` | Remove an edge. Returns `false` if vetoed. Throws when no edge exists. |
| `canAddDependency(fromId, toId, opts?)` | Dry-run check. Returns `{ ok: true }` or `{ ok: false, reason }` (`'self'`/`'task-missing'`/`'duplicate'`/`'cycle'`/`'summary-descendant'`/`'hook-veto'`). |
| `undo()` / `redo()` | Traverse the history stack. No-op (returns `false`) when the stack is empty or `history.enabled` is `false`. |
| `canUndo()` / `canRedo()` | Whether a transaction is available to undo / redo. Gate toolbar buttons off these. |
| `clearHistory()` | Empty both stacks. Fires `historyChange` with `kind: 'clear'`. |
| `getHistorySize()` | `{ undo, redo }` stack sizes. |
| `zoomIn()` / `zoomOut()` | Step through `Day → Week → Month → Quarter → Year`. |
| `getSelectedTasks()` | Array of selected `Task` objects. Requires `enableSelection: true`. |
| `setSelectedTasks(ids)` | Replace selection by id array. |
| `clearSelection()` | Clear selection state. |
| `renderToolbar(container)` | Render the built-in toolbar into a custom DOM slot. |
| `destroy()` | Tear down. Required before unmounting in SPA frameworks. |
| `isDestroyed()` | Guard before calling other methods. |

### Static method

| Method | Description |
|---|---|
| `ApexGantt.setLicense(key)` | Set the global license key once at app startup. Without it the chart renders with a watermark. |

---

## 6. Events

Events are dispatched as `CustomEvent`s on the **container element**, not on the chart instance. Use the `GanttEvents` constants (or the string event names) to subscribe.

```js
import { ApexGantt, GanttEvents } from 'apexgantt';

const container = document.getElementById('chart');
const gantt = new ApexGantt(container, { series: tasks });
gantt.render();

container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, (e) => {
  const { taskId, updatedTask, timestamp } = e.detail;
  // persist to backend, update local state, etc.
});
container.addEventListener(GanttEvents.TASK_DRAGGED, (e) => {
  const { taskId, newStartTime, newEndTime, daysMoved, affectedChildTasks } = e.detail;
});
```

| Constant | Event name | When |
|---|---|---|
| `GanttEvents.TASK_UPDATE` | `taskUpdate` | Update submitted (before validation). |
| `GanttEvents.TASK_VALIDATION_ERROR` | `taskValidationError` | Inline-edit form failed validation. |
| `GanttEvents.TASK_UPDATE_SUCCESS` | `taskUpdateSuccess` | Update completed without error. |
| `GanttEvents.TASK_UPDATE_ERROR` | `taskUpdateError` | Update threw. |
| `GanttEvents.TASK_DRAGGED` | `taskDragged` | Bar dropped at a new position. |
| `GanttEvents.TASK_RESIZED` | `taskResized` | Bar resized via a handle. |
| `GanttEvents.TASK_PROGRESS_CHANGED` | `taskProgressChanged` | In-bar progress handle dragged. |
| `GanttEvents.TASK_ADDED` | `taskAdded` | Task inserted via `addTask()` / toolbar / context menu. |
| `GanttEvents.TASK_DELETED` | `taskDeleted` | Task removed via `deleteTask()`. One event per removed task on cascade. |
| `GanttEvents.TASK_MOVED` | `taskMoved` | Task re-parented via `moveTask()`. |
| `GanttEvents.DEPENDENCY_ADDED` | `dependencyAdded` | Edge created via `addDependency()`. |
| `GanttEvents.DEPENDENCY_REMOVED` | `dependencyRemoved` | Edge removed via `removeDependency()`. |
| `GanttEvents.SELECTION_CHANGE` | `selectionChange` | Selection set changed. |
| `GanttEvents.DEPENDENCY_ARROW_UPDATE` | `dependencyArrowUpdate` | Internal arrow-redraw signal (task moved). Not a CRUD event. |
| `GanttEvents.HISTORY_CHANGE` | `historyChange` | Undo/redo stack changed (`kind`: `'record'`/`'undo'`/`'redo'`/`'clear'`). |

---

## 7. Pitfalls & Anti-Patterns

### Pitfall 1: Date format mismatch

❌ **WRONG** — passing ISO dates without setting the format:
```js
new ApexGantt(el, { series: [{ id: 't1', name: 'X', startTime: '2026-06-01', endTime: '2026-06-15' }] });
// Bars won't render — default inputDateFormat is 'MM-DD-YYYY'.
```

✅ **CORRECT**:
```js
new ApexGantt(el, {
  inputDateFormat: 'YYYY-MM-DD',
  series: [{ id: 't1', name: 'X', startTime: '2026-06-01', endTime: '2026-06-15' }]
});
```

### Pitfall 2: Missing `render()`

❌ `new ApexGantt(el, opts)` alone — nothing paints.
✅ `const g = new ApexGantt(el, opts); g.render();`

### Pitfall 3: Re-creating without destroying (React/Vue leak)

❌ **WRONG**:
```jsx
useEffect(() => {
  const g = new ApexGantt(ref.current, { series: tasks });
  g.render();
}, [tasks]);  // ResizeObservers + listeners pile up on every change
```

✅ **CORRECT**:
```jsx
useEffect(() => {
  const g = new ApexGantt(ref.current, { series: tasks });
  g.render();
  return () => g.destroy();
}, [tasks]);
```

### Pitfall 4: Using `update()` to push a single progress change

❌ Slow — re-layouts the whole timeline:
```js
gantt.update({ series: tasks.map(t => t.id === 't3' ? { ...t, progress: 75 } : t) });
```

✅ Fast — patches only the affected bar:
```js
gantt.updateTask('t3', { progress: 75 });
```

### Pitfall 5: Milestone with `endTime`

❌ **WRONG** — milestone with a duration is rendered as a regular bar:
```js
{ id: 'm1', name: 'Launch', type: 'milestone', startTime: '02-20-2026', endTime: '02-25-2026' }
```

✅ **CORRECT** — omit `endTime`:
```js
{ id: 'm1', name: 'Launch', type: 'milestone', startTime: '02-20-2026' }
```

### Pitfall 6: Listening for events on the gantt instance

❌ **WRONG** — `gantt` is not an `EventTarget`:
```js
gantt.addEventListener('taskDragged', handler);  // TypeError
```

✅ **CORRECT** — listen on the container element:
```js
container.addEventListener(GanttEvents.TASK_DRAGGED, handler);
```

### Pitfall 7: Forgetting `enableSelection`

❌ — selection APIs return empty / no-op:
```js
new ApexGantt(el, { series: tasks });
gantt.getSelectedTasks();  // []
```

✅ — opt in:
```js
new ApexGantt(el, { series: tasks, enableSelection: true });
```

### Pitfall 8: `progress` as a 0–1 fraction

❌ `{ ..., progress: 0.75 }` renders a 1% bar.
✅ `{ ..., progress: 75 }`.

### Pitfall 9: Orphan `parentId`

❌ `parentId: 'phase-x'` when no task with id `'phase-x'` exists — the row is silently flattened to top level.
✅ Make sure every `parentId` refers to a task in the same `series` array.

### Pitfall 10: Forgetting to set the license key

❌ — chart renders a watermark.
✅ — call once at app startup, before constructing any chart:
```js
import { ApexGantt } from 'apexgantt';
ApexGantt.setLicense('YOUR_LICENSE_KEY');
```

### Pitfall 11: Mutating `series` in place

❌ — direct mutation doesn't notify the chart:
```js
options.series.push(newTask);  // chart still shows old data
```

✅ — go through the API:
```js
gantt.update({ series: [...options.series, newTask] });
```

### Pitfall 12: Custom toolbar callback expects `this`

❌ Toolbar `onClick` receives `ToolbarContext`, not the gantt instance.
✅ Capture the instance in closure if you need it:
```js
const gantt = new ApexGantt(el, {
  toolbarItems: [{
    type: 'button',
    label: 'Save',
    requiresSelection: true,
    onClick: ({ selectedTasks }) => save(selectedTasks),  // use closure for gantt
  }],
  enableSelection: true,
  series: tasks,
});
```

---

## 8. Data Parsing (non-standard shapes)

Skip manual `.map()` transforms — pass `parsing` and let ApexGantt read your fields directly. Supports dot-notation and per-field `transform` functions.

```js
const apiData = [
  { task_id: 'T1', task_name: 'Design', start_date: '01-01-2026', end_date: '01-15-2026', completion: 0.75 }
];

new ApexGantt(el, {
  series: apiData,
  parsing: {
    id: 'task_id',
    name: 'task_name',
    startTime: 'start_date',
    endTime: 'end_date',
    progress: { key: 'completion', transform: (v) => v * 100 },  // 0–1 → 0–100
  },
});
```

Supported parse keys: `id`, `name`, `startTime`, `endTime`, `progress`, `type`, `parentId`, `dependency`, `barBackgroundColor`, `rowBackgroundColor`, `collapsed`.

---

## 9. Working Calendar & Sub-Day Scheduling

### `calendar` (`CalendarOptions`)

When set, weekends + holidays drive duration math, summary aggregation, and timeline stripes. Absent = every day is a working day.

```js
new ApexGantt(el, {
  series: tasks,
  calendar: {
    workingWeekdays: [1, 2, 3, 4, 5],          // 0=Sun … 6=Sat; default Mon–Fri
    holidays: ['12-25-2026', { date: '01-01-2027', label: 'New Year' }],
    showNonWorkingStripes: true,               // hatched bands over non-working columns; default true
    holidayTooltip: ({ date, label }) => `<b>${label ?? 'Holiday'}</b>`,
    dragSnapMode: 'next',                       // 'next' (default) | 'previous' | 'allow'
  },
});
```

`dragSnapMode` controls what happens when a drag/resize lands on a non-working day: `'next'`/`'previous'` snap to the adjacent working day (drag preserves working-day duration), `'allow'` permits it. `TaskDependency.lagUnit` defaults to `'working'` when a calendar is set — lag counts working days only; set `'calendar'` to force raw calendar days.

### `snapUnit` / `snapValue` (sub-day)

Granularity that drag/resize/inline edits snap to — independent of the header tier (which follows `pixelsPerDay`).

```js
new ApexGantt(el, {
  inputDateFormat: 'YYYY-MM-DD HH:mm',   // include time tokens for sub-day work
  snapUnit: 'minute',
  snapValue: 15,                         // 15-minute increments
  series: tasks,
});
```

With `'hour'`/`'minute'`, `endTime` is treated as the exclusive end timestamp and `taskDragged.daysMoved` becomes fractional (a 6-hour move reports `0.25`). `'day'` (default) treats `endTime` as inclusive.

---

## 10. Editing: CRUD, Undo/Redo & Interaction Toggles

Programmatic CRUD, all recorded in the undo history, all firing events and passing through optional veto hooks. See `references/editing.md` for full detail.

```js
gantt.addTask({ id: 't9', name: 'Review', startTime: '08-01-2026', endTime: '08-05-2026' });
gantt.addTask({ id: 'subA', name: 'Sub', startTime: '08-01-2026', endTime: '08-03-2026' }, { parentId: 't9' });
gantt.deleteTask('t9', { cascade: 'children' });   // 'forbid' (default) | 'children' | 'orphan'
gantt.moveTask('subA', { newParentId: null });     // re-parent to root
gantt.addDependency('t1', 't2', { type: 'FS', lag: 2 });
if (gantt.canAddDependency('t1', 't2').ok) { /* … */ }
gantt.removeDependency('t1', 't2');
```

**Undo/redo** — every mutating call (drag, resize, progress, edit, add/delete/move, dependency change) is recorded unless `history: { enabled: false }`. Ctrl/Cmd+Z / Ctrl+Y work inside the chart automatically.

```js
gantt.undo(); gantt.redo();
if (gantt.canUndo()) { /* enable Undo button */ }
container.addEventListener(GanttEvents.HISTORY_CHANGE, (e) => {
  const { kind, canUndo, canRedo } = e.detail;   // keep external buttons in sync
});
```

**Veto hooks** — synchronous; return `false` to cancel: `beforeTaskAdd`, `beforeTaskUpdate`, `beforeTaskMove`, `beforeTaskDelete`, `beforeDependencyChange`.

**Interaction toggles** expose the same operations to the user: `enableInlineEdit`, `enableProgressDrag`, `enableTaskEditingShortcuts`, `enableTaskCRUDToolbar`, `enableContextMenu`, `enableAddTaskRow` (see §3).

---

## 11. Reference Routing Table

For deeper detail and full working examples, refer to:

| Topic | Reference File |
|---|---|
| Task data, hierarchy, milestones, baseline, assignees, summary bars | `references/data-format.md` |
| Dependencies (`FS`/`SS`/`FF`/`SF`), lag units, critical path | `references/dependencies.md` |
| Column config, `ProgressRing` / `Wbs`, `renderers`, custom toolbar items, parsing | `references/columns-and-toolbar.md` |
| Events, selection, inline edit, drag/resize, progress drag | `references/events.md` |
| CRUD API, undo/redo, calendar, sub-day scheduling, interaction toggles | `references/editing.md` |
| React, Vue 3, Angular wrappers | `references/framework-wrappers.md` |
