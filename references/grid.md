# Task-List Grid — Sorting, Filtering, Grouping, Column Sizing

Everything here turns the task list into a real data grid. All of it is
**view-only**: sorting, filtering, and grouping change which rows render and in
what order, never the underlying task tree, WBS codes, or task data. New in the
3.13.0 feature wave (shipped under v3.14.0).

## Sorting

Sort by any value column. Sorting is **hierarchy-preserving**: siblings reorder
within each parent, the tree is never flattened, and summary rows sort by their
rolled-up span.

```js
import { ColumnKey } from 'apexgantt';

new ApexGantt(el, {
  series: tasks,
  sortBy: { key: ColumnKey.Name, direction: 'asc' },
  // multi-key: first key wins, ties break on the next
  // sortBy: [{ key: ColumnKey.Progress, direction: 'desc' }, { key: ColumnKey.Name }],
});
```

The chart defaults to **start-time ascending**; pass `sortBy: []` for natural
(input) order. A `SortCriterion` is `{ key: ColumnKey | string; direction?: 'asc' | 'desc' }`
(`direction` defaults to `'asc'`).

Runtime API:

```js
gantt.sort({ key: ColumnKey.Progress, direction: 'desc' });   // single or SortCriterion[]
gantt.toggleSort(ColumnKey.Name);                    // header-click cycle: asc → desc → none
gantt.toggleSort(ColumnKey.StartTime, { append: true }); // Shift+click: add a secondary key
gantt.clearSort();                                   // back to natural order
gantt.getSort();                                     // → SortCriterion[] (empty = natural)
```

Clicking a sortable column header cycles the sort; Shift+click adds it as an
extra key. Every change emits a `sortChange` event
(`{ criteria: { key, direction }[], timestamp }`). Only columns that are
`sortable` participate (see `references/columns-and-toolbar.md`); the `Wbs`
column is not sortable.

## Filtering

Two filter surfaces, both view-only. A task is kept when it matches **or has a
matching descendant**, so ancestors of matches stay visible for context.

### Quick filter (toolbar search box)

```js
new ApexGantt(el, {
  series: tasks,
  enableQuickFilter: true,
  quickFilter: { placeholder: 'Find a task…', fields: ['name'], caseSensitive: false },
});
```

`QuickFilterOptions`: `{ placeholder?, fields? /* default ['name'] */, caseSensitive? /* default false */ }`.

### Advanced filter builder (popover)

A "Filter" toolbar button (with an active-rule count badge) opens a popover for
composing field / operator / value conditions combined with **All** (AND) or
**Any** (OR).

```js
new ApexGantt(el, {
  series: tasks,
  enableFilterBuilder: true,
  filterRules: {
    match: 'all',                                     // 'all' (AND) | 'any' (OR)
    rules: [{ field: ColumnKey.Progress, operator: 'lt', value: 100 }],
  },
});
```

A `FilterRule` is `{ field: ColumnKey | string; operator: FilterOperator; value?: string | number }`.
`FilterOperator` is one of:
`'contains'`, `'notContains'`, `'equals'`, `'notEquals'`, `'startsWith'`,
`'endsWith'`, `'gt'`, `'gte'`, `'lt'`, `'lte'`, `'before'`, `'after'`, `'on'`,
`'isEmpty'`, `'notEmpty'`. Which operators are valid depends on the column's
value type (text / number / date); `isEmpty` / `notEmpty` apply to any type and
ignore `value`.

### Runtime API

```js
gantt.filter((task) => task.progress < 100);         // view-only predicate
gantt.setFilterRules({ match: 'all', rules: [/* … */] }); // structured; null/empty clears
gantt.clearFilter();
gantt.isFiltered();                                  // → boolean
gantt.getFilterRules();                              // → FilterRuleSet | null
```

All emit a `filterChange` event (`{ active, visibleCount, timestamp }`).
`filterRules` takes precedence over `filterBy` when both are set.

## Grouping

Bucket tasks under collapsible group headers (label + member count). While
grouping is active the parent/child tree is **suspended** and every task appears
flat under its group; clearing it restores the exact prior hierarchy.

```js
new ApexGantt(el, { series: tasks, groupBy: ColumnKey.Progress });

// or at runtime
gantt.groupBy(ColumnKey.Progress);
gantt.groupBy({ field: 'status', label: (v) => `Status: ${v}`, direction: 'desc' });
gantt.clearGrouping();
gantt.getGroupBy();                                  // → GroupCriterion | null
gantt.isGrouping();                                  // → boolean
```

A `GroupCriterion` is
`{ field: ColumnKey | string; accessor?: (task) => unknown; label?: (value) => string; direction?: 'asc' | 'desc' }`.
Grouping composes with sort and filter and emits a `groupChange` event
(`{ active, field, groupCount, timestamp }`). A criterion's `accessor` / `label`
functions are **not** serialized by `persistState` / `getState()` (only `field`
and `direction` are), so a restored grouping falls back to the column's own
accessor.

_Deferred:_ nested (multi-level) grouping and per-group summary bars / aggregates
beyond the count.

## Columns: auto-size, resize, reorder

The task-list grid grew four sizing capabilities. See
`references/columns-and-toolbar.md` for the full `ColumnListItem` shape.

**Auto-size (`autoSizeColumns`, default `true`).** Each column sizes to fit its
header title and widest cell content, growing the panel (never below
`tasksContainerWidth`) so nothing is clipped. `minWidth` sets a column's
preferred floor and `maxWidth` (default `'320px'`) the ceiling. The panel stays
freely resizable. Set `false` for the legacy behavior (split
`tasksContainerWidth` purely by `flexGrow`).

**Per-column resize (`resizableColumns`, default `true`).** Drag the handle at a
column header's trailing edge to pin that column to an exact pixel width; the
other columns absorb the leftover space. Double-click a handle to reset to auto
width. Opt one column out with `columnConfig[].resizable: false`.

```js
gantt.setColumnWidth(ColumnKey.Name, 260);
gantt.resetColumnWidths(ColumnKey.Name);   // one column
gantt.resetColumnWidths();                 // all columns
gantt.getColumnWidths();                   // → { key: px, … }
```

**Reorder (`reorderableColumns`, default `true`).** Drag a column header left or
right onto another column to move it; a short drag threshold keeps a plain click
sorting as usual.

```js
gantt.setColumnOrder([ColumnKey.Name, ColumnKey.Progress, ColumnKey.StartTime]);
gantt.getColumnOrder();                    // → string[] (left-to-right)
```

Both resize and reorder are captured by `getState()` / `setState()` (see
`references/interaction.md`) and emit `columnResize`
(`{ key, width, widths, timestamp }`, `width` is `null` on reset) /
`columnReorder` (`{ order, movedKey, timestamp }`, `movedKey` is `null` for a
bulk `setColumnOrder`) events.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Expecting `groupBy` to keep the parent/child tree | Grouping suspends the tree; `clearGrouping()` restores it |
| `operator: 'lessThan'` | Use `'lt'` (also `'gt'`, `'gte'`, `'lte'`, etc.) |
| Sorting a custom column with no `accessor`/`comparator` | Custom columns are sortable only when one is supplied |
| Assuming a filter edits the data | Filters are view-only; the task tree and WBS are untouched |
| Restoring a `GroupCriterion` and expecting its `label`/`accessor` back | Only `field` + `direction` persist; supply the functions again |
