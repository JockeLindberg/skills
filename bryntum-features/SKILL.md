---
name: bryntum-features
description: >
  Find the built-in Bryntum feature for what the user is describing, BEFORE writing a custom
  renderer, custom CSS, or a hand-drawn overlay. Use alongside the `bryntum` skill whenever a
  request is phrased as an outcome rather than an API — "shade 11:00-13:00 as lunch", "grey out
  weekends", "working hours are 9 to 5", "mark today", "draw arrows between tasks", "group by
  city", "a totals row", "a search box", "export to Excel", "print", "undo", "swimlanes",
  "baselines", "critical path", "recurring events", "split a task", "WIP limit", "highlight
  that row". Most of these are one config. Read this before hand-rolling any of it.
metadata:
  tags: bryntum, features, scheduler, schedulerpro, gantt, grid, calendar, taskboard, timeranges, nonworkingtime, export, undo
---

## How to use this page

If a request matches a row below, use the named feature. Do **not** re-implement it with an
`eventRenderer`, a `cellCls`, an absolutely positioned `<div>`, or a CSS `::before`. The
rightmost column is what models usually write when they do not know the feature exists — if
you catch yourself writing that, stop and use the middle column instead.

Everything here was verified against Bryntum 7.3.6 source. Where a product differs from its
siblings, the row says so.

---

## Features in 30 seconds

Features are plugins configured **at construction**, on the `features` object:

```javascript
const scheduler = new Scheduler({
    features : {
        timeRanges     : true,                          // on, defaults
        nonWorkingTime : { showHeaderElements : false }, // on, configured
        eventTooltip   : false                          // explicitly off
    }
});
```

Four rules that cost agents the most time:

1. **The key is the class name with a lowercase first letter.** `GroupSummary` → `groupSummary`,
   `ResourceTimeRanges` → `resourceTimeRanges`, `CriticalPaths` → `criticalPaths`.
2. **Configure features at construction time.** A feature added to `features` after the
   component is built never runs its paint-time init. To turn one on later, it must already be
   in the config — then flip `widget.features.x.disabled = false`.
3. **"Registered on by default" ≠ "visible".** `criticalPaths` is registered on for Gantt but
   ships `disabled : true`; you still have to enable it explicitly.
4. **The same concept has different feature names per product.** Scheduler `eventEdit` vs Gantt
   `taskEdit`; Scheduler `eventTooltip.template` vs Calendar `eventTooltip.renderer`.

---

## Time bands, non-working time, calendars

| The user asks for | Use | Minimal snippet | Usually written instead |
|---|---|---|---|
| A shaded band across the whole timeline ("lunch 11:00–13:00", "maintenance window") | `timeRanges` feature + `project.timeRanges` | `features : { timeRanges : true }`, `project : { timeRanges : [{ startDate, endDate, name : 'Lunch', cls : 'lunch' }] }` | A custom `eventRenderer` plus absolutely positioned divs, or a background gradient on the row |
| A shaded band for **one resource only** | `resourceTimeRanges` feature + `project.resourceTimeRanges` | `features : { resourceTimeRanges : true }`, `project : { resourceTimeRanges : [{ resourceId : 'r1', startDate, endDate, name : 'PTO' }] }` | A fake zero-height event, or per-row CSS |
| A "now" line / marking today | `timeRanges` feature, `showCurrentTimeLine` | `features : { timeRanges : { showCurrentTimeLine : true } }` | A `setInterval` that repositions a div |
| Grey out weekends / outside working hours on the timeline | `nonWorkingTime` feature | `features : { nonWorkingTime : true }` | Iterating dates and injecting grey `<div>`s |
| "Working hours are 08:00–17:00, Mon–Fri" (Scheduler Pro / Gantt) | Project **calendars** — this is what drives scheduling *and* the grey shading | see snippet below | Grey boxes drawn on top, which look right but do not affect scheduling |
| Grey the non-working part **inside** an event bar | `eventNonWorkingTime` (Scheduler) | `features : { eventNonWorkingTime : true }` | A striped background on `.b-sch-event` |
| Grey non-working time **per resource row** (each resource has its own calendar) | `resourceNonWorkingTime` (Scheduler Pro) | `features : { resourceNonWorkingTime : true }` | Reusing `nonWorkingTime`, which only knows the project calendar |
| Grey non-working time in Gantt | `taskNonWorkingTime`, `mode : 'row'` or `'bar'` | `features : { taskNonWorkingTime : { mode : 'row' } }` | — |
| Restrict the hours a Calendar day/week view shows | `dayStartTime` / `dayEndTime`, or `workingTime` | `new Calendar({ dayStartTime : 8, dayEndTime : 18 })` | Scrolling the view with JS on render |
| Vertical lines at each tick on the timeline | `columnLines` (already on for Scheduler/Gantt) | `features : { columnLines : true }` | Repeating-linear-gradient background. Note: `columnLines` draws from the ViewPreset tick levels only — it **cannot** draw a line from a time range |

