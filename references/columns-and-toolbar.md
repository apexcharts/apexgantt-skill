# Columns, Toolbar Items & Parsing

## Column configuration

The task-list panel shows configurable columns. By default these five columns render in this order:

1. `Name`
2. `StartTime`
3. `EndTime` *(hidden by default)*
4. `Duration`
5. `Progress`

Override with `columnConfig`. **When you supply `columnConfig` it is authoritative — only the columns you list render**, in the order you list them.

```js
import { ColumnKey } from 'apexgantt';

new ApexGantt(el, {
  series: tasks,
  columnConfig: [
    { key: ColumnKey.Name,      title: 'Task Name', minWidth: '120px', flexGrow: 3   },
    { key: ColumnKey.StartTime, title: 'Start',     minWidth: '90px',  flexGrow: 1.5 },
    { key: ColumnKey.Duration,  title: 'Duration',  minWidth: '70px',  flexGrow: 1   },
    { key: ColumnKey.Progress,  title: 'Progress',  minWidth: '70px',  flexGrow: 1   },
  ],
});
```

### `ColumnListItem` properties

| Property | Type | Default | Description |
|---|---|---|---|
| `key` | `ColumnKey \| string` | — | Required. A built-in `ColumnKey`, or any other string for a custom column (needs `render`). |
| `title` | `string` | from defaults | Header label. |
| `minWidth` | `string` | `'30px'` | Minimum CSS width (used in `minmax()`); also the preferred width floor when auto-sizing. |
| `maxWidth` | `string` | `'320px'` | Upper bound for auto-sized width, so one long value can't dominate the panel. Ignored when `autoSizeColumns` is off. |
| `flexGrow` | `number` | `1` | CSS Grid `fr` proportion (distributes any extra width). |
| `visible` | `boolean` | `true` | `false` keeps the column in config but hides it. |
| `resizable` | `boolean` | `true` | Whether this column can be resized by its header handle (needs the `resizableColumns` option). `false` locks it. |
| `render` | `ColumnRenderer` | — | Custom cell renderer `(ctx) => string`. Required for custom columns; ignored for built-ins. |
| `accessor` | `(task) => unknown` | native | Extract the cell's underlying value for sorting / filtering / SVG export. What makes a custom column sortable. |
| `sortable` | `boolean` | see note | Whether the column participates in sorting. Defaults `true` for built-in value columns, `false` for `Wbs`, and `true` for custom columns only when `accessor` or `comparator` is set. |
| `comparator` | `(a, b) => number` | — | Custom sort comparator (takes precedence over `accessor`). Returns negative / zero / positive for ascending order; the active direction is applied on top. |

### Hide vs. omit

```js
// Approach 1: omit columns you don't want
columnConfig: [
  { key: ColumnKey.Name },
  { key: ColumnKey.Progress },
],

// Approach 2: keep them in config and toggle at runtime
columnConfig: [
  { key: ColumnKey.Name },
  { key: ColumnKey.StartTime },
  { key: ColumnKey.Duration, visible: false },     // hidden, easy to toggle later
  { key: ColumnKey.Progress },
],
```

### Built-in column keys

`ColumnKey` has **13** members (all opt-in via `columnConfig` except the default set):

| Key | Renders |
|---|---|
| `Name` | Task name. |
| `StartTime` | Start date. |
| `EndTime` | End date (hidden by default). |
| `Duration` | Span in days. |
| `Progress` | Percent complete. |
| `ProgressRing` | Circular progress gauge (3.12.0). |
| `Wbs` | Auto-numbered work-breakdown outline, e.g. `1.2.1` (3.12.0). Not sortable. |
| `Predecessors` | Upstream dependencies, derived from the dependency graph (3.13.0). |
| `Successors` | Downstream dependencies, derived from the dependency graph (3.13.0). |
| `Assignees` | Assigned people/teams, from `TaskInput.assignees` (3.13.0). |
| `BaselineStart` | Baseline start date (3.13.0). |
| `BaselineEnd` | Baseline end date (3.13.0). |
| `BaselineVariance` | Signed slip between actual and baseline (3.13.0). |

The five columns listed at the top of this file (`Name`, `StartTime`, `EndTime`, `Duration`, `Progress`) are the default-rendered set; the rest are opt-in via `columnConfig`. `Predecessors` / `Successors` refresh as dependencies or WBS change. Every built-in column except `Wbs` is sortable and filterable, and all are groupable (see `references/grid.md`).

