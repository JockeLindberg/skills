---
name: bryntum-from-extjs
description: >
  Bryntum's API shares vocabulary with Ext JS (Store, Model, Container, items, tbar, type
  aliases, listeners, cls, flex, renderer) but is a different library. Use alongside the
  `bryntum` skill whenever Ext JS habits could leak in — `store.first()`, `component.down()`,
  `store.get(id)`, `Ext.getCmp`, `Ext.create`, `Ext.define`, `xtype`, `dataIndex`, `renderTo`,
  `Ext.Msg`, `Ext.Ajax`, `grid.getStore()`, `getSelectionModel()`, `store.sync()`, or when the
  user mentions Ext JS, ExtJS, Sencha, `Sch.panel.SchedulerGrid` or `Gnt.panel.Gantt`. Read
  this before writing any of those into Bryntum code.
metadata:
  tags: bryntum, extjs, sencha, migration, store, widget, api-differences
---

## Read this first

Bryntum used to ship **Ext Scheduler** (`Sch.panel.SchedulerGrid`) and **Ext Gantt**
(`Gnt.panel.Gantt`) as add-ons for Ext JS. Those products were discontinued around 2019. The
current products — Scheduler, Scheduler Pro, Gantt, Grid, Calendar, TaskBoard — are standalone
ES modules with **no Ext JS dependency and no shared code**. Blog posts, Stack Overflow answers
and Sencha forum threads under the `Sch.*` / `Gnt.*` namespaces do not apply. If you find
yourself writing `Ext.` anything, or a class under `Sch.` or `Gnt.`, you are on the wrong track.

The danger is not the obvious stuff — nobody writes `Ext.create` into a Bryntum file by
accident. The danger is that both libraries have a `Store` with `first`, a `Container` with
`items`, a column with a `renderer`, and a `type` that looks like `xtype`. The near-misses
below fail **silently or with a confusing error**, not with "Ext is not defined".

Everything on this page was verified against Bryntum 7.3.6 source.

---

## The five that bite hardest

| Ext JS | Bryntum | Why it hurts |
|---|---|---|
| `store.first()` | `store.first` | It is a **getter**. Calling it throws `store.first is not a function` |
| `container.down('#save')` | `container.widgetMap.save` | **There is no `down()`.** `up()` exists, which makes its absence look like a typo |
| `store.find('name', 'Bob')` | `store.findRecord('name', 'Bob')` | Bryntum's `find(fn)` takes a **function**. Passing a field name returns `undefined` — **no error**, just a silently wrong answer |
| `store.get(5)` | `store.getById(5)` | There is no `store.get()` |
| `Toast.show({ type : 'success' })` | `Toast.show('Saved')` | `type` is the **registered widget type**. Setting it throws `Can not mutate 'type' config to different value` |

---

## Construction and the class system

| Ext JS | Bryntum | Note |
|---|---|---|
| `Ext.define('My.Grid', { extend : 'Ext.grid.Panel' })` | `class MyGrid extends Grid { static $name = 'MyGrid'; }` | Plain ES classes. `$name` is used for the feature/type registry, not for a namespace |
| `Ext.create('Ext.grid.Panel', cfg)` | `new Grid(cfg)` | No string-based factory for top-level construction |
| `xtype : 'button'` | `type : 'button'` | Same idea, different key. `type` values are registered via `static type = '…'` on the class |
| `requires : [...]` / `Ext.require` | `import { Grid } from '@bryntum/grid'` | ES module imports |
| `renderTo : 'app'` | `appendTo : 'app'` | Also `insertBefore`, `insertFirst`, `adopt`. There is no `renderTo` |
| `initComponent()` | `static configurable = { … }` plus `changeX()` / `updateX()` hooks | Bryntum's config system calls `changeFoo(value, old)` to normalize and `updateFoo(value, old)` to react |
| `Ext.data.Model` `fields : [...]` | `class MyModel extends Model { static fields = ['name', { name : 'due', type : 'date' }]; }` | |
| `config : { … }` + generated `getFoo()`/`setFoo()` | `static configurable = { … }` + plain property access (`widget.foo = 1`) | No generated getter/setter method pairs |

---

## Finding components

