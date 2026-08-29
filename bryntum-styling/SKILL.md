---
name: bryntum-styling
description: >
  Custom styling of Bryntum Scheduler / Scheduler Pro event bars — eventRenderer layouts,
  event bar DOM structure, padding CSS variables, and sticky content. Use alongside the
  `bryntum` skill when the user customizes what's rendered inside event bars: multi-line
  layouts, icons, time + name rows, or content that should fill the bar. Trigger on
  "eventRenderer", "style the event bars", "customize event content", "two-line events",
  "event bar padding", or content not filling / overflowing the bar. Also covers widget
  `rendition` — the built-in visual variants of buttons (`filled`, `tonal`, `outlined`, `text`,
  `elevated`) and text fields (`outlined`, `filled`), and the form look (aligned labels, a tonal
  primary button). Trigger on "button style", "primary button", "filled button", "rendition",
  "outlined field", "form layout". For picking themes or dark mode use `bryntum-theming` instead.
metadata:
  tags: bryntum, styling, eventrenderer, event-bar, scheduler, schedulerpro, css, rendition, button, form
---

## Event bar structure

The DOM structure of an event bar:

```html
<div class="b-sch-event-wrap">      <!-- renderData.wrapperCls classes land here -->
    <div class="b-sch-event">       <!-- renderData.cls classes land here -->
        <div class="b-sch-event-content">
            <!-- eventRenderer output goes here -->
        </div>
    </div>
</div>
```

- `.b-sch-event-content` already has built-in padding — don't add your own. Adjust it via CSS variable on the wrapper: `--b-sch-event-padding-inline` (horizontal mode) / `--b-sch-event-padding-block` (vertical mode), e.g. `.b-sch-event-wrap { --b-sch-event-padding-inline: 1em; }`.
- Event content is **sticky** by default (kept in view while scrolling the time axis), so it does NOT stretch to fill the bar. For custom layouts that should fill the bar (multi-line, stacked), disable it: `features: { stickyEvents: false }`.
- `eventRenderer({ eventRecord, renderData })` can return a DOM config array for multi-line layouts (flex-column `.b-sch-event-content`). If you render the icon in your own markup, set `renderData.iconCls = null` to suppress the default icon.

---

## Two-line event bar layout (worked example)

A header row with start time + icon, and a bold name below:

```js
const scheduler = new Scheduler({
    features : {
        // Turning off stickyEvents makes event content stretch to fill the bar
        stickyEvents : false
    },

    // Renders two lines in each event bar
    eventRenderer({ eventRecord, renderData }) {
        const { iconCls } = renderData;

        // The icon is rendered in the header markup below, prevent the default icon rendering
        renderData.iconCls = null;

        return [
            {
                class    : 'b-event-header',
                children : [
                    {
                        tag   : 'span',
                        class : 'b-event-time',
                        text  : DateHelper.format(eventRecord.startDate, 'h:mm A').toLowerCase()
                    },
                    iconCls?.length ? { tag : 'i', class : iconCls } : null
                ]
            },
            {
                class : 'b-event-name',
                text  : eventRecord.name
            }
        ];
    }
});
```

```css
.b-scheduler .b-sch-event-content {
    flex-direction               : column;
    align-items                  : stretch;
    justify-content              : center;
    gap                          : 0.2em;

    --b-sch-event-padding-inline : .75em;
}

.b-event-header {
    display         : flex;
    align-items     : center;
    justify-content : space-between;
    gap             : 0.5em;
    font-size       : 0.8em;
}

.b-event-name {
    font-weight : 600;
}

.b-event-time,
.b-event-name {
    overflow      : hidden;
    text-overflow : ellipsis;
    white-space   : nowrap;
}
```

Events supply `iconCls` (e.g. `'fa fa-user'`) in their data for the header icon. Give rows room for two lines (e.g. `rowHeight : 75`).

---

## Widget renditions — built-in visual variants, no CSS

Buttons, text fields and tooltips ship several looks. Pick one with the `rendition` config instead of writing CSS or
inventing class names; the value becomes a class (`b-button-tonal`, `b-text-field-outlined`), so the theme styles it.

| Widget | `rendition` values | Default |
|---|---|---|
| `Button` (and `Button` subclasses) | `'elevated'` raised with a shadow · `'filled'` primary colour · `'tonal'` faded primary · `'outlined'` border, pale/transparent fill · `'text'` transparent, text only | theme-dependent (`text` in Svalbard) |
| `ButtonGroup` | the Button values, plus `'padded'` and `'padded-filled'` — applied to every button in the group | as Button |
| `TextField`, `TextAreaField`, `NumberField`, `DateTimeField` and their subclasses (Combo, DateField, TimeField, DurationField, …) | `'outlined'` · `'filled'` | theme-dependent |
| `Tooltip` | `'plain'` · `'rich'` (title + body chrome) | `'plain'` |

Nothing else takes `rendition` — not Panel, Toolbar, Grid columns or Menu items; style those with CSS variables.

```javascript
// One primary action, secondary actions as text buttons
new Toolbar({
    items : [
        { type : 'button', text : 'Save',   rendition : 'tonal' },   // or 'filled' for the strongest emphasis
        { type : 'button', text : 'Cancel' }                         // default (text) rendition
    ]
});

// A filled field stands out on a dark toolbar
{ type : 'textfield', label : 'Search', rendition : 'filled' }

// A rich tooltip with a title bar
new Tooltip({ forSelector : '.b-sch-event', rendition : 'rich', title : 'Details', html : '…' });
```

### A form that looks like a form

A panel of fields needs three things the defaults do not give it:

```javascript
new Panel({
    title         : 'Employee',
    width         : 480,                  // a form wants a width of its own (400–560); a flex layout stretches it
    labelPosition : 'align-before',       // labels in one column, inputs in another — `'before'` rags the inputs
    items         : [
        { type : 'textfield',  label : 'Full name', name : 'name' },
        { type : 'combo',      label : 'Department', name : 'dept', items : ['Sales', 'Ops'] },
        { type : 'datefield',  label : 'Hire date', name : 'hired' }
    ],
    bbar : [
        '->',
        { type : 'button', text : 'Reset' },
        { type : 'button', text : 'Submit', rendition : 'tonal' }  // the primary action reads as one
    ]
});
```

`labelPosition` is a Container config (`'before'`, `'above'`, `'align-before'`); set on the panel it applies to every
field inside it, including ones added later. `rendition` is per widget — set it on the primary button only.

## Related

- Themes, dark mode, and theme-level CSS variables: `bryntum-theming` skill.
- CSS rules of thumb (prefer CSS-only solutions, no magic z-index): see the `bryntum` skill.
