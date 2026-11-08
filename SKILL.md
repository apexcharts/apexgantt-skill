---
name: apexgantt
description: >
  AI skill for building ApexGantt timeline and Gantt charts. Use whenever the
  user asks to create, configure, render, update, or troubleshoot a Gantt chart,
  task timeline, project schedule, milestone view, dependency arrows, critical
  path, or baseline-vs-actual visualization with `apexgantt`. Covers task data
  format, dependency types (`FS` / `SS` / `FF` / `SF`), date parsing,
  continuous zoom (`pixelsPerDay`), the `update()` / `updateTask()` lifecycle,
  events and veto hooks, custom toolbar items,
  selection, theming via the family `--apx-*` CSS tokens and the free Ignite UI
  add-on (`apexgantt/ignite-ui`), and framework
  integration (React / Vue / Angular). In React /
  Vue / Angular projects, prefer the framework wrapper packages
  (`react-apexgantt`, `vue-apexgantt`, `ngx-apexgantt`) over the core API.
metadata:
  author: ApexCharts
  version: "1.8.0"
  library_version: "3.21.0"
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
3. **`series` is the task array (required).** It is **NOT** optional and **NOT** the same shape as ApexCharts series. Each task is an object with `id`, `name`, `startTime`, and `endTime` (see the data format in §2). Through 3.19, a task with a `startTime` but no `endTime` throws "Task must have an id, start, and end date" unless its `type` is `'milestone'`. From 3.20.0 a task with no `endTime` and no `type` is drawn as a milestone instead (or, when it has children, as a summary bar spanning them); an explicit `type: 'task'` with no `endTime` still throws. The other exceptions: a parent row (one that other tasks name as `parentId`) may omit both dates and derives its span from its children (from 3.20.0 it may also give just one of them; through 3.19 one date without the other throws); a task with `segments` takes its dates from them. A task with no `startTime` and no children throws on every version; from 3.20.0 the message is `Task "<id>" has no startTime and no children. Provide dates or add child tasks.`, and it also covers a leaf with only an `endTime` and a `type: 'milestone'` with no start.
4. **Date strings should match `inputDateFormat`.** The default is `'MM-DD-YYYY'`. ISO dates such as `'2026-06-01'` still draw under the default, because parsing falls back to a lenient read, but edited or dragged dates are written back as `MM-DD-YYYY`, mixing formats in your data. Through 3.19 the edit dialog also parses strictly, so its date fields come up empty (from 3.20.0 the dialog reads ISO and other formats too). For ISO data set `inputDateFormat: 'YYYY-MM-DD'`.
5. **`id` must be unique and stable across renders.** Selection, dependency arrows, and the diff-update path all key off `id`. Re-using `id`s breaks dependency drawing and selection state. Ids are typed `string`; `parentId` and dependency references are matched by value, so a task with `id: 1` is still found by `parentId: '1'`, but use strings.
6. **`parentId` references must point to an existing task `id`.** Through 3.19, a task whose `parentId` matches no task is not rendered at all: it disappears from both the task list and the timeline. From 3.20.0 it is attached to the root with a console warning. Either way the nesting you meant is lost. A `parentId` that makes a cycle (a task naming itself, or a under b under a) overflows the stack through 3.19; from 3.20.0 it logs `[ApexGantt] Task "<id>" has parentId "<p>", which makes a cycle of parents. It is shown at the root.` and the task is shown at the root.
7. **Milestones need only `startTime`.** A task with `type: TaskType.Milestone` (the exact lowercase string `'milestone'`) renders as a diamond at `startTime`. An `endTime` on a milestone is ignored for drawing but still widens the timeline and is read out as a range by screen readers, so omit it. `'Milestone'` with a capital M is not recognised and the row is treated as an ordinary task. From 3.20.0 a task with no `endTime`, no `type` and no children is also drawn as a milestone, but set `type: 'milestone'` explicitly: it is the clearest and works on every version.
8. **`progress` is a percentage (0 to 100), not a 0 to 1 fraction.** It is not clamped: `150` draws a 150% fill.
9. **Each task takes one dependency in its data**: a plain task `id` string (treated as `FS` with `lag: 0`) or a `TaskDependency` object `{ taskId, type, lag }`. An array is not accepted and draws no arrow. For a task with several predecessors, put one in the data and add the rest after `render()` with `gantt.addDependency(fromId, toId, { type, lag })`. Links added that way or drawn by the user are not in `getTasks()`: save them from `gantt.getDependencies()` (3.20.0+) and add them back after loading (see §5).
10. **Use `update()` for full reconfigures**, `updateTask(id, patch)` for surgical single-task changes. `updateTask` skips a full re-layout and is much faster for streaming progress updates. Through 3.19, `update()` with any option rebuilds from the `series` you last passed, so edits made by dragging or through the CRUD methods are lost; from 3.20.0 they survive any `update()` that does not pass `series` or `parsing` (see §4).
11. **For non-standard data shapes use `parsing`, not manual `.map()`.** The `parsing` config maps your fields to `TaskInput` fields and supports dot-notation paths and `transform` functions.
12. **`enableSelection: true` is required** for `getSelectedTasks()` / `setSelectedTasks()` / `clearSelection()` and the `selectionChange` event to fire. Without it they do nothing (from 3.20.0 they also log a one-time console warning).
13. **Custom toolbar items live in `toolbarItems`**, not in a separate render call. Use `position: 'left'` to put them before the built-in zoom/export controls.
14. **Events fire on the host container element**, not on the `gantt` instance. Listen via `container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, ...)`.
15. **Links never move tasks.** Dependencies are drawn and analysed (critical path, Predecessors / Successors columns), but there is no auto-scheduling: moving a predecessor leaves its successors where they are. `enableTaskDrag` reschedules a bar horizontally; it does not reorder rows or draw links (drawing links is `dependencies.editable`).
16. **Veto hooks are synchronous** and take one context object, e.g. `beforeTaskUpdate: ({ task, updates }) => boolean`. Return `false` to cancel; a returned `Promise` is not a veto.

---

## 2. Task Data Format (`series`)

`series` is an array of `TaskInput` objects:

```ts
{
  id: string;                       // REQUIRED, unique, stable
  name: string;                     // REQUIRED, display label
  startTime: string;                // REQUIRED, parsed by inputDateFormat (a parent row may omit both dates; optional in the 3.20.0 type)
  endTime?: string;                 // required with startTime, except on milestones and (3.20.0+) parents
  progress?: number;                // percentage 0 to 100, default 0, not clamped
  type?: 'task' | 'milestone';      // default 'task' (3.20.0+: 'milestone' when endTime is missing and there are no children); exact lowercase
  parentId?: string;                // hierarchical nesting; matched by value; a cycle is shown at the root (3.20.0+)
  dependency?: string | TaskDependency;  // ONE predecessor; add more with gantt.addDependency()
  baseline?: { start: string; end: string };
  barBackgroundColor?: string;      // per-task override
  barTextColor?: string;            // 3.20.0+: per-task colour of the label drawn inside the bar
  rowBackgroundColor?: string;      // per-row override
  collapsed?: boolean;              // hide children
  showSummaryBar?: boolean;         // parents get a summary bar by default; false draws an ordinary bar
  assignees?: { name: string; avatarUrl?: string; initials?: string; color?: string }[];
  segments?: { start: string; end: string }[];  // split task into worked spans (two or more; 3.20.0+ stores a single one as plain dates)
}
```

Rows are **sorted by start date by default**, so the display order is not the array order. Pass `sortBy: []` to keep the order of `series`.

> **Split tasks:** pass `segments` (each `{ start, end }`) to render one task as several worked spans with gaps on a single row. `startTime`/`endTime` become the derived envelope. Split at runtime with `gantt.splitTask(id, at, { resumeAt })`. See `references/data-format.md`.

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

