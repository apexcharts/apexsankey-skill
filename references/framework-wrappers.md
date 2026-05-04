# Framework Wrappers — React, Vue, Angular

## React — `react-apexsankey`

```bash
npm install react-apexsankey apexsankey
```

```jsx
import Sankey from 'react-apexsankey';

const data = {
  nodes: [
    { id: 'a', title: 'Source A' },
    { id: 'b', title: 'Source B' },
    { id: 'c', title: 'Hub' },
    { id: 'd', title: 'Sink' },
  ],
  edges: [
    { source: 'a', target: 'c', value: 10, type: 'flow' },
    { source: 'b', target: 'c', value:  6, type: 'flow' },
    { source: 'c', target: 'd', value: 16, type: 'flow' },
  ],
};

export default function FlowChart() {
  return (
    <Sankey
      data={data}
      width={800}
      height={500}
      nodeWidth={24}
      spacing={60}
      onNodeClick={(node) => console.log(node)}
    />
  );
}
```

Vanilla pattern (when you need direct chart access):

```jsx
import { useEffect, useRef } from 'react';
import { ApexSankey } from 'apexsankey';

function Sankey({ data, options }) {
  const ref = useRef(null);

  useEffect(() => {
    const sankey = new ApexSankey(ref.current, options);
    sankey.render({ ...data, options: sankey.options });
    return () => sankey.destroy();
  }, [data, options]);

  return <div ref={ref} />;
}
```

## Vue 3 — `vue-apexsankey`

```bash
npm install vue-apexsankey apexsankey
```

```vue
<script setup>
import { ref } from 'vue';
import VueSankey from 'vue-apexsankey';

const data = ref({
  nodes: [
    { id: 'a', title: 'Source A' },
    { id: 'b', title: 'Source B' },
    { id: 'c', title: 'Hub' },
    { id: 'd', title: 'Sink' },
  ],
  edges: [
    { source: 'a', target: 'c', value: 10, type: 'flow' },
    { source: 'b', target: 'c', value:  6, type: 'flow' },
    { source: 'c', target: 'd', value: 16, type: 'flow' },
  ],
});
</script>

<template>
  <VueSankey
    :data="data"
    :width="800"
    :height="500"
    :node-width="24"
    :spacing="60"
  />
</template>
```

## Angular — `ngx-apexsankey`

```bash
npm install ngx-apexsankey apexsankey
```

```ts
import { Component } from '@angular/core';
import { NgxApexsankey } from 'ngx-apexsankey';

@Component({
  selector: 'app-flow',
  standalone: true,
  imports: [NgxApexsankey],
  template: `
    <ngx-apexsankey
      [data]="data"
      [width]="800"
      [height]="500"
      [nodeWidth]="24"
      [spacing]="60">
    </ngx-apexsankey>
  `,
})
export class FlowComponent {
  data = {
    nodes: [
      { id: 'a', title: 'Source A' },
      { id: 'b', title: 'Source B' },
      { id: 'c', title: 'Hub' },
      { id: 'd', title: 'Sink' },
    ],
    edges: [
      { source: 'a', target: 'c', value: 10, type: 'flow' },
      { source: 'b', target: 'c', value:  6, type: 'flow' },
      { source: 'c', target: 'd', value: 16, type: 'flow' },
    ],
  };
}
```

## Common pitfalls in framework code

| ❌ | ✅ |
|---|---|
| Recreating chart in every render without `destroy()` | Always return `destroy()` from `useEffect` cleanup / `onBeforeUnmount` / `ngOnDestroy` |
| Mutating `nodes` / `edges` arrays in place | Replace the reference: `data.value = { ...data.value, edges: [...] }` |
| Skipping `options: sankey.options` in render payload (vanilla pattern) | Always include it: `sankey.render({ ...data, options: sankey.options })` |
| Calling `setLicense` per-component | Call `ApexSankey.setLicense(KEY)` exactly once at app startup |