| Ext JS | Bryntum | Note |
|---|---|---|
| `Ext.getCmp('myGrid')` | `Widget.getById('myGrid')` | |
| `Ext.ComponentQuery.query('grid')` | `Widget.queryAll('grid')` | Returns an **array** |
| — | `Widget.query('grid')` | Returns the **first** match only. Do not assume it behaves like `Ext.ComponentQuery.query` |
| `cmp.up('panel')` | `widget.up('panel')` | Selector is a **type string or a function** — not a CSS-ish selector. `closest()` also exists and includes the widget itself |
| `cmp.down('button')` | `container.query(w => w.isButton)` | The instance `query` / `queryAll` take a **function only** |
| `cmp.down('#saveBtn')` | `container.widgetMap.saveBtn` (with `ref : 'saveBtn'`) | This is the idiomatic replacement for `down('#id')` |
| `cmp.getComponent('id')` | `container.getWidgetById('id')` | |
| `cmp.child()` / `cmp.items.getAt(0)` | `container.items[0]` | `items` is a plain array of widgets once built |
| `Ext.getBody()` | `document.body` | |

Static selectors accept `'grid'`, `'#myScheduler'`, `'[ref=timeline]'` or a function, and are
aliased globally as `bryntum.query` / `bryntum.queryAll` — handy from the console.

Prefer `ref` over `id`: `id` must be globally unique across the document, `ref` need not be.

---

## Store

| Ext JS | Bryntum | Note |
|---|---|---|
| `store.first()` / `store.last()` | `store.first` / `store.last` | **Getters** |
| `store.getCount()` | `store.count` | `getCount(options)` also exists, but `count` is the idiomatic read |
| `store.getAt(i)` | `store.getAt(i)` | Same |
| `store.getById(id)` / `store.get(id)` | `store.getById(id)` | No `get()` |
| `store.indexOf(rec)` | `store.indexOf(recordOrId)` | Also accepts an id |
| `store.getRange()` | `store.getRange(start, end)` or `store.records` | |
| `store.each(fn)` | `store.forEach(fn)` | **No `each()`.** `map` and `reduce` are there too |
| `store.find('name', 'Bob')` | `store.findRecord('name', 'Bob')` or `findByField('name', 'Bob')` | Bryntum `find(fn)` takes a predicate, like `Array#find` |
| `store.query('name', /bob/)` | `store.query(fn)` | Also a predicate, not property+value |
| `store.filter('city', 'Paris')` | `store.filter({ property : 'city', value : 'Paris', operator : '=' })` | Also accepts a predicate, an array of filters, or `{ filters, replace }` |
| `store.filterBy(fn)` | `store.filterBy(fn)` | Same |
| `store.clearFilter()` | `store.clearFilters()` | **Plural.** Single removal is `removeFilter(idOrInstance)` |
| `store.sort('name', 'DESC')` | `store.sort('name', false)` | Second arg is `ascending`. Third arg `add : true` appends a sorter for multi-sort |
| `store.group('city')` | `store.group('city')` | Same. Also `addGrouper` / `removeGrouper` / `clearGroupers` |
| `store.add/insert/remove/removeAll` | `store.add(records)` / `insert(index, records)` / `remove(records)` / `removeAll()` | Note `insert` is `(index, records)` |
| `store.sync()` | `store.commit()` | On `AjaxStore`. `acceptChanges()` marks clean without a request; `revertChanges()` rolls back |
| `store.getModifiedRecords()` | `store.changes` / `store.hasChanges` | `changes` is `{ added, modified, removed }` |
| `store.sum('amount')` | `store.sum('amount')` | Also `min`, `max`, `average` |
| `proxy : { type : 'ajax', url : … }` | `new AjaxStore({ readUrl, createUrl, updateUrl, deleteUrl })` | `await store.load()`; `autoLoad` and `autoCommit` exist |
| `grid.getStore()` | `grid.store` | Property, not a method |
| `store.loadData(arr)` | `store.data = arr` | `loadData(data, action)` also exists |

---

## Model / record