`lag` is in days: working days when a calendar is active, calendar days otherwise. It feeds the critical path and the arrow tooltip only. Links never move tasks. `TaskDependency.lagUnit` has no effect (deprecated in 3.20.0).

### Hierarchy

Use `parentId` to nest. Parent rows get a summary bar by default: it spans the children's date range (the parent's own dates are ignored), is read-only (no drag, resize or progress), and the row renders an expand/collapse caret. Set `collapsed: true` on the parent to start collapsed, or `showSummaryBar: false` to draw the parent as an ordinary bar from its own dates. From 3.20.0 a parent may give only a `startTime` or only an `endTime` and still gets the summary bar (through 3.19 one date without the other throws). A start-only parent is typed `'task'`; the same row without children is a milestone.

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
| `theme` | `'light' \| 'dark' \| string` | `'light'` | Built-in palette, or (3.21.0+) the name of a theme registered on the family registry (see Named themes below). See also the family `--apx-*` tokens below. An `update()` that switches the theme resets the theme's own colour options to the new theme's values (see §4); from 3.20.0 every other option carries over. From 3.21.0, left unset (with no `backgroundColor`), it follows a design system that declares a dark variant (Ignite UI, with its add-on). |
| `tokens` | `boolean` | `true` | (3.21.0) `false` stops the chart reading the `--apx-*` family tokens and the page's design system (its variant included). |
| `designSystem` | `boolean` | `true` | (3.21.0) `false` stops the chart reading the page's design system (Ignite UI's `--ig-*` tokens and variant); `--apx-*` still apply. Matters only while the `apexgantt/ignite-ui` add-on is loaded. |
| `pixelsPerDay` | `number` | auto-fit | Initial zoom as pixels-per-day (continuous, clamped to about `0.25` to `7680`; header tiers are chosen automatically, there are no view modes). Reference values: Year ≈ `0.5`, Quarter ≈ `1.6`, Month ≈ `4.9`, Week ≈ `25.7`, Day = `80`. Omit to auto-fit the data range. |
| `inputDateFormat` | `string` | `'MM-DD-YYYY'` | dayjs format for `startTime`/`endTime`. |
| `snapUnit` | `'day' \| 'hour' \| 'minute'` | `'day'` | Granularity that drag / resize / inline edits snap to. `'hour'`/`'minute'` enable sub-day scheduling. From 3.20.0 `'day'` moves by calendar days, so a step across a daylight-saving change lands on the next date. See §9. |
| `snapValue` | `number` | `1` | Multiplier on `snapUnit` (e.g. `'minute'` + `15` gives 15-min steps). |
| `calendar` | `CalendarOptions` | none | Working-calendar (weekends, holidays, non-working stripes, drag snap). From 3.20.0 holiday strings are read with `inputDateFormat` (ISO always works) and `workingWeekdays` accepts numeric strings. See §9. |
| `history` | `{ enabled, maxSize }` | `{ enabled: true, maxSize: 100 }` | Undo/redo stack config. See §10. |
| `width` / `height` | `number \| string` | `'100%'` / `'500px'` | Pixel number or CSS string. From 3.20.0, when `height` is omitted and the container already has a height of its own, the chart fills the container and follows its size, and with more than 50 rows (virtualised) the rows reach the bottom of the container on first paint. Without a container height, a full re-render (collapse, sort, filter, `update()`, add, delete) no longer resets the page scroll from 3.20.0. |
| `rowHeight` | `number` | `40` | px |
| `tasksContainerWidth` | `number` | `425` | Initial task-list panel width in px. |
| `columnConfig` | `ColumnListItem[]` | Task Name, Start, Duration, Progress | Authoritative; only listed columns render. End and the other built-in columns are opt-in. |
| `autoSizeColumns` | `boolean` | `true` | Auto-size each column to fit header + content, growing the panel (never below `tasksContainerWidth`). `false` = legacy split-by-`flexGrow`. See `references/grid.md`. |
| `resizableColumns` | `boolean` | `true` | Drag a column-header handle to pin a column to an exact px width. |
| `reorderableColumns` | `boolean` | `true` | Drag a column header to reorder columns. |
| `sortBy` | `SortCriterion \| SortCriterion[]` | start-time asc | Initial sort (single or multi-key), hierarchy-preserving. Rows are sorted by start date unless you pass `[]` for natural (input) order. From 3.20.0 empty values sort last in both directions. |
| `filterBy` | `(task) => boolean` | — | Initial predicate filter (keeps matches + their ancestors). |
| `filterRules` | `FilterRuleSet` | — | Initial structured filter `{ match: 'all'\|'any', rules }`. Takes precedence over `filterBy`. |
| `enableQuickFilter` | `boolean` | `false` | Toolbar quick-filter search box. |
| `quickFilter` | `QuickFilterOptions` | — | Tune the quick filter: `placeholder`, `fields` (default `['name']`), `caseSensitive`. |
| `enableFilterBuilder` | `boolean` | `false` | Toolbar advanced filter builder (All/Any popover). |
| `groupBy` | `GroupCriterion \| ColumnKey \| string` | — | Initial grouping by a column value. Suspends the tree into collapsible group headers. See `references/grid.md`. |
| `enableTaskDrag` | `boolean` | `true` | Drag a bar horizontally to reschedule it. Does not reorder rows or draw links. From 3.20.0, ArrowLeft / ArrowRight on a focused bar moves it one snap unit. |
| `enableTaskResize` | `boolean` | `true` | Resize bars by dragging handles. From 3.20.0, Shift+ArrowLeft / Shift+ArrowRight on a focused bar resizes its end (not milestones). |
| `enableTaskEdit` | `boolean` | `false` | Edit dialog on bar double-click (or Enter on a focused row). Also turns on `enableInlineEdit` unless that is set to `false`. |
| `enableInlineEdit` | `boolean` | `false` | Edit `name`/`startTime`/`endTime`/`duration`/`progress` directly in list cells. Auto-on when `enableTaskEdit` is `true` unless set `false`. |
| `enableProgressDrag` | `boolean` | `true` | Drag the in-bar handle to set progress. Fires `taskProgressChanged`. |
| `enableSelection` | `boolean` | `false` | Multi-row selection. Required for selection APIs (3.20.0+ warns once when they are used without it). |
| `enableTaskEditingShortcuts` | `boolean` | `false` | `Delete`/`Backspace` delete focused row; `Tab`/`Shift+Tab` indent/outdent. Needs list focus. |
| `enableTaskCRUDToolbar` | `boolean` | `false` | Built-in `+ Add Task` / trash `Delete` toolbar buttons. Delete needs `enableSelection`. |
| `enableContextMenu` | `boolean` | `false` | Right-click menu: Edit, Add child/sibling, Indent, Outdent, Delete (capability-gated). |
| `enableAddTaskRow` | `boolean` | `false` | `+ Add task` row at the bottom of the list. Hidden while more than 50 rows are visible (row virtualisation is on). |
| `enableDrawTask` | `boolean` | `false` | Draw a new task by dragging across empty timeline space (snaps to `snapUnit`/`snapValue`). See `references/interaction.md`. |
| `enableScrollButtons` | `boolean` | `false` | Edge chevrons that jump to an off-screen bar. (Default flipped to `false` in 3.15.0.) |
| `scrollToTaskOnRowClick` | `boolean` | `false` | Clicking a task-list row scrolls its bar into view (nearest-edge). Composes with `enableSelection`. (New in 3.15.0.) |
| `persistState` | `boolean \| { key }` | `false` | Persist/restore UI view state via `localStorage` (`true` = default key `'apexgantt-state'`). See `references/interaction.md`. |
| `exportFormat` | `'svg' \| 'png' \| 'pdf'` | `'svg'` | Format for the toolbar export button. Any format is also available via `gantt.exportChart(format)`. |
| `enableCriticalPath` | `boolean` | `false` | Compute & highlight CPM through dependencies. |
| `dependencies` | `DependencyOptions` | unset | Arrow options: `editable` (`false`; `true` lets users draw links between bar edges and delete them with ✕ or Delete), `allowSummaryDescendantLinks` (from 3.20.0 it also governs `addDependency()`, whether or not `editable` is set; links in the data are not checked), `hitWidth`, `cornerRadius`, `classBuilder`, `tooltipTemplate`. See `references/dependencies.md`. |
| `enableRollups` | `boolean` | `false` | Thin rollup markers under summary bars at each leaf's range (visible even when collapsed). |
| `enableProjectBoundary` | `boolean` | `false` | Two vertical lines at the project's earliest start / latest end. From 3.20.0 they move with drags, resizes, `updateTask` and undo, and carry the class `annotation--project-boundary`; through 3.19 they are recomputed only on a full render. |
| `projectBoundaryColor` | `string` | `'#7C3AED'` | Stroke for the boundary lines. Falls back to `annotationBorderColor`. |
| `enableCrosshair` | `boolean` | `false` | Vertical cursor-following line + date/time label. |
| `crosshairColor` | `string` | theme accent | Crosshair line and label-background color. |
| `crosshairLabelFormat` | `(date, tier) => string` | auto | Custom crosshair label formatter; `tier` is the active header tier. |
| `barLabel` | `BarLabelOptions` | `{ position: 'right' }` | Per-task bar label: `position` `'right'\|'inside'\|'left'\|'auto'`, `field`, `render`, `className`, `leadingPadding`. |
| `columnLines` | `boolean` | `true` | Draw vertical lines between timeline columns. `false` keeps only row dividers. |
| `baseline` | `{ enabled, color }` | disabled | See §2. |
| `parsing` | `ParsingConfig` | none | Field-mapping to avoid manual transforms. See §8. |
| `toolbarItems` | `ToolbarItem[]` | `[]` | Custom toolbar buttons / selects / separators. A select's optional `value` (3.20.0+) sets its initial value, and from 3.20.0 a select keeps its chosen value across re-renders. |
| `tooltipTemplate` | `(task, fmt, context) => string` | built-in | HTML string returned per task. From 3.20.0 a third argument `context` (`TooltipTemplateContext`) carries `durationText`, the task's duration exactly as the Duration column shows it (`'45 m'`, `'5 wd'`); two-argument templates still work, and the default tooltip's Duration line matches the column. |
| `annotations` | `Annotation[]` | `[]` | Line (`x1` only) or range (`x1` to `x2`) markers that run down the task rows; `label.orientation` turns only the label. Per-annotation `borderDashArray` (`number \| number[]`) and `label.orientation`, falling back to the global `annotationBorderDashArray` / `annotationOrientation`, are honoured from 3.20.0. From 3.20.0 a dashed line leaves its gaps empty so the timeline shows through (range annotations keep their fill). |

