---
name: bryntum-theming
description: >
  Bryntum themes and dark mode. Use alongside the `bryntum` skill whenever the user picks or
  switches a Bryntum theme, wants light/dark mode or a runtime theme toggle, asks the component
  to match a design system (Material / Fluent), or customizes colors via CSS variables. Covers
  all products — Scheduler, Scheduler Pro, Gantt, Calendar, Grid, TaskBoard. Trigger on phrases
  like "dark mode", "theme switch", "toggle theme", "DomHelper.setTheme", "which theme",
  "match Material UI", or "Bryntum theme".
metadata:
  tags: bryntum, theme, theming, dark-mode, domhelper, settheme, css-variables
---

## Theme catalog

`svalbard-light` is the default. Every theme ships a light and a dark variant:

`svalbard-light` / `svalbard-dark`, `stockholm-light` / `stockholm-dark`, `visby-light` / `visby-dark`, `material3-light` / `material3-dark`, `fluent2-light` / `fluent2-dark`

The **structural** `{product}.css` must always be imported *in addition to* the theme file — see the CSS setup section of the `bryntum` skill.

---

## Choosing a theme

To make the Bryntum component blend with a design-system host app, start from a matching theme rather than restyling widgets by hand:

- Material UI → `material3-light` / `material3-dark`
- Fluent / Microsoft-style → `fluent2-light` / `fluent2-dark`
- Otherwise → `svalbard-light` (default)

Customize further with CSS variables (`--b-widget-background`, etc.), not by overriding individual class rules.

---

## Dynamic theme switching (dark mode)

**Must** use `DomHelper.setTheme()` with `<link>` tags — CSS `@import` will NOT work.

**1. Load CSS via `<link>` in `index.html`** (not CSS `@import`):
```html
<link rel="stylesheet" href="/node_modules/@bryntum/{product}/fontawesome/css/fontawesome.css" />
<link rel="stylesheet" href="/node_modules/@bryntum/{product}/fontawesome/css/solid.css" />
<link rel="stylesheet" href="/node_modules/@bryntum/{product}/{product}.css" />
<link rel="stylesheet" href="/node_modules/@bryntum/{product}/svalbard-light.css" data-bryntum-theme />
```
The `data-bryntum-theme` attribute is **required** — `setTheme()` finds and swaps this `<link>`.

**2. Call `DomHelper.setTheme()`**:
```javascript
import { DomHelper } from '@bryntum/{product}';
DomHelper.setTheme('svalbard-dark');
```

**What does NOT work**:
- CSS `@import` + `setTheme()` — no `<link>` to target, silently fails
- Manual CSS overrides — only covers a fraction of theme variables

---

## Related

Customizing what's rendered *inside* event bars (`eventRenderer`, multi-line layouts, event bar padding) is the `bryntum-styling` skill — theming only picks the look of the whole component.
