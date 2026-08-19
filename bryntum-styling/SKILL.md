---
name: bryntum-styling
description: >
  Custom styling of Bryntum Scheduler / Scheduler Pro event bars — eventRenderer layouts,
  event bar DOM structure, padding CSS variables, and sticky content. Use alongside the
  `bryntum` skill when the user customizes what's rendered inside event bars: multi-line
  layouts, icons, time + name rows, or content that should fill the bar. Trigger on
  "eventRenderer", "style the event bars", "customize event content", "two-line events",
  "event bar padding", or content not filling / overflowing the bar. For picking themes
  or dark mode use `bryntum-theming` instead.
metadata:
  tags: bryntum, styling, eventrenderer, event-bar, scheduler, schedulerpro, css
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

## Related

- Themes, dark mode, and theme-level CSS variables: `bryntum-theming` skill.
- CSS rules of thumb (prefer CSS-only solutions, no magic z-index): see the `bryntum` skill.