### Family theme tokens (`--apx-*`) (3.18.0+)

ApexGantt reads the shared ApexCharts family CSS tokens directly from the cascade, so a page can declare its brand once on `:root` and have the Gantt follow alongside every other Apex product on the page:

| Token | Role | Gantt options it feeds |
|---|---|---|
| `--apx-accent` | The color that means interactive or selected | `barBackgroundColor` |
| `--apx-fore` | Text and anything that must stay legible on the surface | `fontColor` |
| `--apx-grid` | Hairlines: borders, gridlines, connectors | `cellBorderColor`, `arrowColor` |
| `--apx-surface` | The plane content sits on | `backgroundColor`, `headerBackground`, `rowBackgroundColors` |

ApexGantt does not read `--apx-series-N`. Bars and arrows follow `--apx-accent` / `--apx-grid` (and the product variables `--apex-gantt-bar-fill`, which falls back to `--apx-accent`, and `--apex-gantt-dependency-line`) from 3.20.0; through 3.19 those resolved colours reach only the toolbar buttons and hover states, and bars and arrows paint from the options. Per-task colours (`barBackgroundColor` on a task) always win. Token values are read at render, not live: call `update()` after changing them (mind the 3.19 `update()` caveat in §4).

`--apex-gantt-font-family` and `--apex-gantt-font-size` on the container override `fontFamily` / `fontSize`. From 3.20.0 they also reach bar and holiday tooltips, dependency-arrow tooltips and annotation labels that set no font of their own; through 3.19 those parts use the options even when the variables are set.

```css
:root {
  --apx-accent: #5b21b6;
  --apx-fore: #101828;
  --apx-grid: #e4e7ec;
  --apx-surface: #ffffff;
}

@media (prefers-color-scheme: dark) {
  :root {
    --apx-fore: #f8fafc;
    --apx-grid: #334155;
    --apx-surface: #0f172a;
  }
}
```

Resolution order (highest wins), from 3.21.0:

```
option you passed  >  --apex-gantt-* variable  >  --apx-* token  >  registered theme  >  design system  >  theme default
```

An option counts as passed when it is in the options you gave the constructor or any `update()`, even at its default value; passing it as `undefined` in an `update()` takes it back, and so does an `update()` that switches `theme` for the colours the theme owns. Options you did not pass rank last, so a variable or a token themes them. "Design system" is Ignite UI, read only with its add-on (below). Values resolve when the chart renders, not live: after changing a variable or token, call `gantt.update({})` or `gantt.render()`.

Through 3.20 the order was `product CSS variable > explicit option > --apx-* token > built-in default`, and an option equal to its built-in default lost to the token.

**More tokens (3.21.0).** `--apx-font-family` sets `fontFamily`. `--apx-radius-factor` multiplies Gantt's own built-in corner radii (menus, dialogs, the task form, tooltips, dropdowns, scrollbars, milestones, roll-ups); unset it is `1`. `barBorderRadius` is not scaled.

**Named themes (3.21.0).** `theme` accepts the name of a theme on the registry the whole Apex family shares (for example one registered with `ApexCharts.registerTheme(name, { mode, tokens })` when ApexCharts is on the page). Its tokens fill the same slots as the `--apx-*` tokens, one rank below them. A theme registered with `mode: 'dark'` gives every other colour the dark theme's value; without one it is light. A name nothing is registered under renders as `'light'`; a theme registered under `'light'` or `'dark'` adds its tokens to that built-in theme. TypeScript: the option type is `GanttThemeName` (`ThemeMode | (string & {})`, exported from 3.21.0). Through 3.20 `theme` took only `'light' | 'dark'`.

### Ignite UI theming (from apexgantt 3.21.0)

An opt-in add-on, **free, no licence key**. Load it once and Gantt charts follow the Ignite UI theme on the page: bars, text, gridlines, the header, the tooltip, the font, light or dark.

```js
import 'apexgantt/ignite-ui';
```

```html
<!-- without a bundler: before or after apexgantt.min.js -->
<script src="https://cdn.jsdelivr.net/npm/apexgantt@3.21.0/apexgantt.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/apexgantt@3.21.0/apexgantt.ignite-ui.min.js"></script>
```