The four non-working-time features shade four different things — pick by *where* the grey goes:
the time axis (`nonWorkingTime`), inside the bar (`eventNonWorkingTime`), a resource's own row
(`resourceNonWorkingTime`, Pro), or a Gantt row/bar (`taskNonWorkingTime`).

**Working hours (Scheduler Pro / Gantt).** Non-working time is not decoration — it comes from the
project's calendar, and it is what makes the engine skip weekends when it schedules:

```javascript
const project = new ProjectModel({
    calendars : [
        {
            id                       : 'business',
            name                     : 'Business hours',
            unspecifiedTimeIsWorking : false,
            intervals                : [
                { recurrentStartDate : 'at 08:00',        recurrentEndDate : 'at 17:00',        isWorking : true  },
                { recurrentStartDate : 'on Sat at 00:00', recurrentEndDate : 'on Mon at 00:00', isWorking : false }
            ]
        }
    ],
    calendar : 'business'   // the project default; resources and events can override it
});
```

Use `calendars`, not the deprecated `calendarsData`. A resource or event gets its own calendar by
setting its `calendar` field to a calendar id.

**Time ranges as data.** A time range is a `TimeSpan`: `startDate` plus either `endDate` or
`duration` + `durationUnit`, and the optional presentation fields `name`, `cls`, `iconCls`,
`style`. A `ResourceTimeRangeModel` adds `resourceId` and `timeRangeColor`.

The store properties are **singular** — `project.timeRangeStore` and
`project.resourceTimeRangeStore`. There is no `timeRangesStore`. The inline data configs are
plural (`timeRanges`, `resourceTimeRanges`); `timeRangesData` / `resourceTimeRangesData` are
deprecated as of 6.3.0.

---

## Highlighting, marking, selecting

| The user asks for | Use | Minimal snippet | Usually written instead |
|---|---|---|---|
| Highlight a span from code, temporarily ("show me the free slot") | `timeSpanHighlight` feature (Scheduler Pro / Gantt) | `features : { timeSpanHighlight : true }` then `scheduler.features.timeSpanHighlight.highlightTimeSpan({ startDate, endDate })`; clear with `unhighlightTimeSpans()` | Adding and removing a time range record |
| Highlight one **row** | The `cls` field on the record | `record.cls = 'is-late'` | A `rowRenderer` that rewrites DOM |
| Highlight one **column's cells** | Column `cellCls` (static) or `renderer` setting `cellElement.classList` | `{ field : 'total', cellCls : 'is-total' }` | A `document.querySelectorAll` sweep after render |
| Alternating row colours | `stripe` feature | `features : { stripe : true }` | `:nth-child(even)` CSS, which breaks under row virtualization |
| Let the user drag-select a time span in the header | `timeSelection` feature | `features : { timeSelection : true }` | A custom mousedown/mousemove handler |
| Read what is selected | `grid.selectedRecords` / `selectedRecord` / `selectedCells` | `const rows = grid.selectedRecords;` | `getSelectionModel().getSelection()` (that is Ext JS — see `bryntum-from-extjs`) |

---

## Structure: dependencies, trees, nesting, segments