| Ext JS | Bryntum | Note |
|---|---|---|
| `record.get('name')` | `record.get('name')` or `record.name` | Declared fields are exposed as properties |
| `record.set('name', v)` | `record.set('name', v)` | Also `record.name = v` for declared fields |
| — | `record.getValue(f)` / `setValue(f, v)` | Field-name-driven variants |
| `record.data` | `record.data` | Same |
| `record.dirty` | `record.isModified` | |
| `record.getChanges()` | `record.modifications` | Also `isFieldModified(name)` |
| `record.reject()` | `record.revertChanges()` | |
| `record.commit()` | `record.clearChanges()` | |
| `record.copy()` | `record.copy(newIdOrData, deep)` | |
| `store.remove(record)` | `record.remove()` | Records can remove themselves |

**Assigning an undeclared property does not reach `record.data`.** If the field is not in
`static fields`, `record.foo = 1` lands as a plain own property: no dirty flag, no autoSync, no
persistence. Declare the field.

### Tree nodes

| Ext JS | Bryntum | Note |
|---|---|---|
| `node.appendChild(child)` | `node.appendChild(child, silent, options)` | |
| `node.insertChild(index, child)` | `node.insertChild(child, before, silent, options)` | **The second argument is the sibling record to insert BEFORE, not an index.** Reversed argument order *and* a different second argument |
| `node.removeChild(child)` | `node.removeChild(childRecords, isMove, silent, options)` | |
| `node.isLeaf()` | `node.isLeaf` | Getter |
| `node.getDepth()` | `node.childLevel` | |
| `node.cascadeBy(fn)` | `node.traverse(fn, skipSelf, options)` | |
| `node.expand()` | `await grid.features.tree.expand(record)` | Expansion is driven by the `tree` feature, not the record. Also `collapse`, `toggleCollapse`, `expandAll`, `collapseAll`, `expandTo`, `expandToLevel` |

---

## Config names

| Ext JS | Bryntum | Note |
|---|---|---|
| `xtype` | `type` | |
| Column `dataIndex` | Column `field` | **`dataIndex` does not exist anywhere in Bryntum** |
| Column `locked : true` | Column `locked : true` (sugar for `region : 'locked'`) | Same spelling, and `region` is the underlying config |
| Column `flex` / `width` / `text` / `align` / `hidden` | Same | |
| Column `sortable` / `filterable` | Same, both default `true`; `resizable` and `draggable` too | |
| `handler : fn` on a button | `onClick : fn` | There is no `handler` config. Any `onXxx` config becomes a listener for event `xxx` |
| `listeners : { click : fn, scope : this }` | `listeners : { click : fn, thisObj : this }` | `scope` → **`thisObj`**. `buffer` and `throttle` work as in Ext |
| `cmp.on('click', fn, scope)` | `widget.on({ click : fn, thisObj : this })` | `on`, `un`, `once`, `trigger` all exist. `on()` returns a detacher function |
| `cls` / `style` / `html` / `flex` / `hidden` / `disabled` | Same | Plus `dataset`, `weight`, `tooltip`, `ref`, `masked` |
| `tbar` / `bbar` | `tbar` / `bbar` on `Panel` | Toolbar string shortcuts are `'->'` (filler) and `'|'` (separator) |
| `layout : 'hbox'` | `layout : 'box'` (alias `'hbox'`) | Available: `default`, `box`/`hbox`, `vbox`, `card`, `fit` |
| `defaults : { … }` | `defaults : { … }` on a Container | Same idea |
| `Ext.tab.Panel` `activeTab` | `TabPanel` — setter takes a widget, **getter returns a Number index** | Assert against `activeItem` if you need the widget |
| `Ext.window.Window` | `Popup` | There is no `window` type |
| `Ext.form.Panel` | `Panel` / `Container` with field items | There is no `form` type. Read values with `container.values` or `getValues()`; validity with `container.isValid` |
| `Ext.form.field.ComboBox` `store` + `displayField` + `valueField` | `Combo` with `items` (or `store`), `displayField` (default `'text'`), `valueField` | |
| `beforerender` / `afterrender` | The `paint` event | Fires with `{ firstPaint }` |

### Renderers

Ext JS passes positional arguments; Bryntum passes **one object**:

```javascript
// Ext JS
renderer : function (value, metaData, record, rowIdx, colIdx, store, view) { … }

// Bryntum
renderer : ({ record, value, column, row, cellElement }) => `${value} (${record.city})`
```

Returning a string, a `DomConfig` object, or (with the React wrapper) JSX all work. Return
`undefined` to skip rendering entirely.