- **Load order.** Before the first render; a chart rendered earlier picks it up on its next `render()` or `update()`. Without the add-on the `--ig-*` tokens are ignored, with no error; `--apx-*` work either way.
- **What it maps.** `fontColor` `--ig-gray-800`, `cellBorderColor` and `arrowColor` `--ig-gray-300`, `barBackgroundColor` `--ig-primary-seed`, `fontFamily` `--ig-font-family`, and (Ignite UI 7.5.0+) `headerBackground` `--ig-surface-raised` and `tooltipBGColor` `--ig-surface-overlay`. `backgroundColor` and the row colours keep the theme's own.
- **Ranking.** The lowest layer above the theme defaults: an option you passed, an `--apex-gantt-*` variable and an `--apx-*` token all win.
- **Mode.** When the sheet declares a dark variant and you passed neither `theme` nor `backgroundColor`, the chart takes the dark theme's colours, and goes back to light when the variant changes. Passing `theme: 'light'` pins light and blocks the dark variant.
- **Per chart.** `designSystem: false` turns it off; `tokens: false` turns off `--apx-*` and the design system together.
- **Stylesheet swap.** ApexGantt has no `refreshTokens()`: call `gantt.render()` after switching the Ignite UI sheet.
- **Other products.** The add-on also themes ApexTree and ApexSankey on the page (with several, load it from the newest package). ApexCharts charts take `apexcharts/features/ignite-ui`, which themes Gantt charts too, but on a script-tag page only once `apexcharts.js` has loaded, so load `apexgantt.ignite-ui.min.js` there for the Gantt charts.

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

gantt.destroy();                             // remove the chart's nodes, free observers
```

`update()` preserves scroll position, collapsed/expanded state, and current zoom (`pixelsPerDay`) unless you explicitly override them.

What `update()` does with task data depends on the version:

- **Through 3.19**, any `update()` rebuilds the chart from the `series` you last passed in options. Edits made since then by dragging, `updateTask`, `addTask`, `deleteTask`, `moveTask` or dependency changes are thrown away, and the undo history goes stale. Pass the current data back in `series` if you must call `update()` after edits.
- **From 3.20.0**, `update()` keeps the current task data: those edits survive any `update()` that does not pass `series` or `parsing`. Passing `series` replaces the data and clears the undo history. Passing `calendar` or `inputDateFormat` rebuilds the working calendar, and a theme change keeps `options.calendar`. The parameter type is `Partial<GanttUserOptions>` (`GanttUpdateOptions`), so `series` is no longer required. An `update({ series })` whose data throws (a task with no dates and no children, say) leaves the previous data and options in place; so does a throwing `addTask` or `updateTask`. Through 3.19 the chart is left half-built.

**Switching `theme` with `update()`.** A call that switches the theme returns these 18 colour options to the new theme's defaults, even ones you set yourself, unless the same call passes them: `annotationBgColor`, `annotationBorderColor`, `arrowColor`, `backgroundColor`, `barBackgroundColor`, `barTextColor`, `borderColor`, `cellBorderColor`, `criticalArrowColor`, `criticalBarColor`, `crosshairColor`, `fontColor`, `headerBackground`, `milestoneColor`, `rowBackgroundColors`, `summaryBarColor`, `tooltipBGColor`, `tooltipBorderColor`. From 3.20.0 every other option (columns, toolbar items, hooks, locale, `sortBy`, `barLabel`, `dependencies`, `calendar`, ...) carries over; through 3.19 only a fixed list of 19 keys does. To keep a brand colour, pass it again: `gantt.update({ theme: 'dark', barBackgroundColor: brand })`.

`render()`, re-renders and `destroy()` no longer empty the container (3.19.0+): the children it held when the chart was constructed stay, and everything else is removed as the chart's own. Put your own markup in the container before constructing, not after. Constructing a new `ApexGantt` on a container that already holds a live chart destroys the old one first.

---

## 5. Public API

### Instance methods

| Method | Description |
|---|---|
| `render(_data?)` | Paints the chart. Returns the container `HTMLElement`. Call once after construction. |
| `update(options)` | Merge new options & re-render. Unspecified keys keep current values. Through 3.19 it also resets task data to the last `series` passed; from 3.20.0 edits survive unless you pass `series` or `parsing` (see §4). |
| `updateTask(id, patch, { silent? })` | Surgical single-task update; `patch` is `Partial<TaskInput>`. Throws if `id` not found. Runs `beforeTaskUpdate`. Through 3.19 it fires no events. From 3.20.0 it fires `taskUpdate` / `taskUpdateSuccess` with `source: 'api'`, and `{ silent: true }` suppresses both (use it for rollbacks and server echoes). From 3.20.0 a `collapsed` patch re-renders so rows appear or disappear at once, and a change to a task whose row is not rendered (collapsed, filtered out, outside the virtual window) is stored like any other; through 3.19 such a change is lost. From 3.20.0 an edit keeps the task's tree level, a `segments` patch with one entry turns a split task back into a plain one, and a throwing update leaves the previous data in place. |
| `getTask(id)` | **3.20.0+.** Copy of the stored task with current values (edits included), or `null` for an unknown id. `baseline`, `segments`, `assignees` and an object `dependency` are copied too, so changing the result never changes the chart; custom fields you added are copied by reference. |
| `getTasks()` | **3.20.0+.** Copies (as `getTask`) of every task, including collapsed and filtered-out ones, in tree order (parent before its children). Not a complete save: each task carries only the `dependency` it was loaded with, never links added with `addDependency()` or drawn by the user. Save `getDependencies()` with it. |
| `getDependencies()` | **3.20.0+.** Every link as it is now, as `DependencyLink` copies `{ fromId, toId, type, lag }`: the ones loaded with the tasks plus those added with `addDependency()` or drawn, minus removed ones. Undo and redo are reflected. |
| `expandTask(id)` / `collapseTask(id)` | **3.20.0+.** Open / close a parent. View state only: not recorded in undo history, persisted with the view state like `setState({ collapsed })`, one render. Returns `false` for an unknown id or a task with no children. |
| `addTask(input, { parentId? })` | Insert a task (recorded in history). Returns the `Task`, or `null` if `beforeTaskAdd` vetoed. Throws on duplicate `id`. The task lands under the `parentId` option, else `input.parentId`; from 3.20.0 `beforeTaskAdd` and `taskAdded` report that parent (through 3.19 they ignore `input.parentId`), and `undefined` for an id that matches no task. |
| `deleteTask(id, { cascade? })` | Remove a task. `cascade`: `'forbid'` (default, throws if it has children), `'children'`, `'orphan'`. Returns `false` if `beforeTaskDelete` vetoed. |
| `moveTask(id, { newParentId? })` | Re-parent (`null`/omit → root). Returns `false` if `beforeTaskMove` vetoed. Throws on cycle / missing id. |
| `addDependency(fromId, toId, { type?, lag? })` | Create an edge. Returns `false` if `beforeDependencyChange` vetoed. Throws on missing task / duplicate; from 3.20.0 also on a self-link, a link that would close a cycle and (unless `dependencies.allowSummaryDescendantLinks`) a link between a task and its own ancestor or descendant, the rules `canAddDependency()` reports. Check with `canAddDependency()` first. |
| `removeDependency(fromId, toId)` | Remove an edge. Returns `false` if vetoed. Throws when no edge exists. |
| `canAddDependency(fromId, toId, opts?)` | Dry-run check. Returns `{ ok: true }` or `{ ok: false, reason }` (`'self'`/`'task-missing'`/`'duplicate'`/`'cycle'`/`'summary-descendant'`/`'hook-veto'`). |
| `undo()` / `redo()` | Traverse the history stack. No-op (returns `false`) when the stack is empty or `history.enabled` is `false`. |
| `canUndo()` / `canRedo()` | Whether a transaction is available to undo / redo. Gate toolbar buttons off these. |
| `clearHistory()` | Empty both stacks. Fires `historyChange` with `kind: 'clear'`. |
| `getHistorySize()` | `{ undo, redo }` stack sizes. |
| `zoomIn()` / `zoomOut()` | Multiply / divide `pixelsPerDay` by 1.4 per step (continuous zoom, clamped to about `0.25` to `7680`). There are no view-mode presets; the header tiers follow the zoom. |
| `getSelectedTasks()` | Array of selected `Task` objects (`Task[]`). Requires `enableSelection: true`. |
| `setSelectedTasks(ids)` | Replace selection by id array. |
| `clearSelection()` | Clear selection state. |
| `sort(criteria)` | Apply a sort (single or multi `SortCriterion`; `[]` clears). Hierarchy-preserving. Emits `sortChange`. |
| `clearSort()` / `getSort()` | Clear the sort (natural order) / read the active `SortCriterion[]`. |
| `toggleSort(key, { append? })` | Header-click cycle asc → desc → none. `append: true` (Shift+click) adds a secondary key. |
| `filter(predicate)` | Apply a view-only predicate filter (keeps matches + their ancestors). Emits `filterChange`. |
| `setFilterRules(ruleSet)` | Apply a structured `FilterRuleSet` (pass `null`/empty to clear). Emits `filterChange`. |
| `clearFilter()` / `isFiltered()` / `getFilterRules()` | Clear the filter / query filter state / read the active rules (or `null`). |
| `groupBy(criterion)` | Group by a `ColumnKey`/string or `GroupCriterion`. Suspends the tree. Emits `groupChange`. |
| `clearGrouping()` / `getGroupBy()` / `isGrouping()` | Restore the tree / read the active `GroupCriterion` (or `null`) / query grouping state. |
| `setColumnWidth(key, px)` | Pin a column to an exact pixel width. Emits `columnResize`. |
| `resetColumnWidths(key?)` / `getColumnWidths()` | Clear one (or all) manual widths / read them as `{ key: px }`. |
| `setColumnOrder(keys)` / `getColumnOrder()` | Set / read the left-to-right column order. Emits `columnReorder`. |
| `splitTask(id, at, { resumeAt? })` | Split a task into worked segments at `at` (routes through `updateTask`; undoable). Keeps the task's finish date; to show a slip, pass `segments` through `updateTask`. No-op on milestones/summaries. From 3.20.0 `resumeAt` must fall after `at` and no later than the segment's last day (day-only formats, so a one-day remainder is fine) or before its end (formats with time); anything else is rejected with a console warning. The default `resumeAt` (the day after `at`) leaves no visible gap with day-only formats. |
| `isSplit(id)` | Whether a task currently has multiple segments. |
| `getState()` | Capture the UI view state as a serializable `GanttUiState`. |
| `setState(state, { silent? })` | Restore a (partial) `GanttUiState`. Re-renders once; emits `sortChange`/`filterChange` unless `silent`. |
| `scrollToTask(id, { expand? })` | Scroll a task's bar into view (nearest-edge). Returns `false` if unknown / already visible. Through 3.19 it also returns `false` for a task under a collapsed parent. From 3.20.0 `expand` (default `true`) opens collapsed ancestors and groups first; it returns `false` for an unknown id, a filtered-out task, or `expand: false` with a collapsed ancestor. |
| `exportChart(format?)` | Export as `'svg'`/`'png'`/`'pdf'` and trigger a download. Defaults to `exportFormat`. PNG and PDF are images. Returns a `Promise`. |
| `renderToolbar(container)` | Render the built-in toolbar into a custom DOM slot. |
| `destroy()` | Tear down. Required before unmounting in SPA frameworks. Removes the chart's nodes; children the container held before construction stay. From 3.20.0 a licence change after `destroy()` never paints the trial mark (or sets `position: relative`) on a container the page has reused. |
| `isDestroyed()` | Guard before calling other methods. |

### Static method

| Method | Description |
|---|---|
| `ApexGantt.setLicense(key)` | Set your license key once at application start. ApexGantt is included in the Premium plan and above; there is no free tier. Without a key the chart renders with a trial watermark. From 3.20.0 only a Premium (or higher) key removes the Gantt watermark, and a valid key below Premium logs a one-time console warning saying why the mark stays. A late call (after charts render) still removes it (3.17.0+), but calling it first avoids a one-frame watermark. The key is shared by every Apex product on the page: from 3.20.0 a later valid key whose plan ranks lower than the one in force (a Pro key set by another product after your Premium key) is ignored with a warning, so it cannot re-watermark the Gantt; call `setLicense('')` first to move to a lower plan on purpose. |

---

## 6. Events

Events are dispatched as `CustomEvent`s on the **container element**, not on the chart instance. Use the `GanttEvents` constants (or the string event names) to subscribe.

```js
import { ApexGantt, GanttEvents } from 'apexgantt';