| The user asks for | Use | Minimal snippet | Usually written instead |
|---|---|---|---|
| Arrows between events / task links | `dependencies` feature. **Off** by default in Scheduler, **on** in Scheduler Pro and Gantt | `features : { dependencies : true }`, `project : { dependencies : [{ from : 1, to : 2, type : 2 }] }` | SVG lines drawn by hand over the timeline |
| Let the user draw and edit a dependency | `dependencies` + `dependencyEdit` (off by default) | `features : { dependencies : true, dependencyEdit : true }` | A custom drag handler on the bar terminals |
| Critical path | `criticalPaths` (Gantt). Registered on, but **starts disabled** | `gantt.features.criticalPaths.disabled = false;` | A hand-written longest-path walk over the task tree |
| A tree / hierarchy in a grid | `TreeGrid`, or `Grid` + `tree` feature, with exactly one `{ type : 'tree' }` column and `children` arrays in the data | `new TreeGrid({ columns : [{ type : 'tree', field : 'name' }] })` | Flat rows plus manual indentation in a renderer |
| Turn flat rows into a tree grouped by field | `treeGroup` feature (one tree level per field) | `features : { treeGroup : { levels : ['country', 'city'] } }` | Pre-processing the data into nested arrays by hand |
| Expand / collapse from code | `tree` feature methods | `await grid.features.tree.expandAll()` / `.collapseAll()` / `.expandTo(record)` | Toggling a `collapsed` field and refreshing |
| Events nested **inside** a parent event bar (Scheduler Pro) | `nestedEvents` feature (off by default). Nesting is a `children` array on the event, not a `parentId` | `features : { nestedEvents : true }` | Two stacked schedulers |
| Split a task/event into segments with a gap | `eventSegments` (Pro, on by default) + the model API | `await eventRecord.splitToSegments(splitDate, 2, 'day')`; also `setSegments()`, `mergeSegments()` | Deleting the event and creating two |
| Split the **view** into side-by-side panes | `split` feature — a different thing entirely | `features : { split : true }` | — (this row exists so `split` is not mistaken for event splitting) |
| Drag items from a grid/sidebar onto the timeline | Load the **`bryntum-drag-and-drop`** skill — `DragHelper`, `dropTargetSelector`, `scheduleEvent()` | see that skill | HTML5 drag-and-drop events |

**Dependency data.** The persisted fields are `from` and `to` (`fromEvent` / `toEvent` are
resolved accessors, not what you put in JSON). `type` is an integer:

| `type` | Meaning |
|---|---|
| `0` | StartToStart |
| `1` | StartToEnd |
| `2` | **EndToStart — the default** |
| `3` | EndToEnd |

---

## Grouping, totals, filtering, searching, sorting

| The user asks for | Use | Minimal snippet | Usually written instead |
|---|---|---|---|
| Group rows under headers | `group` feature (on by default for Grid and Scheduler, off for TreeGrid) | `features : { group : 'city' }` or `{ group : { field : 'city', ascending : false } }` | Sorting, then injecting fake header rows |
| A totals row at the bottom | `summary` feature (**off** by default) + `sum` on the column | `features : { summary : true }`, column `{ field : 'score', sum : 'sum' }` | A `<div>` under the grid computed in JS |
| A total per group | `groupSummary` feature (off by default) + the same column `sum` | `features : { group : 'city', groupSummary : true }` | — |
| Several totals in one column | Column `summaries` array (replaces `sum` + `summaryRenderer`) | `{ field : 'score', summaries : [{ sum : 'sum' }, { sum : 'average' }] }` | Two duplicate columns |
| Format the total | Column `summaryRenderer` | `summaryRenderer : ({ sum }) => \`Total: ${sum}\`` | Post-processing the DOM |
| Filter UI in the header | `filter` (menu + popup) or `filterBar` (an inline row of fields) — both off by default | `features : { filterBar : true }` | A custom toolbar of inputs calling `store.filter` |
| Filter from code | `store.filter(...)`, undone with `store.clearFilters()` | `grid.store.filter({ property : 'city', value : 'Paris', operator : '=' })` | Replacing `store.data` with a filtered array, which loses the original rows |
| A search box over all columns | `search` feature — it has the logic but **no UI**; you supply the field | `features : { search : true }` then `grid.features.search.search('paris')` | A regex scan over `store.records` in a keyup handler |
| Type-to-find inside the focused column | `quickFind` feature | `features : { quickFind : true }` | — |
| Sort by more than one column | `store.sort(field, ascending, /* add */ true)`, or an array of sorters | `grid.store.sort('city'); grid.store.sort('name', true, true);` | `records.sort()` on a copy |

