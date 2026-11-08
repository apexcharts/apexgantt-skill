# Framework Wrappers: React, Vue, Angular

ApexCharts ships first-party wrappers for each major framework. They handle ref management, prop diffing, and call `destroy()` automatically on unmount.

All three share one shape (as of react-apexgantt 1.1.2, vue-apexgantt 1.1.3, ngx-apexgantt 1.1.3):

- the task array is the **`tasks`** prop/input (not `series`), and every other core option goes in **`options`**; `width`, `height`, `pixelsPerDay` and `theme` also exist as convenience props;
- event handlers receive the event's **`detail`** object, not the `CustomEvent`;
- these releases bridge 8 core events (Angular: 7, no `taskValidationError`): `taskUpdate`, `taskUpdateSuccess`, `taskUpdateError`, `taskValidationError`, `taskDragged`, `taskResized`, `selectionChange`, `dependencyArrowUpdate`. For the others (CRUD, history, sort, filter, group, column, `taskProgressChanged`), listen on an element that contains the chart: core events bubble, except `dependencyArrowUpdate`. Later wrapper releases bridge all 20; check the wrapper's README;
- each exports `setApexGanttLicense(key)`: set your license key once at application start. ApexGantt is included in the Premium plan and above; there is no free tier, and without a key charts show a trial watermark.

The wrapper releases that require apexgantt 3.20.0 (peer `"apexgantt": ">=3.20.0"`) change these things. ngx-apexgantt stops the core events at the chart so each Output fires once: they no longer bubble out of the component, so listen through the Outputs rather than on a parent element. ngx-apexgantt also supports Angular 18, 19, 20, 21 and 22 (earlier releases: Angular 18 only) and can be server-rendered: on the server it renders its empty container, creates no chart, its methods do nothing and `getInstance()` returns `null`; the chart is created in the browser. react-apexgantt's `onDependencyArrowUpdate` fires (on every frame of a drag, for its own chart only). vue-apexgantt no longer defaults `height` to `'500px'`, so the core's container-height rule applies, and no longer defaults `pixelsPerDay` to `4.9`, so the chart auto-fits like the core. Methods the handles do not wrap, such as `getDependencies()`, are on `getInstance()`.

To save a wrapped chart after the user edited it, save `getInstance().getTasks()` together with `getInstance().getDependencies()` (both apexgantt 3.20.0+), and re-add the links after loading as `references/dependencies.md` shows. A task's `dependency` holds only the link it was loaded with, so added or drawn links are only in `getDependencies()`, which also reflects undo and redo. Do not rebuild the link list from the `dependencyAdded` / `dependencyRemoved` handlers: undo and redo do not fire them, so the list drifts.

## React: `react-apexgantt`

```bash
npm install react-apexgantt apexgantt
```

```jsx
import { useMemo, useState } from 'react';
import { ApexGanttChart, setApexGanttLicense } from 'react-apexgantt';

setApexGanttLicense('YOUR_LICENSE_KEY');   // once, at application start

function ProjectTimeline() {
  const [tasks, setTasks] = useState([
    { id: 't1', name: 'Design', startTime: '2026-01-01', endTime: '2026-01-15', progress: 60 },
    { id: 't2', name: 'Develop', startTime: '2026-01-16', endTime: '2026-02-15', progress: 25, dependency: 't1' },
  ]);
  // Memoise options: a function inside (hook, template) that is new on every render
  // makes the wrapper resend all options to the chart.
  const options = useMemo(() => ({ inputDateFormat: 'YYYY-MM-DD', enableSelection: true }), []);

  return (
    <ApexGanttChart
      tasks={tasks}
      options={options}
      pixelsPerDay={25.7}
      onTaskUpdateSuccess={(detail) => console.log('saved', detail.updatedTask)}
      onTaskDragged={(detail) => console.log('moved', detail.daysMoved)}
    />
  );
}
```