const container = document.getElementById('chart');
const gantt = new ApexGantt(container, { series: tasks });
gantt.render();

container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, (e) => {
  const { taskId, updatedTask, source, timestamp } = e.detail;   // source: 3.20.0+
  // persist to backend, update local state, etc.
});
container.addEventListener(GanttEvents.TASK_DRAGGED, (e) => {
  const { taskId, newStartTime, newEndTime, daysMoved, affectedChildTasks } = e.detail;
});
```

| Constant | Event name | When |
|---|---|---|
| `GanttEvents.TASK_UPDATE` | `taskUpdate` | An edit is being committed. Through 3.19: only from the edit dialog and inline cell edits. From 3.20.0: once per committed edit on every path, with `detail.source`. |
| `GanttEvents.TASK_VALIDATION_ERROR` | `taskValidationError` | The edit dialog failed validation; nothing is applied. |
| `GanttEvents.TASK_UPDATE_SUCCESS` | `taskUpdateSuccess` | The edit was applied. Same coverage as `taskUpdate`; carries `detail.source` from 3.20.0. |
| `GanttEvents.TASK_UPDATE_ERROR` | `taskUpdateError` | Applying a dialog or inline edit threw. From 3.20.0 it fires on every edit path; `updateTask()` still throws afterwards. |
| `GanttEvents.TASK_DRAGGED` | `taskDragged` | Bar dropped at a new position. From 3.20.0 only after the commit is accepted (not when `beforeTaskUpdate` vetoes). |
| `GanttEvents.TASK_RESIZED` | `taskResized` | Bar resized via a handle. From 3.20.0 only after the commit is accepted. |
| `GanttEvents.TASK_PROGRESS_CHANGED` | `taskProgressChanged` | In-bar progress handle dragged. From 3.20.0 only after the commit is accepted. |
| `GanttEvents.TASK_ADDED` | `taskAdded` | Task inserted via `addTask()` / toolbar / context menu. `detail.parentId` is the parent it landed under (3.20.0+ also when given as `input.parentId`). |
| `GanttEvents.TASK_DELETED` | `taskDeleted` | Task removed via `deleteTask()`. One event per removed task on cascade. |
| `GanttEvents.TASK_MOVED` | `taskMoved` | Task re-parented via `moveTask()`. |
| `GanttEvents.DEPENDENCY_ADDED` | `dependencyAdded` | Edge created via `addDependency()` or drawn by the user (`dependencies.editable`). Not fired by undo/redo. |
| `GanttEvents.DEPENDENCY_REMOVED` | `dependencyRemoved` | Edge removed via `removeDependency()`, the arrow's ✕ or Delete key, or `deleteTask()`. Not fired by undo/redo. |
| `GanttEvents.SELECTION_CHANGE` | `selectionChange` | Selection set changed. |
| `GanttEvents.DEPENDENCY_ARROW_UPDATE` | `dependencyArrowUpdate` | Internal arrow-redraw signal (task moved). Not a CRUD event. |
| `GanttEvents.HISTORY_CHANGE` | `historyChange` | Undo/redo stack changed (`kind`: `'record'`/`'undo'`/`'redo'`/`'clear'`). |
| `GanttEvents.SORT_CHANGE` | `sortChange` | Sort changed. `detail`: `{ criteria: { key, direction }[], timestamp }`. |
| `GanttEvents.FILTER_CHANGE` | `filterChange` | Filter changed. `detail`: `{ active, visibleCount, timestamp }`. |
| `GanttEvents.GROUP_CHANGE` | `groupChange` | Grouping changed. `detail`: `{ active, field, groupCount, timestamp }`. |
| `GanttEvents.COLUMN_RESIZE` | `columnResize` | Column width changed. `detail`: `{ key, width, widths, timestamp }` (`width` is `null` on reset). |
| `GanttEvents.COLUMN_REORDER` | `columnReorder` | Column order changed. `detail`: `{ order, movedKey, timestamp }` (`movedKey` is `null` for bulk `setColumnOrder`). |

**`source` (3.20.0+).** `taskUpdate` and `taskUpdateSuccess` details carry `source: TaskUpdateSource`, one of `'api' | 'dialog' | 'drag' | 'inline' | 'keyboard' | 'progress' | 'resize'`. They do not fire for undo/redo, or for `updateTask(..., { silent: true })`. From 3.20.0 one `taskUpdateSuccess` listener sees every edit, so an app that saves on both `taskDragged` and `taskUpdateSuccess` saves twice: pick one. `detail.updatedTask` is the stored task after the rebuild, so its `level`, `wbs` and summary values are current, and an edit keeps the task's tree level.

**Parent drags and keyboard moves (3.20.0+).** Dragging a parent drawn as an ordinary bar, or moving it with the arrow keys, commits the bar and its descendants as one batch. Each task still gets `beforeTaskUpdate`, `taskUpdate` and `taskUpdateSuccess`, and the history entries merge into one undo step, but all of the gesture's `taskUpdate` events fire first and its `taskUpdateSuccess` events follow after the data rebuild. Undo and redo of any entry apply as one batch too. A `beforeTaskUpdate` veto of one child keeps only that child in place; later siblings are still asked and move, and `taskDragged.affectedChildTasks` lists them. Vetoing the parent skips all its children. Descendants with no `startTime` (undated summary rows) are not edited, so they get no events. After every accepted edit the bar's `aria-label`, `aria-valuenow` and `aria-valuetext` are refreshed.

Through 3.19, persist drags, resizes and progress drags from `taskDragged` / `taskResized` / `taskProgressChanged`, and dialog / inline edits from `taskUpdateSuccess`; `updateTask()` fires nothing.

---

## 7. Pitfalls & Anti-Patterns

### Pitfall 1: Date format mismatch

❌ **WRONG**: passing ISO dates without setting the format:
```js
new ApexGantt(el, { series: [{ id: 't1', name: 'X', startTime: '2026-06-01', endTime: '2026-06-15' }] });
// The bars draw (parsing falls back), but edited or dragged dates are written
// back as MM-DD-YYYY, and through 3.19 the edit dialog shows empty date fields.
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
}, [tasks]);  // no cleanup: on unmount the chart's ResizeObservers and global listeners outlive the component
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