Grouping by more than one field is not supported by the grid UI — the multi-grouper store API is
marked internal. For a multi-level breakdown use `treeGroup` instead.

---

## Editing, undo, tooltips

| The user asks for | Use | Minimal snippet | Usually written instead |
|---|---|---|---|
| An editor popup for an event / task | `eventEdit` (Scheduler, on) / `taskEdit` (Scheduler Pro + Gantt, on) | `features : { eventEdit : true }` | A hand-built modal wired to `eventRecord.set()` |
| Add a field to that editor | `features.taskEdit.items.<tab>.items.<ref>` — object notation, not arrays | `features : { taskEdit : { items : { generalTab : { items : { risk : { type : 'combo', label : 'Risk', name : 'risk', items : ['Low','High'] } } } } } }` | Replacing the whole editor |
| Remove a field or tab | Set its ref to `false` | `items : { notesTab : false }` | CSS `display : none` |
| A lighter inline event editor (just a name field) | `simpleEventEdit` (Scheduler, off) | `features : { simpleEventEdit : true }` | — |
| Hover tooltip on an event bar | `eventTooltip`. Scheduler/Gantt customize with **`template`**; Calendar customizes with **`renderer`** / `titleRenderer` | Scheduler: `features : { eventTooltip : { template : ({ eventRecord }) => eventRecord.note } }` | `title=""` attributes, or a third-party tooltip library |
| Tooltip on a grid cell | `cellTooltip` feature (off) | `features : { cellTooltip : { tooltipRenderer : ({ record }) => record.name } }` | — |
| Undo / redo | The project's `StateTrackingManager`. It is **disabled until enabled** | `project : { stm : { autoRecord : true } }` then `project.stm.undo()` / `.redo()` | Snapshotting `store.json` into an array on every change |
| Undo / redo **buttons** | The `UndoRedo` widget — it exists for **Scheduler and TaskBoard only**, under different type names | Scheduler/Gantt: `tbar : [{ type : 'undoredo' }]`; TaskBoard: `{ type : 'taskboardundoredo' }` | — |

Gantt's editor tabs are `generalTab`, `predecessorsTab`, `successorsTab`, `resourcesTab`,
`advancedTab`, `notesTab`. For deeper editor work, load the `bryntum-editor` skill.

Undo/redo for a plain Grid store is possible but manual — there is no Grid `UndoRedo` widget, and
the `stm` shortcut lives on the project, which a plain Grid does not have:

```javascript
import { StateTrackingManager } from '@bryntum/grid';

const stm = new StateTrackingManager({ autoRecord : true });
stm.addStore(grid.store);
stm.enable();          // nothing is recorded until this is called
// later: stm.undo(); stm.redo();
```

---

## Getting data out

| The user asks for | Use | Minimal snippet | Gotcha |
|---|---|---|---|
| Export to Excel | `excelExporter` feature (**experimental**, off) | `features : { excelExporter : { xlsProvider : writeXlsxFile } }` then `grid.features.excelExporter.export({ filename : 'data' })` | Needs the external `write-excel-file` npm package supplied as `xlsProvider` — it throws without one |
| Export to PDF | `pdfExport` feature (off) | `features : { pdfExport : { exportServer : 'http://localhost:8080' } }` then `await grid.features.pdfExport.export()` | **Requires a running export server** (the separate `bryntum/pdf-export-server`). There is no client-only PDF path |
| Print | `print` feature (off), then call it **on the component** | `features : { print : true }` then `grid.print()` | It is `grid.print()`, **not** `grid.features.print.print()`. No server needed |
| Export a Gantt to MS Project | `mspExport` feature (Gantt, off) | `features : { mspExport : true }` then `gantt.features.mspExport.export()` | Produces MS Project XML; no server needed |
| Print a Calendar | Calendar's own `print` feature | `features : { print : true }` then `calendar.print()` | Calendar has no dedicated PDF-export feature; printing drives the browser dialog |

