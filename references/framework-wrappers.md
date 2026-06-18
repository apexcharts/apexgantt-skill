# Framework Wrappers — React, Vue, Angular

ApexCharts ships first-party wrappers for each major framework. They handle ref management, prop diffing, and call `destroy()` automatically on unmount.

## React — `react-apexgantt`

```bash
npm install react-apexgantt apexgantt
```

```jsx
import { useState } from 'react';
import Gantt, { GanttEvents } from 'react-apexgantt';

function ProjectTimeline() {
  const [tasks, setTasks] = useState([
    { id: 't1', name: 'Design', startTime: '2026-01-01', endTime: '2026-01-15', progress: 60 },
    { id: 't2', name: 'Develop', startTime: '2026-01-16', endTime: '2026-02-15', progress: 25, dependency: 't1' },
  ]);

  return (
    <Gantt
      series={tasks}
      inputDateFormat="YYYY-MM-DD"
      pixelsPerDay={25.7}
      enableSelection
      onTaskUpdateSuccess={(e) => console.log('saved', e.detail.updatedTask)}
      onTaskDragged={(e) => console.log('moved', e.detail.daysMoved)}
    />
  );
}
```

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

## Vue 3 — `vue-apexgantt`

```bash
npm install vue-apexgantt apexgantt
```

```vue
<script setup>
import { ref } from 'vue';
import VueGantt from 'vue-apexgantt';

const tasks = ref([
  { id: 't1', name: 'Design',  startTime: '2026-01-01', endTime: '2026-01-15', progress: 60 },
  { id: 't2', name: 'Develop', startTime: '2026-01-16', endTime: '2026-02-15', progress: 25, dependency: 't1' },
]);

function onTaskDragged(e) {
  console.log('moved', e.detail.daysMoved);
}
</script>

<template>
  <VueGantt
    :series="tasks"
    input-date-format="YYYY-MM-DD"
    :pixels-per-day="25.7"
    enable-selection
    @task-update-success="(e) => console.log('saved', e.detail.updatedTask)"
    @task-dragged="onTaskDragged"
  />
</template>
```

## Angular — `ngx-apexgantt`

```bash
npm install ngx-apexgantt apexgantt
```

```ts
// app.module.ts
import { NgxApexgantt } from 'ngx-apexgantt';

@NgModule({
  imports: [NgxApexgantt],
})
export class AppModule {}
```

```ts
// project-timeline.component.ts
import { Component } from '@angular/core';
import type { TaskInput, TaskDraggedEventDetail } from 'apexgantt';

@Component({
  selector: 'project-timeline',
  template: `
    <ngx-apexgantt
      [series]="tasks"
      inputDateFormat="YYYY-MM-DD"
      [pixelsPerDay]="25.7"
      [enableSelection]="true"
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

  onSaved(detail: { taskId: string; updatedTask: any }) {
    console.log('saved', detail.updatedTask);
  }

  onDragged(detail: TaskDraggedEventDetail) {
    console.log('moved', detail.daysMoved);
  }
}
```

## Typed events in wrappers

All wrappers re-export the `GanttEventMap` type from `apexgantt`. Use it for fully-typed event handlers when bridging to your app's event system:

```ts
import type { GanttEventMap } from 'apexgantt';

function onDragged(e: GanttEventMap['taskDragged']) {
  // e.detail is fully typed
  e.detail.daysMoved;
}
```

## Common pitfalls in framework code

| ❌ | ✅ |
|---|---|
| Recreating the chart in every render without `destroy()` | Always return `destroy()` from `useEffect` cleanup / `onBeforeUnmount` / `ngOnDestroy` |
| Mutating `series` array reactively (`tasks.value.push(...)`) | Replace the reference: `tasks.value = [...tasks.value, newTask]` |
| Listening for camel-case events in templates (Vue) | Vue normalises kebab-case in templates: `@task-dragged` (not `@taskDragged`) |
| Missing `inputDateFormat` for ISO dates | Pass `inputDateFormat="YYYY-MM-DD"` whenever your data is ISO |