### Pitfall 5: A milestone with the wrong `type` casing, or an `endTime`

❌ **WRONG**: `'Milestone'` is not recognised, so this row is not a milestone, and with a `type` set but no `endTime` it throws:
```js
{ id: 'm1', name: 'Launch', type: 'Milestone', startTime: '02-20-2026' }
```

❌ **ALSO WRONG on 3.19 and earlier**: no `type` and no `endTime` throws. From 3.20.0 this row is drawn as a milestone, but say so explicitly:
```js
{ id: 'm1', name: 'Launch', startTime: '02-20-2026' }
```

❌ **ALSO WRONG**: a milestone still draws as a diamond at `startTime`, but the later `endTime` widens the timeline and is announced as a range:
```js
{ id: 'm1', name: 'Launch', type: 'milestone', startTime: '02-20-2026', endTime: '02-25-2026' }
```

✅ **CORRECT**: lowercase type, `startTime` only:
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

❌ Selection APIs return empty / no-op (from 3.20.0 with a one-time console warning):
```js
new ApexGantt(el, { series: tasks });
gantt.getSelectedTasks();  // []
```

✅ — opt in:
```js
new ApexGantt(el, { series: tasks, enableSelection: true });
```

### Pitfall 8: `progress` as a 0–1 fraction

❌ `{ ..., progress: 0.75 }` renders a 0.75% fill, which looks empty.
✅ `{ ..., progress: 75 }`.

### Pitfall 9: Orphan `parentId`

❌ `parentId: 'phase-x'` when no task with id `'phase-x'` exists: through 3.19 the row is not rendered at all, in the list or the timeline; from 3.20.0 it is attached to the root with a console warning.
✅ Make sure every `parentId` refers to a task in the same `series` array.

### Pitfall 10: Forgetting to set the license key

❌ The chart works without a key, but renders with a trial watermark. ApexGantt is included in the Premium plan and above; there is no free tier, so production use needs a key (from 3.20.0 a key below Premium does not remove the Gantt watermark).
✅ Set your license key once at application start, before constructing any chart:
```js
import { ApexGantt } from 'apexgantt';
ApexGantt.setLicense('YOUR_LICENSE_KEY');
```

> **License signature verification (3.17.0+):** license keys issued since 2026-07-27 are ECDSA-signature-verified at runtime; a tampered key renders the chart with a watermark. Unsigned (older) keys remain valid until 2027-07-31. Use the exact key string ApexCharts issued you, verbatim.

### Pitfall 11: Mutating `series` in place

❌ Direct mutation doesn't notify the chart:
```js
options.series.push(newTask);  // chart still shows old data
```

✅ Go through the API. For one task, use the CRUD methods (undoable, fire events):
```js
gantt.addTask(newTask);
```
To swap in a whole new dataset, pass `series` to `update()` (from 3.20.0 this also clears the undo history):
```js
gantt.update({ series: freshTasks });
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

### Pitfall 13: Saving the same edit twice (3.20.0+)

❌ From 3.20.0 a drag fires both `taskDragged` and `taskUpdateSuccess` (`source: 'drag'`), so saving from both posts every drag twice:
```js
container.addEventListener(GanttEvents.TASK_DRAGGED, (e) => save(e.detail.taskId));
container.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, (e) => save(e.detail.taskId));
```
✅ On 3.20.0+ save from `taskUpdateSuccess` alone, and roll back a failed save with `gantt.updateTask(id, previous, { silent: true })` so the rollback does not fire another save. See `references/events.md`.

### Pitfall 14: Expecting links to reschedule tasks

❌ Moving a predecessor and expecting the successor to follow. Links are drawn and analysed (critical path, Predecessors / Successors columns) but never move tasks, whatever their `type` or `lag`.
✅ Move dependent tasks yourself (`updateTask`) if your app needs that.

### Pitfall 15: Saving `getTasks()` alone and losing links

❌ `getTasks()` carries each task's loaded `dependency` only, so links added with `addDependency()` or drawn by the user vanish on reload:
```js
await save({ tasks: gantt.getTasks() });
```
✅ Save `getDependencies()` (3.20.0+) with the tasks, and after loading add back each link the tasks do not already carry, then clear the history so the restore cannot be undone:
```js
await save({ tasks: gantt.getTasks(), links: gantt.getDependencies() });