A `ref` gives an `ApexGanttHandle` with `update`, `updateTask`, `zoomIn`, `zoomOut`, `destroy` and `getInstance()` (the underlying `ApexGantt`, for methods the handle does not wrap, such as `addTask`, `getTask`, `getDependencies` or `scrollToTask`).

Vanilla pattern (when you need direct chart access):

```jsx
import { useEffect, useRef } from 'react';
import { ApexGantt, GanttEvents } from 'apexgantt';

function GanttChart({ tasks }) {
  const ref = useRef(null);

  useEffect(() => {
    const gantt = new ApexGantt(ref.current, {
      series: tasks,
      inputDateFormat: 'YYYY-MM-DD',
    });
    gantt.render();

    const handler = (e) => console.log(e.detail);
    ref.current.addEventListener(GanttEvents.TASK_UPDATE_SUCCESS, handler);

    return () => {
      ref.current?.removeEventListener(GanttEvents.TASK_UPDATE_SUCCESS, handler);
      gantt.destroy();
    };
  }, [tasks]);

  return <div ref={ref} />;
}
```

## Vue 3: `vue-apexgantt`

```bash
npm install vue-apexgantt apexgantt
```

```vue
<script setup>
import { ref } from 'vue';
import { ApexGanttChart } from 'vue-apexgantt';

const tasks = ref([
  { id: 't1', name: 'Design',  startTime: '2026-01-01', endTime: '2026-01-15', progress: 60 },
  { id: 't2', name: 'Develop', startTime: '2026-01-16', endTime: '2026-02-15', progress: 25, dependency: 't1' },
]);
const options = { inputDateFormat: 'YYYY-MM-DD', enableSelection: true };

function onTaskDragged(detail) {
  console.log('moved', detail.daysMoved);
}
</script>

<template>
  <ApexGanttChart
    :tasks="tasks"
    :options="options"
    :pixels-per-day="25.7"
    @task-update-success="(detail) => console.log('saved', detail.updatedTask)"
    @task-dragged="onTaskDragged"
  />
</template>
```

`pixelsPerDay` defaults to `4.9` (month density) in vue-apexgantt 1.1.3, not to the core's auto-fit; pass it explicitly there. The vue-apexgantt release built for apexgantt >=3.20.0 has no default and auto-fits on first render; unsetting the prop later keeps the current zoom. A template ref exposes `update`, `updateTask`, `zoomIn`, `zoomOut`, `destroy` and `getInstance()`.

## Angular: `ngx-apexgantt`

```bash
npm install ngx-apexgantt apexgantt
```

`NgxApexGanttComponent` is standalone; `NgxApexGanttModule` wraps it for NgModule apps.

```ts
// main.ts
import { setApexGanttLicense } from 'ngx-apexgantt';
setApexGanttLicense('YOUR_LICENSE_KEY');   // once, before bootstrapping
```

```ts
// project-timeline.component.ts
import { Component } from '@angular/core';
import { NgxApexGanttComponent } from 'ngx-apexgantt';
import type { GanttUserOptions, TaskInput, TaskDraggedEventDetail, TaskUpdateSuccessEventDetail } from 'ngx-apexgantt';

@Component({
  selector: 'project-timeline',
  standalone: true,
  imports: [NgxApexGanttComponent],
  template: `
    <ngx-apexgantt
      [tasks]="tasks"
      [options]="options"
      [pixelsPerDay]="25.7"
      (taskUpdateSuccess)="onSaved($event)"
      (taskDragged)="onDragged($event)">
    </ngx-apexgantt>
  `,
})
export class ProjectTimelineComponent {
  tasks: TaskInput[] = [
    { id: 't1', name: 'Design',  startTime: '2026-01-01', endTime: '2026-01-15', progress: 60 },
    { id: 't2', name: 'Develop', startTime: '2026-01-16', endTime: '2026-02-15', progress: 25, dependency: 't1' },
  ];
  // No `series` here: the wrapper takes it from [tasks] when options.series is unset.
  options = { inputDateFormat: 'YYYY-MM-DD', enableSelection: true } as GanttUserOptions;

  onSaved(detail: TaskUpdateSuccessEventDetail) {
    console.log('saved', detail.updatedTask);
  }

  onDragged(detail: TaskDraggedEventDetail) {
    console.log('moved', detail.daysMoved);
  }
}
```