### Custom columns & built-in renderers (3.12.0)

A `ColumnListItem.key` can be a **custom string** (instead of a `ColumnKey`) paired with a `render(ctx) => string` returning an HTML string. Use the exported `escapeHtml` helper for any task data you interpolate:

```js
import { ApexGantt, ColumnKey, renderers, escapeHtml } from 'apexgantt';

columnConfig: [
  { key: ColumnKey.Name, title: 'Task' },
  { key: 'owner', title: 'Owner', render: (ctx) => `<span>${escapeHtml(ctx.task.name)}</span>` },
]
```

Two ready-made renderers ship in the `renderers` namespace:

```js
// Stacked assignee avatars — reads TaskInput.assignees (see data-format.md)
{ key: 'team', title: 'Team',
  render: renderers.avatars({ accessor: (task) => task.assignees, max: 4 }) }

// Circular progress ring
{ key: ColumnKey.ProgressRing, title: 'Done',
  render: renderers.progressRing({ size: 32, showLabel: true }) }
```

- `renderers.avatars(options)` — `{ accessor: (task) => Assignee[], max?: 4, size?: 24, overlap?: 8, borderColor?, fallbackColor? }`.
- `renderers.progressRing(options?)` — `{ accessor?: (task) => number /* default task.progress */, size?: 32, strokeWidth?: 3, progressColor?, trackColor?, showLabel?: true, labelColor? }`.

## Custom toolbar items

The built-in toolbar carries zoom in/out and an export button. Add custom controls via `toolbarItems`.

### Button

```js
{
  type: 'button',
  label: 'Send to Buffer',
  tooltip: 'Publish selected tasks',
  position: 'left',                    // 'left' (before built-ins) | 'right' (default)
  requiresSelection: true,             // auto-disable when nothing selected
  showCount: true,                     // append " (N)" to label
  onClick: ({ selectedTasks }) => publish(selectedTasks),
}
```

### Select

```js
{
  type: 'select',
  label: 'Filter',
  placeholder: 'All',
  options: [
    { value: 'open',   text: 'Open' },
    { value: 'closed', text: 'Closed' },
  ],
  onChange: (value, { selectedTasks }) => filterByStatus(value),
}
```

### Separator

```js
{ type: 'separator' }
```

### `ToolbarContext` (passed to callbacks)

```ts
{
  selectedTasks: Task[];   // empty array when nothing selected
}
```

You don't get a reference to the gantt instance in the callback — capture it via closure if you need it:

```js
let gantt;
gantt = new ApexGantt(el, {
  series: tasks,
  enableSelection: true,
  toolbarItems: [{
    type: 'button',
    label: 'Mark complete',
    requiresSelection: true,
    onClick: ({ selectedTasks }) => {
      selectedTasks.forEach(t => gantt.updateTask(t.id, { progress: 100 }));
    }
  }],
});
gantt.render();
```

## Conditional disabled

`disabled` accepts a function for selection-aware logic. `requiresSelection: true` is shorthand for the common case:

```js
{
  type: 'button', label: 'Lock',
  disabled: ({ selectedTasks }) => selectedTasks.length > 5,
  onClick: ({ selectedTasks }) => lock(selectedTasks),
}
```

## Parsing config (mapping non-standard data)

If your API doesn't use the canonical field names, use `parsing` instead of a manual transform:

```js
new ApexGantt(el, {
  series: apiData,
  parsing: {
    id: 'task_id',
    name: 'title',
    startTime: 'start_date',
    endTime: 'end_date',
    progress: { key: 'pct_complete', transform: Number },
    parentId: 'parent.id',                        // dot-notation for nested
  },
});
```

Each value is a dot-notation string path or `{ key, transform }`. `transform` runs after the path is resolved.

**Supported keys:** `id`, `name`, `startTime`, `endTime`, `progress`, `type`, `parentId`, `dependency`, `barBackgroundColor`, `rowBackgroundColor`, `collapsed`.

## Mounting the toolbar elsewhere

`renderToolbar(container)` lets you place the toolbar in a custom DOM slot outside the chart (e.g. inside a sticky page header):

```js
const gantt = new ApexGantt(chartEl, { series: tasks });
gantt.render();
gantt.renderToolbar(document.getElementById('header-toolbar'));
```

The default in-chart toolbar still renders unless you suppress it via your layout / CSS.