// Load
const { tasks, links } = await load();
const gantt = new ApexGantt(el, { series: tasks });
gantt.render();
for (const { fromId, toId, type, lag } of links) {
  if (gantt.canAddDependency(fromId, toId).ok) gantt.addDependency(fromId, toId, { type, lag });
}
gantt.clearHistory();
```
Alternatively record links as they change from `dependencyAdded` / `dependencyRemoved` (both fire for drawn links and arrow deletes, and `dependencyRemoved` also when a deleted task takes its links with it).

### Pitfall 16: Calling `addDependency()` without checking first

❌ From 3.20.0 `addDependency()` throws for a self-link, a link that would close a cycle, and (unless `dependencies.allowSummaryDescendantLinks: true`) a link between a task and its own ancestor or descendant. Code that relied on it accepting those now throws.
✅ `if (gantt.canAddDependency(a, b).ok) gantt.addDependency(a, b);` and read `reason` when it is not.

### Pitfall 17: Ignite UI theme ignored, or stuck light (3.21.0)

❌ An Ignite UI stylesheet on the page and no add-on: every `--ig-*` token is ignored silently. Or `theme: 'light'` passed (directly, or by the `theme` prop default of vue-apexgantt 1.2.0 and earlier) on a page with a dark Ignite UI theme: the pinned theme blocks the dark variant.
✅ `import 'apexgantt/ignite-ui'` once (free) before the first render, leave `theme` (and `backgroundColor`) unset, and call `gantt.render()` after swapping the Ignite UI stylesheet.

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
    progress: { key: 'completion', transform: (v) => v * 100 },  // fraction to percent
  },
});
```

Supported parse keys: `id`, `name`, `startTime`, `endTime`, `progress`, `type`, `parentId`, `dependency`, `barBackgroundColor`, `rowBackgroundColor`, `collapsed`. From 3.20.0 also `showSummaryBar`, `baseline`, `segments`, `assignees` and `barTextColor`.

`id`, `name` and `startTime` must be mapped. Through 3.19 every row whose mapped start date is empty is skipped with a warning, so a parent row with no dates of its own disappears. From 3.20.0 a row with no start date is kept only when it is the parent of a dated row (directly or further down), and gets its span from its children (a parent with only an end date included). Any other row with no start date (an unscheduled backlog item, a row with only an end date, null dates) is skipped with the warning `DataParser: Skipping item at index N: it has no start date and no dated child rows`.

---

## 9. Working Calendar & Sub-Day Scheduling

### `calendar` (`CalendarOptions`)

When active, weekends + holidays drive duration math, summary aggregation, and timeline stripes. No `calendar` option means every day is a working day.

```js
new ApexGantt(el, {
  series: tasks,
  calendar: {
    workingWeekdays: [1, 2, 3, 4, 5],          // 0=Sun ... 6=Sat; default Mon-Fri
    holidays: ['12-25-2026', { date: '01-01-2027', label: 'New Year' }],
    showNonWorkingStripes: true,               // tinted bands over non-working columns; default true
    holidayTooltip: ({ date, label }) => `<b>${label ?? 'Holiday'}</b>`,
    dragSnapMode: 'next',                       // 'next' (default) | 'previous' | 'allow'
  },
});
```

**When the calendar turns on.** From 3.20.0 any `calendar` object you pass, even `{}`, turns on the working calendar with a Mon-Fri week. Through 3.19 it turns on only when `workingWeekdays` or `holidays` is set, so `calendar: { showNonWorkingStripes: true }` alone does nothing.

**Stripes and holiday tooltips.** From 3.20.0 the stripes draw over the row backgrounds and under the bars, and `holidayTooltip` shows on hover. Through 3.19 the opaque row backgrounds cover the stripes and the tooltip does not show.

**Holidays and weekdays (3.20.0+).** Holiday strings are read with `inputDateFormat`, falling back to ISO, so in a `'DD-MM-YYYY'` chart `'03-04-2026'` is 3 April (through 3.19 it can read as 4 March, and `'25-12-2026'` can be dropped). `workingWeekdays` accepts numeric strings (`'1'`). A list that names no weekday from 0 to 6 (`[]`, `[7]`) logs a warning and falls back to Monday to Friday; through 3.19 it makes every day non-working.

`dragSnapMode` controls what happens when a drag/resize lands on a non-working day: `'next'`/`'previous'` snap to the adjacent working day (drag preserves working-day duration), `'allow'` permits it. Dependency `lag` counts working days while a calendar is active (calendar days otherwise) and only feeds the critical path and the arrow tooltip. `TaskDependency.lagUnit` has no effect and is deprecated in 3.20.0. From 3.20.0 the `BaselineVariance` column also counts working days (suffix `wd`) while a calendar is active.

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

From 3.20.0, keyboard and mouse moves and resizes with `snapUnit: 'day'` step by calendar days, so in a time zone with daylight saving a one-day step across the clock change lands on the next date (through 3.19 it can leave the date unchanged, stretching the task, or jump two days). Hour and minute snaps still move by elapsed time.

**Duration column and its inline editor (3.20.0+).** The column shows base units of `snapUnit` (`45 m`, `6 h`, `14 d`), or working days (`5 wd`) when a calendar is set with day snapping, and the inline Duration editor uses the same rule for every date format: with a format that carries time, day snapping counts day boundaries and the new end keeps the current end's time of day. Sorting, filtering and grouping on Duration use that same count, so a filter rule `{ field: ColumnKey.Duration, operator: 'equals', value: 5 }` matches a task showing `5 d` or `5 wd` (through 3.19 sort and filter compared milliseconds), and grouping with hour or minute snapping gives groups such as `4 h` / `8 h`.

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

**Read and expand (3.20.0+)**: `gantt.getTask(id)` / `gantt.getTasks()` return copies with current values (nested `baseline`, `segments`, `assignees` and object `dependency` included); `gantt.getDependencies()` returns every link, including added and drawn ones that the tasks do not carry; `gantt.expandTask(id)` / `gantt.collapseTask(id)` change view state only (not undoable).

**Tree level (3.20.0+)**: a committed edit (drag, `updateTask`, dialog, inline, keyboard) keeps the task's level, so `getTask().level`, `taskUpdateSuccess.updatedTask.level`, the row indent and `aria-level` stay right after later renders. `moveTask` sets the moved subtree's new depth, and `addTask` under a parent starts at the parent's depth plus one.

**Undo/redo**: every mutating call (drag, resize, progress, edit, add/delete/move, dependency change) is recorded unless `history: { enabled: false }`. Ctrl/Cmd+Z / Ctrl+Y work inside the chart automatically.

```js
gantt.undo(); gantt.redo();
if (gantt.canUndo()) { /* enable Undo button */ }
container.addEventListener(GanttEvents.HISTORY_CHANGE, (e) => {
  const { kind, canUndo, canRedo } = e.detail;   // keep external buttons in sync
});
```

**Veto hooks**: synchronous, each takes one context object; return `false` to cancel (a returned `Promise` is not a veto):

```js
new ApexGantt(el, {
  series: tasks,
  beforeTaskAdd:          ({ input, parentId }) => input.name.trim().length > 0,
  beforeTaskUpdate:       ({ task, updates }) => !task.locked,
  beforeTaskMove:         ({ task, oldParentId, newParentId }) => true,
  beforeTaskDelete:       ({ task, descendantIds, cascade }) => true,
  beforeDependencyChange: ({ change, fromId, toId, type, lag }) => true,
});
```

`beforeTaskUpdate` coverage: through 3.19 it runs only for `updateTask()` and inline cell edits. From 3.20.0 it runs for every edit path (drag, resize, progress drag, dialog save, inline edit, keyboard move/resize, `updateTask()`), and a veto repaints the bar from the model. During a parent drag or keyboard move it is asked once per moved task: a veto of one child keeps only that child in place, a veto of the parent skips all its children (see §6).

`beforeTaskAdd`'s `parentId` (3.20.0+) is the parent the task lands under: the `addTask()` option, else `input.parentId`, and `undefined` for an id that matches no task. Through 3.19 `input.parentId` is not reported, so a hook keyed on the parent is bypassed.

