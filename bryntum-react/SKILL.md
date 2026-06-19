---
name: bryntum-react
description: >
  React-specific patterns for Bryntum components. Use this alongside the `bryntum` skill when
  building or integrating any Bryntum product (Scheduler, Gantt, Calendar, Grid, TaskBoard,
  Scheduler Pro) in a React project. Trigger when the project has React in package.json or when
  the user asks about React + Bryntum integration, JSX config, StrictMode issues, or
  @bryntum/*-react packages.
metadata:
  tags: bryntum, react, strictmode
---

## Quick-start guide

Fetch before writing code:
`https://bryntum.com/products/{product}/docs-llm/guide/{Product}/quick-start/react.md`

---

## React

Framework wrapper package: `@bryntum/{product}-react`

### Component

```jsx
import { BryntumGantt } from '@bryntum/gantt-react';

const App = () => {
    const [ganttProps] = useState(useGanttProps());
    return <BryntumGantt {...ganttProps} />;
};
```

Pass all config as JSX props. Use `useRef` for instance access:

```jsx
const ganttRef = useRef(null);
// access instance: ganttRef.current.instance
<BryntumGantt ref={ganttRef} {...ganttProps} />
```

### StrictMode (React 18+)

React StrictMode double-mounts in dev (mount → unmount → mount). Use `useState` for config — it preserves the config object across the remount cycle, avoiding side effects that need cleanup:

```javascript
const App = () => {
    const [ganttProps] = useState(useGanttProps());
    return <BryntumGantt {...ganttProps} />;
};

createRoot(document.getElementById('root')!).render(
    <StrictMode><App /></StrictMode>
);
```

**Never retain references to destroyed Bryntum instances after unmount.** Do not use `useEffect`/`useRef` patterns that hold the instance across the unmount cycle.

### Sizing

`html`, `body`, and `#root` all need an explicit height:

```css
html, body, #root { height: 100%; margin: 0; }
```

### Vite config

Include Bryntum packages in `optimizeDeps` to prevent multiple bundle loading in dev:

```js
optimizeDeps: {
    include: ['@bryntum/gantt', '@bryntum/gantt-react']
}
```

When adding to an **existing** `vite.config`, merge into the current `optimizeDeps.include` array — don't only check for this on fresh scaffolds, and don't overwrite an existing `optimizeDeps`.

---

## Event / Task editing — keep the built-in editor

**Default to Bryntum's built-in editor.** It already handles create/edit/delete, validation, and writes straight back to the stores. Do NOT build a custom React/MUI dialog unless the user explicitly asks for one.

To match a Material UI host app, use Bryntum's **`material3-light` / `material3-dark`** theme so the built-in editor and bars match MUI styling — not a reason to replace the editor.

**Customize the built-in editor via the feature's `items` config** (Scheduler Pro / Gantt = `taskEdit`; Scheduler = `eventEdit`). This is how you remove fields you don't want (e.g. the Gantt-only `% Complete`, `Duration`, Predecessors/Advanced tabs) or add your own:

```jsx
features: {
    taskEdit: {
        items: {
            generalTab: {
                items: {
                    percentDoneField : false,         // remove a built-in field
                    durationField    : false,
                    myField : { type: 'textfield', name: 'color', label: 'Color' } // add one; `name` binds to the data field
                }
            },
            predecessorsTab : false,                  // remove whole tabs
            successorsTab   : false,
            advancedTab     : false
        }
    }
}
```
Use object notation (not arrays) for built-in items. Run-time tweaks: listen to `beforeTaskEditShow` and adjust `editor.widgetMap.<ref>`.

**Only if a fully custom dialog is explicitly requested:** keep the feature ENABLED and return `false` from `beforeEventEdit`/`beforeTaskEdit` to suppress the built-in popup 

— do NOT set the feature to `false`, which removes the hook entirely (silent no-op). Write changes back via `eventRecord.set({...})` so the stores stay in sync.

Bootstrap example: 

```js
let editingRecord = null;

const scheduler = new Scheduler({
    listeners : {
        beforeEventEdit({ eventRecord }) {
            // Show custom editor
            $('#customEditor').modal('show');

            // Fill its fields
            $('#home').val(eventRecord.resources[0].id);
            $('#away').val(eventRecord.resources[1].id);
            $('#startDate').val(DateHelper.format(eventRecord.startDate, 'YYYY-MM-DD'));
            // ...

            editingRecord = eventRecord;

            // Prevent built-in editor
            return false;
        }
    }
});

// When clicking save in the custom editor
$('#save').on('click', () => {
    const
        // Extract teams
        home      = $('#home').val(),
        away      = $('#away').val(),
        // Extract date
        date      = $('#startDate').val();
        // ...

    // Update record
    editingRecord.set({
        startDate : DateHelper.parse(date, 'YYYY-MM-DD'),
        resources : [away, home]
    });
});
```