In ngx-apexgantt 1.1.3 the inputs are read when the chart is created; changing `[tasks]` or `[options]` later is not applied. Get the component with `@ViewChild(NgxApexGanttComponent)` and call its `update()` / `updateTask()` instead.

## Typed events in wrappers

All wrappers re-export the `GanttEventMap` type from `apexgantt`. Wrapper handlers receive the detail, so type them with the detail type:

```ts
import type { GanttEventMap } from 'apexgantt';

function onDragged(detail: GanttEventMap['taskDragged']['detail']) {
  detail.daysMoved;   // fully typed
}
```

From apexgantt 3.20.0 the `taskUpdate` / `taskUpdateSuccess` details carry `source` (`'api' | 'dialog' | 'drag' | 'inline' | 'keyboard' | 'progress' | 'resize'`), and `taskUpdateSuccess` fires for drags too: do not save from both `onTaskDragged` and `onTaskUpdateSuccess`. See `references/events.md`.

## Web components (shadow DOM)

Since 3.19.0 the core API works inside a shadow root: pass an element inside it as the container. Styles, dropdowns and keyboard focus stay inside that root, and the `--apx-*` tokens set on the host page still reach the chart. The chart adds its own nodes next to whatever the container held when it was constructed, and re-renders and `destroy()` keep those children, so a component can keep its own markup in the same element (add it before constructing the chart).

```js
import { ApexGantt } from 'apexgantt';

class ProjectGantt extends HTMLElement {
  connectedCallback() {
    if (!this.el) {
      this.el = document.createElement('div');
      this.attachShadow({ mode: 'open' }).appendChild(this.el);
    }
    this.gantt = new ApexGantt(this.el, { series: this.tasks ?? [] });
    this.gantt.render();
  }
  disconnectedCallback() {
    this.gantt?.destroy();
  }
}
customElements.define('project-gantt', ProjectGantt);
```

## Common pitfalls in framework code

| ❌ | ✅ |
|---|---|
| Recreating the chart in every render without `destroy()` | Always return `destroy()` from `useEffect` cleanup / `onBeforeUnmount` / `ngOnDestroy` |
| Mutating `series` array reactively (`tasks.value.push(...)`) | Replace the reference: `tasks.value = [...tasks.value, newTask]` |
| Listening for camel-case events in templates (Vue) | Vue normalises kebab-case in templates: `@task-dragged` (not `@taskDragged`) |
| Missing `inputDateFormat` for ISO dates | Pass `inputDateFormat: 'YYYY-MM-DD'` in `options` whenever your data is ISO |
| `series={tasks}` / `:series` / `[series]` on a wrapper | The wrapper prop is `tasks`; other core options go in `options` |
| `(e) => e.detail.updatedTask` in a wrapper handler | Wrapper handlers receive the detail: `(detail) => detail.updatedTask` |
| Importing `apexgantt/ignite-ui` inside a late-mounting component | Import it once in the app entry, before the first chart renders (free, no licence key; apexgantt 3.21.0). Leave the `theme` prop and `options.theme` unset so a dark Ignite UI variant can apply; call `render()` on the ApexGantt instance after swapping the Ignite UI stylesheet |
| Relying on `vue-apexgantt` to leave `theme` unset | `vue-apexgantt` 1.2.0 and earlier default the `theme` prop to `'light'` and always send it, which pins light and blocks a dark Ignite UI variant. The next vue-apexgantt release (after 1.2.0) leaves `theme` unset when the prop is unset. Until then, bind `:theme` reactively to the page's current Ignite UI variant (update it where the app swaps the stylesheet) rather than to a fixed `"dark"`, which would not follow a light/dark switch |