**Interaction toggles** expose the same operations to the user: `enableInlineEdit`, `enableProgressDrag`, `enableTaskEditingShortcuts`, `enableTaskCRUDToolbar`, `enableContextMenu`, `enableAddTaskRow` (see §3). From 3.20.0 a focused bar also answers ArrowLeft/ArrowRight (move one snap unit, `enableTaskDrag`) and Shift+ArrowLeft/ArrowRight (resize its end, `enableTaskResize`), mirrored in RTL, through the same commit path as dragging (`source: 'keyboard'`, not `taskDragged` / `taskResized`), and Escape cancels drawing a task or a dependency wherever focus is. With a calendar the keys skip non-working days in the direction of travel (`dragSnapMode: 'allow'` is honoured), a move also moves the bar's children, and bars carry `aria-keyshortcuts` listing the keys that work.

**Edit dialog and inline editors (3.20.0+)**: the dialog reads ISO and other date formats, shows an empty field for an invalid date (never "Invalid Date"), and uses `datetime-local` for a format with time. A vetoed save keeps the dialog open; a submit with no changes, or an inline Start / End / Duration editor left without a change, commits nothing (no events, no undo step). Typed spaces and Home/End stay in an inline editor, and cancel, veto or no-change restores the cell with its expand arrow working. `taskUpdateError` fires on every path; `updateTask` still throws afterwards, interactive paths log, repaint and return.

---

## 11. What changed in 3.20.0

Everything below is also marked "3.20.0+" where it appears above.

- **New methods**: `getTask(id)`, `getTasks()`, `getDependencies()`, `expandTask(id)`, `collapseTask(id)`; `scrollToTask(id, { expand })` (expands collapsed ancestors by default); `updateTask(id, patch, { silent })`. `getTask` / `getTasks` return copies (nested fields included) and are not a complete save: save `getDependencies()` with them.
- **`update()` keeps task data** unless you pass `series` or `parsing`; passing `series` clears undo history; `calendar` / `inputDateFormat` rebuild the working calendar. A theme switch resets only the theme's 18 colour options; every other option carries over. A throwing `update({ series })`, `addTask` or `updateTask` leaves the previous data (and options) in place.
- **Edits always land**: a change to a task whose row is not rendered is stored; `updateTask(id, { collapsed })` re-renders; dragging a parent moves collapsed children too; an edit keeps the task's tree level.
- **One event stream**: `beforeTaskUpdate` on every edit path; `taskUpdate` / `taskUpdateSuccess` once per committed edit with `source`; `taskDragged` / `taskResized` / `taskProgressChanged` only after the commit is accepted; `taskUpdateError` on every path. Apps saving on both `taskDragged` and `taskUpdateSuccess` now save twice. A parent drag or keyboard move commits as one batch: all its `taskUpdate` events first, then its `taskUpdateSuccess` events, one undo step; a veto of one child keeps only that child. `beforeTaskAdd` / `taskAdded` report `input.parentId` too.
- **Dependencies**: `addDependency()` throws for a self-link, a cycle and (unless `allowSummaryDescendantLinks`) a summary-descendant link, as `canAddDependency()` reports. Before, only a missing task or a duplicate threw.
- **Data**: a task with no `endTime` and no `type` is a milestone, or a summary bar when it has children; a parent may give only one date; a task with no `startTime` and no children throws `Task "<id>" has no startTime and no children. ...`; parsing keeps a row with no start date only when it is the parent of a dated row; tasks with an unknown or cyclic `parentId` are attached to the root with a warning; ids are matched by value; a single segment is stored as plain dates; `ParsingConfig` gains `showSummaryBar`, `baseline`, `segments`, `assignees`, `barTextColor`; `TaskInput.barTextColor`; `TaskDependency.lagUnit` deprecated (it never had an effect).
- **Options**: any `calendar` object turns on a Mon-Fri week; holiday strings are read with `inputDateFormat`; `workingWeekdays` takes numeric strings and an empty list falls back to Mon-Fri with a warning; an omitted `height` fills a container that has its own height (virtualised rows fill it on first paint); `ToolbarSelect.value`; annotation `borderDashArray` and `label.orientation` honoured, with a dashed line's gaps left empty; `tooltipTemplate(task, fmt, context)` with `context.durationText`; built-in columns given by key with no `title` get their standard title.
- **Display**: Duration column in base units (`45 m` with `snapValue: 15`), and sorting, filtering and grouping on Duration use that count; groups ordered by value with "(None)" last; empty values sort last in both directions; `BaselineVariance` in working days (`wd`) with a calendar; calendar stripes and `holidayTooltip` visible; bars and arrows follow `--apex-gantt-bar-fill` / `--apx-accent` and `--apex-gantt-dependency-line`; `--apex-gantt-font-family` / `--apex-gantt-font-size` reach tooltips and annotation labels; project boundary lines follow edits; a default-height chart keeps the page scroll on re-render; no "Task row element not found" / "Bar row element not found" console warning per hidden task while filtered or collapsed.
- **Editing**: the edit dialog reads ISO and other formats and never writes "Invalid Date"; a vetoed dialog save keeps it open; a dialog or inline editor with no change commits nothing; the inline Duration editor follows the Duration column for every format; day snapping steps by calendar days across daylight-saving changes; `splitTask` accepts a `resumeAt` on the segment's last day (day-only formats) and rejects one that is not after `at`.
- **Keyboard and RTL**: arrow-key move/resize on a focused bar; Escape cancels drawing anywhere; bar aria values refresh after every edit; RTL opens at the start of the timeline (saved horizontal scroll is measured from the right edge, so state saved by older versions restores differently), keeps bar positions after zooming, and keeps the progress handle on the fill edge.
- **Types**: `TaskInput.startTime` optional; `TaskInput.type` accepts the `'milestone'` string; `updateTask` takes `Partial<TaskInput>`; aria label contexts gain a required `task` (direct calls to `DEFAULT_GANTT_MESSAGES.barAriaLabel(...)` must pass it); new exports `TaskUpdateSource`, `GanttUpdateOptions`, `ScrollToTaskOptions`, `UpdateTaskOptions`, `DependencyLink`, `TooltipTemplate`, `TooltipTemplateContext`.
- **License**: only a Premium (or higher) key removes the Gantt watermark, and a lower valid key logs why; a later lower-plan key from any product is ignored with a warning (`setLicense('')` first to downgrade on purpose); `destroy()` releases the container.
- **Script tag**: `window.ApexGantt` carries every runtime export (`GanttEvents`, `TaskType`, `DataParser`, `DEFAULT_GANTT_MESSAGES`, themes and the rest).

---

## 12. Reference Routing Table

For deeper detail and full working examples, refer to:

| Topic | Reference File |
|---|---|
| Task data, hierarchy, milestones, baseline, assignees, summary bars, split tasks | `references/data-format.md` |
| Dependencies (`FS`/`SS`/`FF`/`SF`), lag, critical path | `references/dependencies.md` |
| Column config, `ProgressRing` / `Wbs`, built-in column keys, `renderers`, custom toolbar items, parsing | `references/columns-and-toolbar.md` |
| Task-list grid: sorting, filtering, grouping, auto-size / resize / reorder columns | `references/grid.md` |
| UI state persistence, draw-to-create, scroll-to-task, export (SVG/PNG/PDF) | `references/interaction.md` |
| Events (incl. `source`), selection, edit dialog, drag/resize, progress drag, saving and rolling back edits | `references/events.md` |
| CRUD API, `getTask`/`getTasks`/`getDependencies`, saving and reloading links, `expandTask`/`collapseTask`, veto hooks, undo/redo, calendar, sub-day scheduling, interaction toggles | `references/editing.md` |
| React, Vue 3, Angular wrappers, web components (shadow DOM) | `references/framework-wrappers.md` |
