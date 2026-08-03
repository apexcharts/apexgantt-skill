# Interaction — UI State, Draw-to-Create, Scroll-to-Task, Export

Opt-in interaction features from the 3.13.0 wave (shipped under v3.14.0) and
3.15.0. All are additive; existing charts are unaffected unless you enable them.

## UI state persistence

Capture and restore "where the user left off" — zoom, scroll, collapse,
selection, sort, filter, manual column widths, and column order — as a
serializable, versioned, SSR-safe snapshot.

```js
// Manual: persist to your own backend
const saved = gantt.getState();      // → GanttUiState (JSON-serializable)
gantt.setState(saved);
gantt.setState({ sort: [{ key: ColumnKey.Name, direction: 'asc' }] });  // partial
gantt.setState({ collapsed: ['phase-1'] }, { silent: true });           // no events

// Automatic: localStorage
new ApexGantt(el, { series: tasks, persistState: true });
new ApexGantt(el, { series: tasks, persistState: { key: 'project-42-gantt' } });
```

`setState()` accepts a **partial** `GanttUiState` (any omitted field is left
untouched), re-renders once, then emits `sortChange` / `filterChange` for the
parts that changed (`{ silent: true }` suppresses them). `persistState` restores
on the first `render()` and saves (debounced) on every view change. `true` uses
the default key `'apexgantt-state'`; pass `{ key }` for a custom one.

### `GanttUiState` shape

```ts
interface GanttUiState {
  version: number;                              // schema version, set by getState()
  zoom?: number;                                // pixels-per-day
  scroll?: { horizontal: number; vertical: number };
  collapsed?: string[];                         // ids of collapsed summary rows
  selected?: string[];                          // ids of selected rows (needs enableSelection)
  sort?: { key: string; direction: 'asc' | 'desc' }[];
  filterRules?: FilterRuleSet | null;
  quickFilter?: string;                         // text in the quick-filter box
  group?: { field: string; direction?: 'asc' | 'desc' } | null;
  columnWidths?: Record<string, number>;        // manual per-column px widths
  columnOrder?: string[];                       // user-chosen left-to-right order
}
```

Only serializable parts persist: a grouping's custom `accessor` / `label`
functions are dropped (restore falls back to the column's accessor).

## Draw a task on the timeline (`enableDrawTask`)

With `enableDrawTask: true`, drag across an **empty** stretch of the timeline to
sweep out a date range; on release a task is created with the range snapped to
`snapUnit` / `snapValue`. A preview bar follows the cursor, a plain click creates
nothing, and Escape cancels mid-drag. Existing bars stay draggable/resizable,
since the draw gesture only starts on empty space.

```js
new ApexGantt(el, { series: tasks, enableDrawTask: true, snapUnit: 'day' });
```

Off by default. Creation flows through the same command bus and `beforeTaskAdd`
veto hook as every other add, so it is undoable and emits `taskAdded`.

## Scroll to a task

`scrollToTask()` scrolls the timeline (and row list, if needed) so a task's bar
is in view, using nearest-edge alignment (the minimum scroll that reveals it).
It works even when the target row is virtualized out of the DOM, and returns
`false` when the task is unknown or already fully visible.

```js
gantt.scrollToTask('task-42');       // → boolean
```

Two options surface the same behavior in the UI:

- **`enableScrollButtons`** (default `false` since 3.15.0): shows a chevron at
  the edge of any row whose bar is scrolled out of the visible window; clicking
  it jumps the bar into view. `scrollToTask()` works regardless of this option.
- **`scrollToTaskOnRowClick`** (default `false`, new in 3.15.0): clicking a
  task-list row scrolls that task's bar into view. Composes with
  `enableSelection` (a single click both selects and reveals). Clicks on the
  selection checkbox, the toggle chevron, or an active inline-edit input are
  ignored.

> **Migration (3.15.0):** `enableScrollButtons` flipped from `true` to `false`.
> To keep the chevrons showing as in 3.13.0 / 3.14.x, set
> `enableScrollButtons: true` explicitly.

## Export (SVG / PNG / PDF)

The toolbar export button and `gantt.exportChart()` produce **SVG, PNG, or
PDF**. SVG is vector; PNG rasterizes the chart; PDF embeds a raster on a single
page and is dependency-free.

```js
new ApexGantt(el, { series: tasks, exportFormat: 'pdf' });  // toolbar button format
await gantt.exportChart('png');                             // programmatic, any format
```

`exportChart(format?)` returns a `Promise` that resolves once the download has
been triggered; `format` defaults to the configured `exportFormat` (`'svg'`).
Under row virtualization the full dataset is expanded for the snapshot and
restored afterward. Excel/XLSX data export remains a separate, future task.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Expecting scroll chevrons by default | `enableScrollButtons` is `false` since 3.15.0; opt in |
| Passing a full state and losing other slices | `setState()` merges — omitted fields stay untouched |
| Relying on a persisted group's `label` function | Only `field` + `direction` persist; reapply the function |
| `exportChart('png')` without awaiting | It returns a `Promise`; `await` it if you need completion |
