# Localization: Locale Packs, Messages, Date Formats, Week Start, Fiscal Year

Everything a user reads in the chart can be translated through one `locale`
option, plus two header settings on `calendar`. `direction`, `dateLocale` and
`messages` predate 3.22; everything marked 3.22.0 below is new in that release.
With no `locale` set the chart renders English exactly as before (from 3.22.0
the only markup change on an English chart is an inline `max-width` on the top
header cells and the edit dialog's `aria-labelledby` fix, besides task names
holding `<`, `&`, `"` or `'`, which the built-in tooltips now escape).

## The `locale` option

```js
import ApexGantt from 'apexgantt';
import localeFr from 'dayjs/locale/fr';   // a dayjs locale object (needs the dayjs package)

const gantt = new ApexGantt(el, {
  series: tasks,
  locale: {
    tag: 'fr',                            // 3.22.0
    dateLocale: localeFr,
    displayDateFormat: 'DD/MM/YYYY',      // 3.22.0: read-only dates only
    messages: {
      addTask: 'Ajouter une tâche',
      editTaskTitle: 'Modifier : {name}', // 3.22.0: a template string; a function still works
    },
  },
});
gantt.render();
```

| `locale` key | Type | Default | Notes |
|---|---|---|---|
| `pack` | `GanttLocalePack` | none | (3.22.0) A whole language as plain JSON-safe data, applied **under** every other `locale` field. Object only: a string logs one warning and is ignored. See Locale packs below. |
| `tag` | `string` (BCP 47) | the pack's `tag` | (3.22.0) Sets the chart root's `lang`, the plural rules and number formatting of message templates, the collation used for text sorting and grouping, and turns the progress fields (task form and inline editor) into text fields with a decimal keyboard so a full-width IME value (`５０`) reads as 50. A malformed tag (`'ja_JP'`, should be `'ja-JP'`) is ignored as if unset. Without a tag or pack the root's `lang` is never touched. |
| `direction` | `'ltr' \| 'rtl' \| 'auto'` | `'ltr'` (else the pack's `dir`) | `'rtl'` mirrors the whole chart, time axis included (time flows right to left). |
| `dateLocale` | `GanttDateLocale` | the pack's `dateLocale` | A `dayjs/locale/*` object (ApexGantt registers it on its own bundled dayjs). Localizes month, weekday and AM/PM names in the timeline header, the dates in bar aria-labels and the crosshair weekday. Its `weekStart` sets where week cells start unless `calendar.weekStart` is set (3.22.0). It does **not** change dates shown with `inputDateFormat` (task list, tooltips, task form); use `displayDateFormat` for the read-only ones. From 3.22.0 `meridiem` may be an `[am, pm]` pair and `ordinal` a `'{n}日'` template, so the object can be JSON. |
| `displayDateFormat` | `string` (dayjs pattern) | none | (3.22.0) Pattern for **read-only** dates: the Start, End, Baseline Start and Baseline End columns and the bar, milestone and baseline tooltips, in the date locale. Editors keep `inputDateFormat`, and stored dates are unchanged. Unset: the pack's `displayDateTimeFormat` when `inputDateFormat` has time tokens, else the pack's `displayDateFormat`, else `inputDateFormat`. |
| `crosshairFormat` | `{ date?, time? }` | `'ddd MM/DD/YYYY'` / `'MM/DD HH:mm'` | (3.22.0) Patterns of the default crosshair label: `date` on day-and-coarser tiers, `time` on halfday, hour and minute tiers. The top-level `crosshairLabelFormat` function wins over both. |
| `timelineHeaderFormat` | `TimelineHeaderFormat` | built-in labels | (3.22.0) Header labels per tier and row. See Timeline header formats below. |
| `messages` | `GanttMessagesInput` | `{}` (English, `DEFAULT_GANTT_MESSAGES`) | Any subset of the generated strings. See Messages below. |

Two related settings live on `calendar` (3.22.0) and only shape the header:

| `calendar` key | Range | Default | Notes |
|---|---|---|---|
| `weekStart` | `0` (Sun) to `6` (Sat) | the pack's `weekStart`, else the date locale's `weekStart`, else `0` | Week cells in the header start on this day, and week numbers (`w`, the built-in `Wk` labels) follow it. Anything else logs a warning and is ignored, so the pack's or date locale's week start applies. |
| `fiscalYearStartMonth` | `1` (Jan) to `12` (Dec) | the pack's `fiscalYearStartMonth`, else `1` | Quarter and year cells start on fiscal boundaries. Built-in labels read as fiscal periods: with `4`, April 2026 to March 2027 reads `FY2026` (named by the year it starts in) and its first quarter `#1 Apr-Jun`. Month, week and day cells are unaffected. Anything else logs a warning and the pack's value is used, else January. |

A `calendar` holding **only** `weekStart` and/or `fiscalYearStartMonth` does not turn the working calendar on: every day stays a working day. Add `workingWeekdays`, `holidays` or any other calendar key and it turns on as usual (Mon-Fri by default).

```js
new ApexGantt(el, {
  series: tasks,
  calendar: { weekStart: 1, fiscalYearStartMonth: 4 },   // Monday weeks, April fiscal year; no working calendar
});
```

**Changed in 3.22.0: week cells follow the week start.** With a `dateLocale` whose `weekStart` is not 0 (Monday for `de`, `fr`, `ru`, `en-gb` and most European locales; Saturday for `ar`), week cells now start on that day and each cell's week number matches it. Through 3.21 the cells always ran Sunday to Saturday while the numbers followed the locale, so a `de` chart labelled the week starting Sunday 4 January 2026 as week 1. English charts, and `ja` without the pack, are unchanged (Sunday). To keep Sunday weeks under such a locale, set `calendar: { weekStart: 0 }`.

Other 3.22.0 display changes under a `dateLocale`: bar aria-labels use the locale's long date (`LL`: `2026年1月4日`, `4 janvier 2026`; without a `dateLocale` they stay `January 4, 2026`), the default crosshair label names the weekday in the locale, half-day labels take the meridiem from 09:00 and 15:00 (so `ar`, `zh` and `ru` show distinct morning and afternoon names), and the header rows get `lang` set to the locale's name.

## Locale packs (3.22.0)

A pack sets a whole language at once: messages, date names, header word order, read-only date pattern, crosshair patterns, week start and fiscal year start. It is the base layer; every explicit `locale.*` and `calendar.*` option wins over it, and a pack never turns working days on.

ApexGantt ships one first-party pack, **Japanese**, at `apexgantt/locales/ja`, in its own file (about 2.8 KB gzip; the core bundle does not carry it).

```js
// ESM
import ApexGantt from 'apexgantt';
import { ja } from 'apexgantt/locales/ja';      // also the default export

new ApexGantt(el, { series: tasks, locale: { pack: ja } }).render();
```

```js
// CommonJS
const { ja } = require('apexgantt/locales/ja');
```

```html
<!-- Script tag: sets window.ApexGanttLocales.ja; load before or after the library -->
<script src="https://cdn.jsdelivr.net/npm/apexgantt@3.22.0/apexgantt.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/apexgantt@3.22.0/locales/ja.min.js"></script>
<script>
  new ApexGantt(document.getElementById('gantt'), {
    series: tasks,
    locale: { pack: ApexGanttLocales.ja },
  }).render();
</script>
```

The pack is also published as JSON at `apexgantt/locales/ja.json` (for tooling, or a server that sends it to the page). Types: `locales/ja.d.ts` for ESM; the CommonJS `locales/ja.d.cts` needs TypeScript 5.3 or later under `node16` / `nodenext` resolution.

What the `ja` pack sets:

| Field | Value |
|---|---|
| `tag` | `'ja'` (root `lang="ja"`, Japanese plural, number and collation rules) |
| `status` | `'draft'` |
| `messages` | All 123 string keys (every `GanttMessages` key except `filterOperatorLabel`, which reads the 15 translated `filterOperator*` keys) |
| `dateLocale` | dayjs `ja` data as plain data (month and weekday names, `L`/`LL`, 午前/午後, `{n}日` ordinals) |
| `timelineHeaderFormat` | Japanese word order: `2025年12月` over `12/29～` at week zoom; `2026年度` over `2026年度 第1四半期` zoomed out |
| `displayDateFormat` / `displayDateTimeFormat` | `'YYYY/MM/DD'` / `'YYYY/MM/DD HH:mm'` |
| `crosshairFormat` | `{ date: 'YYYY/M/D(ddd)', time: 'M/D H:mm' }` |
| `weekStart` | `1` (Monday) |
| `fiscalYearStartMonth` | `4` (April fiscal year, 年度) |

**Draft status.** `ja.status` is `'draft'`: the wording awaits a native-speaker review and may change in a minor release when it is signed off. Do not snapshot-test against its exact strings, and do not tell users the translation is final. Override any part of it locally:

```js
locale: {
  pack: ja,
  messages: { addTask: '追加' },                                 // one key; the rest stay the pack's
  timelineHeaderFormat: { week: { sub: 'M/D～{end:M/D}' } },     // one row; the rest stay the pack's
},
calendar: { weekStart: 0, fiscalYearStartMonth: 1 },             // Sunday weeks, calendar-year quarters
```

`update({ locale: { pack: undefined } })` returns the chart to English.

**Custom packs.** Any object of the `GanttLocalePack` shape is a pack: `tag` and `status` (`'draft' | 'reviewed'`) are required; `dir`, `messages`, `dateLocale`, `timelineHeaderFormat`, `displayDateFormat`, `displayDateTimeFormat`, `crosshairFormat`, `weekStart` and `fiscalYearStartMonth` are optional. Keep every message a string (template strings for the function keys) and the pack stays JSON. Pack messages are inserted like `locale.messages` (not escaped), so never build one from untrusted input.

### Precedence

| Setting | Highest to lowest |
|---|---|
| Messages, per key | `locale.messages`, `pack.messages`, English |
| Date locale | `locale.dateLocale`, `pack.dateLocale`, none |
| Header format, per tier and row | `locale.timelineHeaderFormat`, `pack.timelineHeaderFormat`, built-in |
| Week start | `calendar.weekStart`, `pack.weekStart`, the date locale's `weekStart`, `0` |
| Fiscal year start | `calendar.fiscalYearStartMonth`, `pack.fiscalYearStartMonth`, `1` |
| Direction | `locale.direction` (when set), `pack.dir`, `'ltr'` |
| Language tag | `locale.tag`, `pack.tag`, none |
| Read-only date pattern | `locale.displayDateFormat`, the pack's `displayDateTimeFormat` (time-bearing `inputDateFormat`) or `displayDateFormat`, `inputDateFormat` |
| Crosshair label | `crosshairLabelFormat` function, `locale.crosshairFormat`, `pack.crosshairFormat`, built-in |

## Messages

`locale.messages` overrides any subset of the strings the chart generates: toolbar (zoom, add, delete, undo, redo, export, selection count), quick filter and filter builder, context menu, task form and validation, bar and baseline tooltips, column titles, durations, the add-task row, the export file name, and every aria-label. Unset keys keep the pack's text, else the English defaults, exported as `DEFAULT_GANTT_MESSAGES`.

**New keys in 3.22.0** (72, each defaulting to the English text it replaces, so English output is unchanged):

- Bar tooltip labels: `tooltipTaskLabel`, `tooltipStartLabel`, `tooltipEndLabel`, `tooltipDurationLabel`, `tooltipProgressLabel`, `tooltipDateLabel`, `tooltipDependsOnLabel` (separate from the baseline tooltip's `startLabel` / `endLabel`).
- Column titles: `columnName`, `columnStartTime`, `columnEndTime`, `columnDuration`, `columnProgress`, `columnProgressRing`, `columnAssignees`, `columnPredecessors`, `columnSuccessors`, `columnBaselineStart`, `columnBaselineEnd`, `columnBaselineVariance`, `columnWbs`. A `columnConfig[].title` still wins.
- Values: `durationDays` (`'{count} d'`), `durationWorkingDays` (`'{count} wd'`), `durationHours`, `durationMinutes`, `progressValue` (`'{value}%'`), `listSeparator` (`', '`), `tooltipDependencyValue` (`'{taskId} · {type}{lag}'`), `dependencyLag` (`' ({lag}d)'`), `dependencyTypeFS` / `SS` / `FF` / `SF`.
- Toolbar and data: `selectedCount` (`'{count} selected'`), `newTaskName` (`'New task'`), `exportFileName` (`'gantt-chart-{timestamp}'`, extension added after it).
- The 15 filter operators: `filterOperatorContains`, `filterOperatorNotContains`, `filterOperatorEquals`, `filterOperatorNotEquals`, `filterOperatorStartsWith`, `filterOperatorEndsWith`, `filterOperatorGt`, `filterOperatorGte`, `filterOperatorLt`, `filterOperatorLte`, `filterOperatorBefore`, `filterOperatorAfter`, `filterOperatorOn`, `filterOperatorIsEmpty`, `filterOperatorNotEmpty`.
- Aria labels: `summaryAriaLabelExpand`, `summaryAriaLabelCollapse`, `barAriaLabelWithProgress`, `progressRingAriaLabel`, `taskListAriaLabel` (`'Gantt task list'`; the `taskListAriaLabel` option wins), `timelineHeaderAriaLabel`, `selectAllAriaLabel`, `selectTaskAriaLabel`, `expandAriaLabel`, `collapseAriaLabel`, `columnHeaderAriaLabel`, `sortAscending`, `sortDescending`, `sortNone`, `sortPosition`, `resizeColumnAriaLabel`, `resizePanelsAriaLabel`, `scrollToTaskEarlier`, `scrollToTaskLater`, `dependencyAriaLabel`, `removeDependencyAriaLabel`, `closeDialog`.

### Template strings (3.22.0)

A string message fills `{param}` placeholders, and also understands `{n, number}` (a number formatted in the chart's `tag`) and `{n, plural, one {...} other {...}}` (plural rules of the `tag`):

```js
messages: {
  selectedCount: '{count, plural, one {# task selected} other {# tasks selected}}',
  durationDays: '{count, number} d',
}
```

The seven **function keys** also take a template string from 3.22.0, so a whole `messages` object can be JSON (or a Blazor `Dictionary<string, string>`). A function you pass is used as it is.

| Function key | English template | Params |
|---|---|---|
| `editTaskTitle` | `'Edit Task: {name}'` | `name` |
| `groupCountLabel` | `'({count})'` | `count` |
| `filterOperatorLabel` | `'{operator}'` | `operator` (already the `filterOperator*` text, so setting those 15 keys is usually enough) |
| `summaryAriaLabel` | `'{name} (summary)'` | `name`; a toggleable summary reads `summaryAriaLabelExpand` / `summaryAriaLabelCollapse` instead |
| `barAriaLabel` | `'{name}: {start} to {end}'` | `name`, `start`, `end`; a read-only bar reads `barAriaLabelWithProgress` (adds `progress`) |
| `barAriaValueText` | `'{start} to {end}, {durationDays} days'` | `start`, `end`, `durationDays` |
| `progressAriaLabel` | `'{name} progress'` | `name` |

**Types (changed in 3.22.0).** `locale.messages` is `GanttMessagesInput`: every key optional, and each function key `function | string`. Code that reads a function key back from your own options object needs a `typeof` check before calling it. `GanttMessages` (the resolved shape) gained 72 required keys, so a complete `GanttMessages` literal written by hand no longer type-checks: spread `DEFAULT_GANTT_MESSAGES` into it, or type it as `GanttMessagesInput`. The resolved messages the chart hands out (`TooltipTemplateContext.messages`) still hold functions.

**Messages are not escaped.** Message text is inserted as text or HTML depending on where it appears, so never put untrusted input in `messages` or a pack. Task data passed as params (names, ids) keeps its own escaping. The two messages that land in an attribute inside markup (`progressRingAriaLabel`, `closeDialog`) are escaped as a whole.

## Date format tokens (changed in 3.22.0)

Every place ApexGantt formats a date (`inputDateFormat`, `displayDateFormat`, `crosshairFormat`, `timelineHeaderFormat` patterns) now reads these tokens on top of the core dayjs ones: `Q` (quarter), `Do` (ordinal day), `k` / `kk` (hour 1 to 24), `X` / `x` (Unix seconds / milliseconds), `w` / `ww` / `wo` (week of year, following the week start), `W` / `WW` (ISO week), `gggg` / `GGGG` (week year, ISO week year), and `L` / `LL` / `LLL` / `LLLL` with their lowercase forms (the date locale's own formats).

**Upgrade check for `inputDateFormat`.** A format that held one of `Q`, `Do`, `k`, `w`, `W`, `X`, `x`, `L`, `l`, `gggg` or `GGGG` as literal text (`LT` and `ll` included) now prints a value there. Wrap literal text in square brackets (`'YYYY-MM-DD [Week]'`); the chart logs one warning when it finds such a letter outside brackets. The default `'MM-DD-YYYY'` and the usual `'YYYY-MM-DD'`, `'DD/MM/YYYY'`, `'YYYY-MM-DD HH:mm'` are unaffected. `z` / `zzz` still print as written (no timezone plugin is loaded).

## Timeline header formats (3.22.0)

`locale.timelineHeaderFormat` relabels the header one tier and row at a time; cell positions and widths never change. Tiers: `minute`, `hour`, `halfday`, `day`, `week`, `month`, `quarter`, `year`; rows: `top` (coarse) and `sub` (fine). Rows left unset keep the built-in labels.

```js
import ja from 'dayjs/locale/ja';

new ApexGantt(el, {
  series: tasks,
  calendar: { weekStart: 1, fiscalYearStartMonth: 4 },
  locale: {
    dateLocale: ja,
    timelineHeaderFormat: {
      year:    { top: '{fy}年度', sub: '{fy}年度' },
      quarter: { sub: [{ minWidth: 160, format: '{fy}年度 第{fq}四半期' }, { format: '第{fq}四半期' }] },
      month:   { top: 'YYYY年M月', sub: 'M月' },
      week:    { top: 'YYYY年M/D～', sub: 'M/D～' },
      day:     { sub: 'D' },
    },
  },
});
```

A row takes:

- **a pattern string**, used at every width (a label wider than its cell is cut with an ellipsis; `''` blanks the row). Latin literal text goes in brackets (`'[Wk] w'`); CJK text needs none. Placeholders: `{end:PATTERN}` (the cell's last instant, e.g. `'M/D-{end:M/D}'`), `{fy}` and `{fq}` (fiscal year and quarter). `YYYY` and `Q` stay calendar-based.
- **a candidate array** `[{ minWidth?, format }]`, sorted widest-first by the chart; the first whose `minWidth` fits the cell is used. An empty array keeps the built-in labels.
- **a function** `(date, ctx) => string`, called for every cell on every render and zoom step; `ctx` has `tier`, `position`, `cellWidth`, `end`, `week`, `quarter`, `fiscalYear`, `fiscalQuarter`, `locale` and `format(pattern, date?)`. Keep it pure and cheap. A function that throws falls back to the built-in label and logs one warning per row.

Output is HTML-escaped. Which rows show depends on zoom: at week zoom (`pixelsPerDay` about 25.7) the rows are `month.top` over `week.sub`; `week.top` shows only from day-cell zoom (42 px per day or more). Set both rows of a tier when users can zoom. `minute.top`, `halfday.top` and `quarter.top` never render.

In React (or any wrapper that diffs options), pattern strings and arrays can be inline, but define formatter functions outside the component or memoize the options, or every render sends an update.

## Input methods (IME) and search (3.22.0)

- The quick filter waits for a composition to end before filtering (on every chart, not only localized ones). On some Android keyboards that compose Latin words too, the list filters when a word is finished rather than on every letter.
- An Enter that confirms a conversion does not commit an inline edit, and an Escape that cancels one does not close the task dialog.
- The quick filter and the filter builder's text operators compare NFKC-folded text, so full-width and compatibility forms match their plain forms (`ＡＢＣ` finds `abc`, `ｶﾞ` finds `ガ`). What is shown is unchanged.
- With `locale.tag` (or a pack), text sorting and grouping use that language's collation, and the progress fields accept full-width digits.

## Tooltips and escaping (security fix in 3.22.0)

Through 3.21 the default bar tooltip, the baseline tooltip and the edit dialog title wrote the task name into the page as HTML, so a name such as `<img src=x onerror=...>` ran script. From 3.22.0 they show it as text, as do the dependency id and the progress value in the default tooltip. Names without `<`, `&`, `"` or `'` render exactly as before. Upgrade if task names come from users or external data.

A custom `tooltipTemplate` returns HTML and is **still responsible for its own escaping**. Pass every piece of task data through the exported `escapeHtml`. From 3.22.0 its third argument also carries `messages` (the resolved, translated strings), `startText` / `endText` (dates as the task list shows them, in `displayDateFormat` when set) and `tag`, so a custom template can match the default:

```js
import { ApexGantt, escapeHtml } from 'apexgantt';
import { ja } from 'apexgantt/locales/ja';

new ApexGantt(el, {
  series: tasks,
  locale: { pack: ja },
  tooltipTemplate: (task, dateFormat, ctx) => {
    const m = ctx?.messages;
    return `
      <div><strong>${m?.tooltipTaskLabel ?? 'Task:'}</strong> ${escapeHtml(task.name)}</div>
      <div><strong>${m?.tooltipStartLabel ?? 'Start:'}</strong> ${escapeHtml(ctx?.startText ?? '')}</div>
      <div><strong>${m?.tooltipEndLabel ?? 'End:'}</strong> ${escapeHtml(ctx?.endText ?? '')}</div>
      <div><strong>${m?.tooltipDurationLabel ?? 'Duration:'}</strong> ${escapeHtml(ctx?.durationText ?? '')}</div>
      <div><strong>${m?.tooltipProgressLabel ?? 'Progress:'}</strong> ${escapeHtml(task.progress)}%</div>`;
  },
});
```

The same rule holds for every option that returns HTML: column `render` functions, `holidayTooltip`, `dependencies.tooltipTemplate`. Labels from `timelineHeaderFormat` are escaped by the chart.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Expecting `dateLocale` to localize the task-list and tooltip dates | It covers the header, bar aria-labels and the crosshair; set `locale.displayDateFormat` (3.22.0) for read-only dates |
| `locale: { pack: 'ja' }` | A pack is an object: `import { ja } from 'apexgantt/locales/ja'`, then `locale: { pack: ja }` (a string is ignored with a warning) |
| `locale: { tag: 'ja_JP' }` | BCP 47 uses a hyphen: `'ja-JP'`; the underscore form is ignored |
| `calendar: { weekStart: 1 }` expecting Mon-Fri working days | Header-only keys never turn the working calendar on; add `workingWeekdays` to schedule by working days |
| A `de` / `fr` chart on 3.22 whose weeks now start Monday | Intended; set `calendar: { weekStart: 0 }` for Sunday weeks |
| `inputDateFormat: 'YYYY-MM-DD Q1'` style literals | Bracket literal text: `'YYYY-MM-DD [Q1]'` (3.22.0 reads `Q`, `Do`, `k`, `w`, `W`, `X`, `x`, `L`, `l`, `gggg`, `GGGG`) |
| `` tooltipTemplate: (t) => `<b>${t.name}</b>` `` | `escapeHtml(t.name)`; the built-in tooltip escapes from 3.22.0, a custom one never does |
| Calling `options.locale.messages.editTaskTitle(name)` after passing a string | Function keys may be strings in `GanttMessagesInput`; check `typeof`, or read `ctx.messages` |
| Building a full `GanttMessages` literal by hand | Spread `DEFAULT_GANTT_MESSAGES`, or type it `GanttMessagesInput` |
| Translating messages from user input | Messages are not escaped; keep them static |
| Treating the `ja` pack wording as final | `status: 'draft'`; it may change in a minor release |