Do not build "export" by serializing `store.records` to CSV unless the user actually asked for
CSV — Excel, PDF and print are all real features that preserve columns, grouping and styling.

---

## Gantt

| The user asks for | Use | Minimal snippet | Usually written instead |
|---|---|---|---|
| Baselines / "planned vs actual" | `baselines` feature (off) + the task's `baselines` store field | `features : { baselines : true }`, task `{ baselines : [{ startDate : '2026-01-01', endDate : '2026-01-05' }] }` | A second ghost task per row |
| Capture the current dates as a baseline | `setBaseline(version)` — **1-based**, and `TaskStore` has its own | `gantt.taskStore.setBaseline(1)` | Copying dates into custom fields |
| A progress / status line | `progressLine` feature (off; needs `percentBar`, which is on for Gantt) | `features : { progressLine : { statusDate : new Date() } }` | An SVG polyline drawn over the chart |
| Child bars summarized on the parent bar | `rollups` feature (off) + `rollup : true` on the tasks | `features : { rollups : true }` | A custom parent renderer |
| Deadline / constraint markers | `indicators` feature (off) — draws `earlyDates`, `lateDates`, `constraintDate`, `deadlineDate` | `features : { indicators : { items : { deadlineDate : true, earlyDates : false } } }` | Icons injected by a `taskRenderer` |
| A percent-done bar inside the task bar | `percentBar` (already on for Gantt) + the `percentDone` field | `task.percentDone = 40` | A nested div sized in a renderer |
| Project start/end marker lines | `projectLines` (already on) | `features : { projectLines : { showStatusDate : true } }` | — |
| Shade the span of a parent's children | `parentArea` feature (off) | `features : { parentArea : true }` | — |
| Forward vs backward (ALAP) scheduling | `project.direction : 'Forward' \| 'Backward'` | `project : { direction : 'Backward' }` | Reversing the dependency list |
| Save / compare project versions | `versions` feature (off, Gantt + Scheduler Pro) | `features : { versions : true }` | — |

---

## Calendar

| The user asks for | Use | Minimal snippet | Notes |
|---|---|---|---|
| Day / week / month / year / agenda views | `mode` + `modes` | `new Calendar({ mode : 'month' })` | `day`, `week`, `month`, `year`, `agenda` are present by default; `week` is the default `mode` |
| Extra view types | Add them to `modes` | `modes : { list : true, resource : true }` | Also available: `list`, `resource`, `dayresource`, `dayagenda`, `monthagenda`, `monthgrid`, `yearplanner` |
| Remove a view | Null it out | `modes : { year : null }` | |
| Recurring events | The `recurrenceRule` field (an RRULE string) + `exceptionDates` | `{ name : 'Standup', startDate : '2026-01-05T09:00', duration : 15, durationUnit : 'minute', recurrenceRule : 'FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR' }` | **Not Calendar-only** — `recurrenceRule` lives on the shared `TimeSpan` mixin, so Scheduler, Scheduler Pro and time ranges support it too |
| Hide weekends / set the first day of the week | `nonWorkingDays`, `hideNonWorkingDays`, `weekStartDay` | `new Calendar({ hideNonWorkingDays : true, weekStartDay : 1 })` | |
| A resource filter in the sidebar | The Sidebar's `resourceFilter` config | `sidebar : { resourceFilter : { … } }` | |

Calendar has **no dependencies** — do not add a `dependencies` prop to it.

Calendar working hours are a different mechanism from Scheduler Pro / Gantt: it is
`workingTime : { fromHour, toHour, fromDay, toDay }` (or `dayStartTime` / `dayEndTime`
directly), *not* a `CalendarModel` with intervals.

---

## TaskBoard

