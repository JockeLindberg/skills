---
name: bryntum-editor
description: >
  Customize Bryntum's built-in event/task editor — the popup shown when you create or
  double-click an event/task — across any product (Scheduler, Scheduler Pro, Gantt, Calendar,
  TaskBoard). Use alongside the `bryntum` skill whenever the user wants to add, remove, or
  reorder editor fields or tabs, tweak validation, react to the editor opening/saving, or
  replace the popup with a fully custom dialog. Trigger on phrases like "edit event popup",
  "task editor", "remove the % Complete field", "add a field to the editor", "custom event
  editor", "eventEdit"/"taskEdit" config, or "beforeEventEdit"/"beforeTaskEdit".
metadata:
  tags: bryntum, editor, eventedit, taskedit, popup, scheduler, gantt
---

## Keep the built-in editor

**Default to Bryntum's built-in editor.** It already handles create/edit/delete, validation, and writes straight back to the stores. Do NOT build a custom dialog (a React/MUI modal, a framework dialog, hand-rolled HTML) unless the user explicitly asks for one.

To match a design-system host app (e.g. Material UI), switch the Bryntum **theme** (`material3-light` / `material3-dark`) so the built-in editor and bars match — see the `bryntum-theming` skill. That is not a reason to replace the editor.

---

## Which feature owns the editor

| Product | Editor feature |
|---|---|
| Scheduler | `eventEdit` |
| Scheduler Pro | `taskEdit` |
| Gantt | `taskEdit` |
| Task Board | `taskEdit` |
| Calendar | `eventEdit` |

---

## Customize via the feature's `items` config

Remove fields you don't want or add your own through the feature's `items`. Use **object notation** (not arrays) for built-in items — `name` binds a field to a data field:

```js
features: {
    taskEdit: {                                   // eventEdit for Scheduler
        items: {
            generalTab: {
                items: {
                    percentDoneField : false,     // remove a built-in field
                    durationField    : false,
                    myField : { type: 'textfield', name: 'color', label: 'Color' } // add one
                }
            },
            predecessorsTab : false,              // remove whole tabs
            successorsTab   : false,
            advancedTab     : false
        }
    }
}
```

Run-time tweaks: listen to `beforeTaskEditShow` (or `beforeEventEditShow`) and adjust `editor.widgetMap.<ref>`.

---

## Replacing with a fully custom dialog (only when explicitly asked)

Keep the feature **ENABLED** and return `false` from `beforeEventEdit` / `beforeTaskEdit` to suppress the built-in popup.

**Do NOT set the feature to `false`** — that removes the hook entirely (silent no-op). Write changes back via `eventRecord.set({...})` so the stores stay in sync.

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
        home = $('#home').val(),
        away = $('#away').val(),
        date = $('#startDate').val();
        // ...

    // Update record — keeps the stores in sync
    editingRecord.set({
        startDate : DateHelper.parse(date, 'YYYY-MM-DD'),
        resources : [away, home]
    });
});
```

In a framework, instantiate/show your dialog from the listener and commit on save the same way — the `beforeEventEdit`/`beforeTaskEdit` → `record.set()` contract is identical.

---

## Checklist

- [ ] Customize via the editor feature's `items` (object notation) before considering a custom dialog
- [ ] Right feature for the product: `eventEdit` (Scheduler / Calendar) vs `taskEdit` (Scheduler Pro / Gantt / Task Board)
- [ ] Custom dialog: feature stays ENABLED, `beforeEventEdit`/`beforeTaskEdit` returns `false`, writes back via `record.set()`
- [ ] Never set the feature to `false` to get a custom dialog — that kills the hook
