# Columns, Toolbar Items & Parsing

## Column configuration

The task-list panel shows configurable columns. By default all five built-in columns render in this order:

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
| `key` | `ColumnKey` | — | Required. Which built-in column. |
| `title` | `string` | from defaults | Header label. |
| `minWidth` | `string` | `'30px'` | Minimum CSS width (used in `minmax()`). |
| `flexGrow` | `number` | `1` | CSS Grid `fr` proportion. |
| `visible` | `boolean` | `true` | `false` keeps the column in config but hides it. |

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