| The user asks for | Use | Minimal snippet | Notes |
|---|---|---|---|
| Swimlanes | `swimlaneField` + `swimlanes` | `new TaskBoard({ swimlaneField : 'prio', swimlanes : ['high', 'low'] })` | Or `autoGenerateSwimlanes : true` to derive them from the data |
| **A WIP limit per column** | **There is no such config.** See below | — | Nothing in TaskBoard caps the number of cards in a column |
| Control what appears on a card | `headerItems` / `bodyItems` / `footerItems`. The **key is the field name**, `type` names the item type | `bodyItems : { prio : { type : 'text' }, tags : { type : 'tags', field : 'labels' } }` | Your items are merged with the defaults (`headerItems.text`→`name`, `bodyItems.text`→`description`, `footerItems.resourceAvatars`). Item types: `text`, `template`, `tags`, `progress`, `rating`, `image`, `chart`, `todoList`, `resourceAvatars`, `taskMenu`, `separator`, `collapse`, `jsx`. Remove one by setting it to `null` |
| Cards side by side within a column | `tasksPerRow` on the board, column or swimlane | `columns : [{ id : 'doing', text : 'Doing', tasksPerRow : 2 }]` | A **layout** setting, not a capacity limit |
| Filter cards | `columnFilter` (per-column, honours `ColumnModel.filterable`) or `filterBar` (quick search, honours `searchable`) | `features : { filterBar : true }` | There is no `taskFilter` feature |
| Lock a column to one edge | `locked : 'start' \| 'end'` on the column | `{ id : 'done', text : 'Done', locked : 'end' }` | |

**WIP limits.** TaskBoard has no `wipLimit`, `maxTasks`, `capacity` or equivalent. `tasksPerRow`
is how many cards sit side by side, not how many a column may hold. If the user wants a hard WIP
limit, you have to build it — veto the drop and render the count yourself:

```javascript
const taskBoard = new TaskBoard({
    columns  : [{ id : 'doing', text : 'Doing (max 3)' }],
    listeners : {
        beforeTaskDrop({ targetColumn, taskRecords }) {
            if (targetColumn.id === 'doing' && targetColumn.tasks.length + taskRecords.length > 3) {
                return false;   // veto the drop
            }
        }
    }
});
```

Say plainly that this is a custom limit, not a product feature.

---

## Resource load and utilisation (Scheduler Pro)

| The user asks for | Use | Notes |
|---|---|---|
| A histogram of how loaded each resource is | The `ResourceHistogram` view | A view class, not a feature. Give it the same `project` as the scheduler, or link the two with `partner` so they scroll together |
| A grid of assignments per resource per period | The `ResourceUtilization` view | Same wiring |

---

## Things that do not exist

Do not reach for these — they are the most common invented APIs in this area:

- **TaskBoard WIP limits.** No config. Build it with `beforeTaskDrop` (above).
- **`timeRangesStore` / `resourceTimeRangesStore`.** The store properties are singular:
  `timeRangeStore`, `resourceTimeRangeStore`.
- **A `taskFilter` TaskBoard feature.** It is `columnFilter` or `filterBar`.
- **`Grid#refresh()`.** Grid has `refreshRows()` and `refreshColumn()`. Calendar, TaskBoard,
  Scheduler and Gantt do have `refresh()`.
- **Multi-field grouping in the grid UI.** Use `treeGroup` for a multi-level breakdown.
- **A Grid `UndoRedo` widget.** The widget exists for Scheduler and TaskBoard only.
- **A client-only PDF export.** `pdfExport` needs the export server.
- **`columnLines` fed from time ranges.** It draws from the ViewPreset ticks only.

---

## Related

- Dragging onto the timeline from a grid, a sidebar, or a non-Bryntum source: `bryntum-drag-and-drop`.
- Customizing the event/task editor beyond adding a field: `bryntum-editor`.
- Event bar content, `eventRenderer`, and widget `rendition`: `bryntum-styling`.
- Themes and dark mode: `bryntum-theming`.
- Ext JS idioms that do not apply here: `bryntum-from-extjs`.
