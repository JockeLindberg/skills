# 6.x → 7.x hop

Applies when `installed < 7.0.0 <= target`, on top of what the guides say.

- **CSS class names are kebab-case** in 7.0: `.b-buttongroup` → `.b-button-group`, `.b-timeline-subgrid` → `.b-timeline-sub-grid`, and many more.
- **SASS themes are gone** (Grid 7.0.0 guide: "New themes & styling changes"). Themes are plain nested CSS + CSS variables: Svalbard (default), Stockholm, Visby, Material3, High Contrast, and Fluent2 (added in 7.1.0), each as `<theme>-light.css` / `<theme>-dark.css`, always loaded together with the structural `<product>.css`. Single-file imports like `gantt.stockholm.css` no longer exist. Which theme to pick is the user's call, so put it under Unknowns in the plan. Propose Svalbard, the v7 default. Offer the same-named `-light` sheet (`gantt.stockholm.css` → `stockholm-light.css`) as the option closest to today's look.
- **Even the same-named theme looks different.** Compared with v6 `gantt.stockholm.css`, both `stockholm-light` and `svalbard-light` show:
  - sentence-case grid headers and buttons (v6 uppercased them)
  - a light-blue selected row instead of orange
  - round icon-only buttons
  - roughly double the grid cell `padding-left`
  - thinner parent task bars

  List these under Unknowns so the user can ask for overrides before approving.
- **FontAwesome Free is no longer built in** (Grid 7.0.0 guide: "FontAwesome Free no longer built in"). Import `fontawesome/css/fontawesome.css` + `solid.css` from the package, and icon classes lose the `b-fa` prefix: `'b-fa b-fa-plus'` → `'fa fa-plus'`.
- **Classes that moved rather than being renamed** don't appear in any rename table, and a custom selector that relies on one silently stops matching. In Gantt 7, a selected task gets `b-selected` on `.b-gantt-task-wrap`, and the built-in selection styling targets `.b-gantt-task-wrap.b-selected`. `b-task-selected` is still set, but on the inner `.b-gantt-task`, so a rule like `.b-gantt-task-wrap.b-task-selected` never matches. For each custom selector, confirm it matches in the running app (computed style or a DOM class check) before and after the upgrade. The kebab-case renames also hit the selectors in your own baseline/QA scripts. `.b-sch-header-timeaxis-cell` is `.b-sch-header-time-axis-cell` in 7.x, so write such selectors to match both, e.g. `:is(.b-sch-header-timeaxis-cell, .b-sch-header-time-axis-cell)`.
- **Button colour classes are gone.** v6 `cls : 'b-green'` (`b-blue`, `b-red`, …) matches nothing in 7.x `gantt.css`, and the button silently loses its colour. Use `color : 'b-green'`, which sets `--b-primary`, and add a `rendition`, because the default depends on the theme. Stockholm defaults to `outlined` (transparent with a grey border), and Svalbard to `text` (no border). `tonal` comes closest to the v6 Stockholm coloured button, but only approximately. v6 had a near-transparent fill with bright green text and a green border; `tonal` gives a visible pale fill, darker green text and no border. `filled` gives a solid fill with white text. Compare the computed `color` / `background-color` against the baseline. `.b-raised` / `.b-transparent` also become `rendition` values (`migrate-to-new-css.md` item 4 shows `b-raised` → `filled`). Grep `cls` values for `b-(red|green|blue|…)`, and compare the result with the baseline.
- **Project data props** are the most common leftover in 6.x code. They were deprecated in 6.3.0, not 7.0 (rollup `## Gantt v6.3.0` → "Naming simplification for project data properties"): `tasksData` → `tasks`, `eventsData` → `events`, `resourcesData` → `resources`, `assignmentsData` → `assignments`, `dependenciesData` → `dependencies`, `timeRangesData` → `timeRanges`, `calendarsData` → `calendars`. Also applies to the `inlineData` / `json` shapes.

## The codemod

The shipped codemod (`tools.migrate6to7` in the index, `<Product>/migrate.js` in a zip) rewrites CSS class names and font imports only — never the JS API. It rewrites selectors in CSS and in JS/HTML strings, which is what you want, but review its diff for generic class names it may have caught.

With no index key and no zip, skip it and handle the selector renames as plan actions. Otherwise fetch it to a temp file and dry-run first:

```bash
curl -sf <baseUrl><tools.migrate6to7> -o /tmp/bryntum-migrate.js
node /tmp/bryntum-migrate.js ./src --migrations css,fonts --exclude "node_modules/**" --dry-run
node /tmp/bryntum-migrate.js ./src --migrations css,fonts --exclude "node_modules/**"
```

Options: `--migrations css,fonts,all`, `--include "*.js"`, `--exclude "node_modules/**"`, `--dry-run`.

Run it before any manual CSS edits.
