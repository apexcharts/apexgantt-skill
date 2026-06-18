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
  version: "1.0.0"
  library_version: "3.11.1"
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
| `width` / `height` | `number \| string` | `'100%'` / `500` | Pixel number or CSS string. |
| `rowHeight` | `number` | `28` | px |
| `tasksContainerWidth` | `number` | `425` | Initial task-list panel width in px. |
| `columnConfig` | `ColumnListItem[]` | (all) | Authoritative; only listed columns render. |
| `enableTaskDrag` | `boolean` | `true` | Reorder rows by dragging. |
| `enableTaskResize` | `boolean` | `true` | Resize bars by dragging handles. |
| `enableTaskEdit` | `boolean` | `false` | Inline edit form on row click. |
| `enableSelection` | `boolean` | `false` | Multi-row selection. Required for selection APIs. |
| `enableCriticalPath` | `boolean` | `false` | Compute & highlight CPM through dependencies. |
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
| `GanttEvents.SELECTION_CHANGE` | `selectionChange` | Selection set changed. |
| `GanttEvents.DEPENDENCY_ARROW_UPDATE` | `dependencyArrowUpdate` | Dependency edited via the arrow tool. |

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

## 9. Reference Routing Table

For deeper detail and full working examples, refer to:

| Topic | Reference File |
|---|---|
| Task data, hierarchy, milestones, baseline | `references/data-format.md` |
| Dependencies (`FS`/`SS`/`FF`/`SF`), critical path | `references/dependencies.md` |
| Column config, custom toolbar items, parsing | `references/columns-and-toolbar.md` |
| Events, selection, inline edit, drag/resize | `references/events.md` |
| React, Vue 3, Angular wrappers | `references/framework-wrappers.md` |