---

## Dialogs, ajax, formatting, masking

| Ext JS | Bryntum | Note |
|---|---|---|
| `Ext.Msg.alert(title, msg)` | `await MessageDialog.alert({ title, message })` | `MessageDialog` is a **singleton instance**, not a class you construct |
| `Ext.Msg.confirm(...)` | `const result = await MessageDialog.confirm({ title, message })` | Resolves to `MessageDialog.okButton` / `.cancelButton` |
| `Ext.Msg.prompt(...)` | `await MessageDialog.prompt({ … })` | Resolves to `{ button, text }` |
| `Ext.toast('Saved')` | `Toast.show('Saved')` | Config form takes `html`, `timeout`, `showProgress`. **Never pass `type`** |
| `Ext.Ajax.request({ url, success })` | `await AjaxHelper.get(url, { parseJson : true })` | Also `post(url, payload, options)` and `fetch`. Promise-based; read `parsedJson` |
| `Ext.Date.format(d, 'Y-m-d')` | `DateHelper.format(date, 'YYYY-MM-DD')` | Also `parse`, `add`, `diff`, `as`, `startOf`, `endOf`, `clearTime`. **The format tokens are different** — Bryntum uses Unicode-style patterns, not PHP-style |
| `Ext.util.Format.htmlEncode` | `StringHelper.encodeHtml` | Also `capitalize`, `hyphenate`, `stripHtmlTags`, `xss` tagged template |
| `Ext.Array` / `Ext.Object` | `ArrayHelper` / `ObjectHelper` | |
| `cmp.mask('Loading')` | `widget.masked = 'Loading'` | Or `Mask.mask(text, targetElement)` |
| `Ext.tip.ToolTip` | `Tooltip` widget, or a grid/scheduler tooltip feature | See `bryntum-features` |

---

## Refreshing and selection

| Ext JS | Bryntum | Note |
|---|---|---|
| `grid.getView().refresh()` | `grid.refreshRows()` | **`Grid` has no `refresh()`.** Calendar, TaskBoard, Scheduler and Gantt do |
| refresh one column | `grid.refreshColumn(column)` | Grid only |
| `grid.getSelectionModel().getSelection()` | `grid.selectedRecords` | Also `selectedRecord`, `selectedCells` |
| `sm.select(rec)` / `deselectAll()` | `grid.selectRow({ record })` / `grid.deselectRow(record)` / `grid.deselectAll()` | |
| `selModel : { mode : 'MULTI' }` | `selectionMode : { multiSelect : true, cell : false, checkbox : true }` | One config object, not a selection-model class |

In cell-selection mode, `grid.isSelected(record)` does **not** report rows that are selected via
their cells. Use `grid.selectedRecords.includes(record)` for "is this row in the active
selection".

---

## Things that have no Ext JS counterpart

Do not try to map these — learn them instead:

- **The `ProjectModel` and the scheduling engine.** Scheduler Pro and Gantt do not just display
  data; a constraint solver recalculates dates from dependencies, calendars and constraints.
  There is no Ext JS analogue. Never write to `endDate` to "move" a task — set `startDate` and
  `duration`, or a constraint, and let the engine resolve.
- **Features, not plugins.** Bryntum's `features : { … }` object is configured at construction
  and is much broader than Ext's `plugins`. See the `bryntum-features` skill.
- **`ref` + `widgetMap`** as the standard way to reach a descendant widget.
- **The config system's `changeX` / `updateX` hooks**, which replace `apply*` / `update*` plus
  `initComponent`.
- **Framework wrappers.** React/Angular/Vue components are first-class (`bryntum-react`,
  `bryntum-angular`, `bryntum-vue`), not an afterthought.

---

## When you are unsure

Do not guess an API name from the Ext JS one. Look it up:

- MCP: `mcp__bryntum__search_bryntum_docs` (pass `product` + `version`).
- Docs: `https://bryntum.com/products/{product}/docs/`

A wrong API name that happens to read like Ext JS is the single most expensive mistake in this
codebase, because half of them fail silently rather than throwing.

---

## Related

- `bryntum` — the core skill; install first.
- `bryntum-features` — the built-in feature for what the user is describing.
- `bryntum-crud` — CrudManager and AjaxStore backends (the `proxy` replacement).
