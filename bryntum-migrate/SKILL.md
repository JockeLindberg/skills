---
name: bryntum-migrate
description: >
  Upgrade an existing Bryntum app (Gantt, Scheduler Pro, Scheduler, Calendar, Grid, TaskBoard)
  from its installed version to a newer one — 6.x → 7.x, 7.x → 7.y, 7.x → 8.0, 8.x → 8.y (e.g.
  Gantt 6.0.3 → 7.2.1, Scheduler Pro 7.3 → Scheduler 8.0 `tier : 'enterprise'`). Detects installed
  and target versions, gathers the ORDERED upgrade guides / what's-new guides / changelogs across the
  product AND the products it inherits from, cross-references them with the customer's code,
  writes a migration plan the user must explicitly approve, then applies and verifies it.
  Trigger on "upgrade/update/migrate Bryntum to <version>", "bump @bryntum/*", "update Bryntum",
  "what changed between 6.x and 7.x", "move from Scheduler Pro to Scheduler 8", "my Gantt/Scheduler
  broke after updating", or the runtime error "CSS version X doesn't match bundle version Y". NOT for
  migrating from another vendor
  (DHTMLX, Syncfusion, FullCalendar, DevExpress… — those are the
  `guides/migration/migrate-*-to-bryntum` docs) and NOT for first-time installation (use the
  `bryntum` skill).
metadata:
  tags: bryntum, migrate, upgrade, update, changelog, breaking-changes, codemod, versions, schedulerpro, tier
---

## Load alongside this skill

Load the `bryntum` skill plus the framework skill for the app (`bryntum-react`, `bryntum-angular`, `bryntum-vue`, `bryntum-vanilla`), or fetch the raw file if not installed: `https://raw.githubusercontent.com/bryntum/skills/refs/heads/main/<skill>/SKILL.md`.

Use the MCP tool `mcp__bryntum__search_bryntum_docs` (pass `product` + `version`) to look up any API item you meet in a guide. It does **not** index upgrade guides or changelogs — those come from the sources in Phase 1. If the MCP is unavailable, ask the user to add it:

```bash
claude mcp add --transport http bryntum https://mcp.bryntum.com
```

---

## Rules that apply to every phase

| Do | Don't |
|----|-------|
| Bump **every** `@bryntum/*` package to the same exact version | Upgrade only one package — the runtime throws `CSS version X doesn't match bundle version Y` |
| Read the upgrade guides of the product **and** its sibling (inherited) products | Read only the top-level product's guide — a Gantt app breaks on Grid/Scheduler changes too |
| Read all upgrade guides + API/STYLING digests in full | Read 30 bug-fix-only changelogs — list them, don't read them |
| Take the installed version from `node_modules` or the lockfile | Trust the semver range in `package.json` |
| Targeting 8.x from a Scheduler Pro app: switch to `@bryntum/scheduler*` + `tier : 'enterprise'` | Keep `@bryntum/schedulerpro` when targeting 8.x — the package line ends at 7.x, the classes are deprecated aliases until 9.0.0 |
| Treat the "v8 headline changes" table as a checklist the plan must answer | Assume the package name is stable across the hop |
| Write the plan, then **stop** and wait for explicit approval | Edit a single file before the user approves the plan |
| Keep the plan file updated as a checklist while applying | Apply from memory and report at the end |
| Use `search_bryntum_docs` with the **target** `version` for API lookups | Guess replacement APIs from the old version's docs |
| Tell the user to check the browser console for deprecation warnings afterwards | Claim the migration is complete because the build is green |

---

## Phase sequence

Work through **0 → 4 in order**. Phase 2 ends with a hard stop for user approval.

### 0. Detect

1. **Product(s) and framework.** Read `package.json` `dependencies` + `devDependencies` for `@bryntum/*`. Strip `-trial`, `-thin`, and the wrapper suffix to get the product id:

   | Package | `<product>` | `<Product>` | Framework |
   |---------|-------------|-------------|-----------|
   | `@bryntum/gantt`, `-trial`, `-thin` | `gantt` | `Gantt` | — |
   | `@bryntum/schedulerpro` … | `schedulerpro` — becomes `scheduler` when target ≥ 8.0.0 (step 4) | `SchedulerPro` → `Scheduler` | — |
   | `@bryntum/scheduler` … | `scheduler` | `Scheduler` | — |
   | `@bryntum/calendar` … | `calendar` | `Calendar` | — |
   | `@bryntum/grid` … | `grid` | `Grid` | — |
   | `@bryntum/taskboard` … | `taskboard` | `TaskBoard` | — |
   | `@bryntum/<product>-react` | as above | as above | React |
   | `@bryntum/<product>-angular` | as above | as above | Angular |
   | `@bryntum/<product>-vue-3` | as above | as above | Vue 3 |
   | `@bryntum/<product>-vue` (no `-3`) | as above | as above | **Vue 2 — removed in 8.0.0**, no target ≥ 8 possible without a Vue 3 migration first |
   | `@bryntum/<product>-angular-view` | as above | as above | Angular ≤ 11 View Engine — no longer published in 8.0.0 |

   No `@bryntum/*` dependency but a local `build/package.json` → **zip install**. The zip ships `docs/` (upgrade guides + what's-new), `changelog.md`, and a `migrate.js` codemod at the product root. Several products installed (e.g. Gantt + Grid)? Run the sequence once with the **highest** product; its sibling list covers the rest.

2. **Installed version — authoritative source.** In this order: `node_modules/@bryntum/<pkg>/package.json` → `"version"`; else the resolved version in `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` / `bun.lock`; zip install → `build/package.json`. Never the range in `package.json`. Check that every `@bryntum/*` package resolves to the same version — a mismatch is itself the first action item.

3. **Target version.** User-given, else the latest stable from `npm view @bryntum/<product> versions --json` (works for licensed `npm.bryntum.com` and public trial packages; if the licensed registry rejects the call, query `@bryntum/<product>-trial` — same version list). Then:
   - target < installed → **refuse** (this skill does not downgrade) and stop.
   - target == installed → nothing to migrate; report and stop.
   - target contains `-` (alpha/beta/rc) → warn that prereleases have no guides yet and confirm before continuing.

4. **Package name across the hop.** Derive `<product>` from the **installed** package, then map it for the **target**: when installed is `schedulerpro` and target ≥ 8.0.0, the target product is `scheduler` — Scheduler Pro is merged into Scheduler in 8.0 as its Enterprise tier (`tier : 'enterprise'`; tiers are `'community' | 'business' | 'enterprise'`). Every `@bryntum/schedulerpro*` package (`-react`, `-angular`, `-vue-3`, `-thin`, `-trial`) becomes the matching `@bryntum/scheduler*` package. `SchedulerPro` / `SchedulerProBase` classes and the `schedulerpro` / `schedulerprobase` widget types survive as deprecated aliases until 9.0.0, so old code runs but warns. Record "package rename: `@bryntum/schedulerpro` → `@bryntum/scheduler`" for the plan header. All other products keep their name.

5. **Sibling products whose guides also apply** (the product inherits from them). Use the installed product for the 7.x part of the range and the target product for the 8.x part:

   | App product | Releases < 8.0.0 | Releases ≥ 8.0.0 |
   |-------------|------------------|------------------|
   | Gantt | Grid → Scheduler → SchedulerPro → Gantt | Grid → Scheduler → Gantt |
   | SchedulerPro (→ Scheduler in 8) | Grid → Scheduler → SchedulerPro | Grid → Scheduler |
   | Scheduler | Grid → Scheduler | Grid → Scheduler |
   | Calendar | Grid → Scheduler → Calendar | Grid → Scheduler → Calendar |
   | Grid | Grid | Grid |
   | TaskBoard | TaskBoard | TaskBoard |

   No SchedulerPro guides exist for 8.x — its changes are in the Scheduler 8.x guides. Core and Chart have no upgrade guides of their own. This table is for the **fallback** path: when `index.json` is available (Phase 1 step 1) use its `products` list instead — it adds the Core and Chart changelog digests, which carry Store / Model / DomHelper / CSS-variable breaking changes that are never mirrored into product changelogs.

6. **Package manager** from the lockfile: `package-lock.json` → npm, `yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, `bun.lock`/`bun.lockb` → bun.

7. **Environment facts a v8 target needs** (skip when target < 8.0.0): Angular version from `@angular/core` (8.0 requires 15+), any `*.umd.js` import or `<script src=".../<product>.umd.js">` (UMD bundle no longer shipped), any Vue 2 wrapper. Each hit is a mandatory plan item, not a note.

8. Tell the user in one block: product (with package rename if any), framework, installed → target, sibling list, package manager. Continue without waiting.

### 1. Gather the ordered reading list

1. **Primary source — try first.** `curl -sf https://bryntum.com/products/<product>/docs-llm/migration/index.json`. On 200, use it and skip step 2. Schema:

   ```json
   {
     "schemaVersion": 1,
     "product": { "id": "gantt", "name": "Gantt" },
     "baseUrl": "https://bryntum.com/products/gantt/docs-llm/migration/",
     "docsApiBaseUrl": "https://bryntum.com/products/gantt/docs/api/",
     "docsGuideBaseUrl": "https://bryntum.com/products/gantt/docs/guide/",
     "docsLlmGuideBaseUrl": "https://bryntum.com/products/gantt/docs-llm/guide/",
     "products": ["Gantt", "SchedulerPro", "Scheduler", "Grid", "Chart", "Core"],
     "productOrder": "most-specific first; read guides for a version in reverse order (base product first)",
     "guideCoverage": {
       "Gantt": { "upgrades": true, "whatsNew": true }, "SchedulerPro": { "upgrades": true, "whatsNew": true },
       "Scheduler": { "upgrades": true, "whatsNew": true }, "Grid": { "upgrades": true, "whatsNew": true },
       "Chart": { "upgrades": false, "whatsNew": false }, "Core": { "upgrades": false, "whatsNew": false }
     },
     "digests": { "Gantt": "Gantt/changelog/api-changes.md", "Grid": "Grid/changelog/api-changes.md", "Core": "Core/changelog/api-changes.md" },
     "tools": { "migrate6to7": "tools/migrate.js", "selectors": "tools/selectors.md" },
     "releases": [
       { "version": "7.2.1", "date": "2026-02-26", "products": {
           "Gantt": { "changelog": { "url": "Gantt/changelog/7.2.1.md", "sections": ["demos", "bug-fixes"] } },
           "Grid":  { "changelog": { "url": "Grid/changelog/7.2.1.md",  "sections": ["api-changes", "bug-fixes"] } }
       } },
       { "version": "6.2.0", "date": "2025-04-10", "products": {
           "Gantt": {
             "changelog": { "url": "Gantt/changelog/6.2.0.md", "sections": ["features-enhancements", "api-changes", "bug-fixes"], "breaking": true },
             "upgrade":   { "url": "Gantt/upgrades/6.2.0.md",  "source": "Gantt/upgrades/6.0.0+.md",  "anchor": "Gantt v6.2.0" },
             "whatsNew":  { "url": "Gantt/whats-new/6.2.0.md", "source": "Gantt/whats-new/6.0.0+.md", "anchor": "Gantt v6.2.0" }
           } } }
     ]
   }
   ```

   - All `url` values are relative to `baseUrl`. `releases` is sorted **descending** — reverse it.
   - `products` is most-specific first and includes the base libraries (Core, Chart) — `productOrder` says so. For each version read guides in **reverse** `products` order (base product first). Core and Chart contribute changelogs + `digests` only, never guides.
   - `guideCoverage.<Product>.upgrades === false` means that product is changelog-only in this index (Core, Chart, and SchedulerPro on the 8.x line). A missing `upgrade` key for such a product does **not** mean "no changes this version" — for SchedulerPro 8.x they live in the Scheduler 8.x guides.
   - Per-release 7.x/8.x guides have no `source`/`anchor`; `date` is `null` for guide-only versions.
   - `sections` ids: `features-enhancements`, `api-changes`, `styling-changes`, `locale-updates`, `demos`, `bug-fixes`. A `changelog` object may also carry `"breaking": true` and/or `"deprecated": true` (omitted when false), derived from `[BREAKING]` / `[DEPRECATED]` markers on the entries — the tightest filter you have (see the priority table in step 3).
   - `upgrade` / `whatsNew` are already **pre-sliced per version** (rollups were split for you) — never slice. A slice may still contain **several** `## <Product> v<x.y.z>` sections (a version appearing twice in a rollup, or Scheduler + Scheduler Pro sharing a file on 8.x) — read the **whole** slice, not just the first section. `source`/`anchor` name the first rollup origin; `sources: [...]` lists all of them when several contributed.
   - Fetch `digests.<Product>` for **every** entry in `products` (Core and Chart included) **first**: each is a small file holding only API CHANGES + STYLING CHANGES across all releases. It is the fastest way to see the whole breaking surface.
   - `tools.migrate6to7` (CSS/fonts codemod, Phase 3) and `tools.selectors` are present **only when the files exist** — check the key before fetching.
   - Resolve relative links inside a guide: `(#Gantt/model/ProjectModel#config-x)` → `docsApiBaseUrl + "Gantt/model/ProjectModel#config-x"`; `(#Gantt/guides/basics/x.md)` → `docsLlmGuideBaseUrl + "Gantt/guides/basics/x.md"` (or `docsGuideBaseUrl` for the rendered page).

2. **Fallback — only when the index returns 404.**

   | Need | Source |
   |------|--------|
   | Release versions | `npm view @bryntum/<product> versions --json`, keep `(installed, target]` |
   | Upgrade guide | `https://bryntum.com/products/<product>/docs/guide/<Product>/upgrades/<version>` |
   | What's new | `https://bryntum.com/products/<product>/docs/guide/<Product>/whats-new/<version>` |
   | Version history (all changelogs, one page) | `https://bryntum.com/products/<product>/docs/guide/<Product>/changelog` |
   | API diff table | `https://bryntum.com/products/<product>/docs/?v=<version>#apidiff` |
   | v7 CSS migration guide | `https://bryntum.com/products/<product>/docs-llm/guide/<Product>/migration/migrate-to-new-css.md` |
   | Zip install | `docs/` + `changelog.md` inside the extracted archive |

   Guides exist **sparsely** — not every release has one; a 404 for a version is normal. Pre-7.0 guides are rollups named `6.0.0+`, `5.0.0+`, … with `## <Product> v6.2.0`-style headings inside — fetch `.../upgrades/6.0.0+` and slice it yourself by heading, keeping only versions in `(installed, target]`. From 7.0.0 there is one file per release (`7.0.0`, `7.1.0`, `7.2.0`, …, `8.0.0`, `8.1.0` — 8.x uses the same URL shapes, e.g. `.../docs/guide/Scheduler/upgrades/8.0.0`; a former Scheduler Pro app reads them under `/products/scheduler/`). Repeat for every sibling product, swapping `<product>`/`<Product>` (Grid guides live at `/products/grid/docs/guide/Grid/...`).

3. **Build the reading list**: every release in `(installed, target]`, ascending; per release, every sibling product in the order from step 0.5; per product, the upgrade guide, what's-new, and changelog sections present. Then read by priority:

   | Priority | Read | Why |
   |----------|------|-----|
   | 1 | **Every** upgrade guide, fully | This is where breaking changes and Old/New code live |
   | 1 | Every changelog flagged `"breaking": true` (index only), fully | `[BREAKING]` entries; far tighter than "has `api-changes`" — mark these releases ✱ in the plan |
   | 2 | `api-changes` + `styling-changes` (digests, or those changelog sections); changelogs flagged `"deprecated": true` | Renames, removals, deprecations, CSS class/theme changes not in a guide |
   | 3 | What's-new, skim | New features that replace a customer workaround or override |
   | 4 | `bug-fixes` | Only when customer code references an issue number or carries a workaround comment |
   | — | Releases with only `bug-fixes` / `demos` / `locale-updates` and no guide | List in the plan; do not read |

   A changelog has no "BREAKING CHANGES" section — breaking changes are in `API CHANGES`, `STYLING CHANGES`, and the upgrade guides. Entries look like `* Fixed #12206 - ...`.

4. **6 → 7 hop specifics** (when `installed < 7.0.0 <= target`), on top of the guides:
   - CSS class names normalized to kebab-case: `.b-buttongroup` → `.b-button-group`, `.b-timeline-subgrid` → `.b-timeline-sub-grid`, and many more.
   - The old SASS-built themes are gone (Grid 7.0.0 guide: "New themes & styling changes"). New themes are plain nested CSS + CSS variables: Svalbard (default), Stockholm, Visby, Material3, High Contrast — each as `<theme>-light.css` / `<theme>-dark.css`, always loaded **with** the structural `<product>.css`. A single-file import such as `gantt.stockholm.css` no longer exists. Which theme to pick is an **Unknown for the user** in the plan.
   - FontAwesome Free is no longer built into the Bryntum CSS (Grid 7.0.0 guide: "FontAwesome Free no longer built in"): import `fontawesome/css/fontawesome.css` + `solid.css` from the package yourself, and icon classes lose the `b-fa` prefix — `'b-fa b-fa-plus'` → `'fa fa-plus'`.
   - The shipped codemod (`tools.migrate6to7` in the index, `<Product>/migrate.js` in a zip) rewrites **CSS class names and font imports only**. It does not touch the JS API.
   - Deprecated project data props are the most common leftover in 6.x code — deprecated in **6.3.0**, not 7.0 (rollup section `## Gantt v6.3.0` → "Naming simplification for project data properties"): `tasksData` → `tasks`, `eventsData` → `events`, `resourcesData` → `resources`, `assignmentsData` → `assignments`, `dependenciesData` → `dependencies`, `timeRangesData` → `timeRanges`, `calendarsData` → `calendars`. Also applies to the `inlineData` / `json` shapes.

5. **7 → 8 hop — v8 headline changes** (when `installed < 8.0.0 <= target`). The 8.0.0 upgrade guides hold the details and the Old/New code — read them, do not work from this table. Use it as a checklist so the plan answers every row with *applies / not used*:

   | Headline change (8.0.0) | Grep the customer code for |
   |-------------------------|----------------------------|
   | Scheduler Pro merged into Scheduler; `SchedulerPro`/`SchedulerProBase` and `schedulerpro` widget types are deprecated aliases (removed 9.0.0); set `tier : 'enterprise'`; prefer `isEnterpriseTier` over `isSchedulerPro`; `*Enterprise` model/store classes are resolved from the tier — use tier-neutral `ProjectModel`, `EventModel`, … | `@bryntum/schedulerpro`, `SchedulerPro`, `schedulerpro`, `isSchedulerPro`, `ProjectModelPro`, `*Enterprise` |
   | UMD bundle no longer shipped — ES modules only; build your own with webpack if needed | `.umd.js`, `bryntum.<product>` global |
   | Vue 2 wrapper removed (`@bryntum/<product>-vue`); 7.3.x is the last Vue 2 line | `@bryntum/<product>-vue"`, `vue@2` |
   | Angular 15+ required; `-angular-view` package gone | `@angular/core` version, `-angular-view` |
   | `GridFeatureManager` removed — features register via the `Factoryable` pattern, `defaultEnabled` may be keyed by tier | `GridFeatureManager`, `registerFeature` |
   | Project is now a CrudManager; standalone `CrudManager` class and `crudManager` config deprecated (removed 9.0.0); `scheduler.crudManager` returns the project | `new CrudManager`, `crudManager :`, `CrudManager load response` |
   | Color API emits predefined `b-` names (`b-red`) instead of hex; swatches use `.b-color-<name>`; `DomHelper.resolveColorValue` / `createColorStyle` for literals | `eventColor`, `ColorField`, `ColorPicker`, `ColorColumn`, hex comparisons on color fields |
   | New Temporal-based time zone implementation is the default | `timeZone`, `TimeZoneHelper` |
   | Vendored `later.js` removed | `Engine/vendor/later`, `later.` |
   | Themes also ship as family sheets (`svalbard.css` renders light/dark from `color-scheme`) — optional, no migration required | theme `<link>` / `@import`, `setTheme` |
   | Per-product extras named in the guides (e.g. `amPm` preset → `sixHoursAndDay`, Grid column virtualization, Calendar DayView sticky content, Gantt `toggleParentTasksOnClick` default) | whatever the guide heading names |

6. Turn what you read into a working list of **candidate items**: `{ version, product, source URL + heading, kind, old, new, grep terms }`. Kind is one of `breaking` · `behaviour change` · `deprecation` · `cosmetic`. Grep terms are the concrete identifiers a customer file would contain: config names, method/event names, class names, CSS selectors, import paths.

### 2. Cross-reference and write the plan — STOP here

1. Grep the customer's source for every candidate item's terms: `**/*.{js,jsx,ts,tsx,vue,html,css,scss}` excluding `node_modules`, `dist`, `build`, `.angular`, coverage output. Mark each item **applies** (hits, files listed) · **not used** (zero hits) · **unsure** (dynamic access, string-built keys, config spread from another object, generic term with false positives).

2. Write `./bryntum-migration-<from>-to-<to>.md` in the project root:

   ````markdown
   # Bryntum migration: <Product> <from> → <to>

   Product: <Product> (+ siblings: …) · Framework: <framework> · Package manager: <pm>
   Package rename: none | `@bryntum/schedulerpro*` → `@bryntum/scheduler*` (Scheduler Pro is the Enterprise tier of Scheduler from 8.0)
   Releases crossed: <n> (<list>) · Upgrade guides read: <n> · Source: index.json | docs fallback

   ## Reading list
   | Version | Product | Upgrade guide | What's new | Changelog sections (✱ = BREAKING) |
   |---------|---------|---------------|------------|-----------------------------------|

   ## Actions (in version order)
   ### <version> — <Product>
   - [ ] **<short title>** · risk: breaking | behaviour change | deprecation | cosmetic
     Source: <url> → "<heading>"
     Files: `src/…`, `src/…`
     **Old code**
     ```js
     …
     ```
     **New code**
     ```js
     …
     ```

   ## Unsure — needs your answer
   - …

   <details><summary>## Not applicable (n items, zero hits in this codebase)</summary>
   - <version> · <Product> · <title> · <source>
   </details>

   ## Verification
   - [ ] Build / typecheck: `<command>`
   - [ ] Grep for removed identifiers returns zero hits: …
   - [ ] Tests: `<command>` (if present)
   - [ ] Manual: open the app, search the browser console for `Deprecation warning`

   ## Unknowns for the user
   - Theme: v7 replaces `<old theme>` — pick `svalbard-light` (default) or another
   - …
   ````

   Copy the guide's **Old code / New code** blocks into each action, adapted to the customer's actual file. Actions are grouped by version, then by product in the order of step 0.5, so they can be applied top to bottom.

3. Ask for approval — use `AskUserQuestion` if available, otherwise write: *"Review `bryntum-migration-<from>-to-<to>.md`, edit it if you like, and reply **approve** to continue."*

4. **HARD RULE: no file edits (package.json included) before approval.** When the user replies, re-read the plan file — they may have edited it — and apply the file, not your memory of it. Unsure items the user did not answer stay unapplied.

### 3. Apply — after approval only

1. **Zip install?** Instruct the user to download the target zip, replace `build/` (and `docs/` if they keep it), then re-run this skill from Phase 0 so versions are re-detected. Stop.

2. **Package rename first** (Scheduler Pro app, target ≥ 8.0.0): in `package.json` replace each `@bryntum/schedulerpro<suffix>` with `@bryntum/scheduler<suffix>` (same suffix, same trial alias shape), then rewrite imports `from '@bryntum/schedulerpro…'` → `from '@bryntum/scheduler…'`, `new SchedulerPro({` → `new Scheduler({ tier : 'enterprise',`, `type : 'schedulerpro'` → `type : 'scheduler', tier : 'enterprise'`, CSS `schedulerpro.css` → `scheduler.css`, and wrapper component/selector names exactly as the 8.0.0 guide of the wrapper spells them. Remove the old package from `node_modules` via the install step below; a leftover `@bryntum/schedulerpro` alongside `@bryntum/scheduler` loads two bundles.

   Then set **every** `@bryntum/*` dependency to the **exact** target (`"7.2.1"`, no `^`/`~`); keep any `@npm:@bryntum/<product>-trial` alias. Install with the detected package manager so the lockfile updates (`npm install` / `yarn install` / `pnpm install` / `bun install`). Confirm with `node_modules/@bryntum/<pkg>/package.json` that every package reports the target.

3. **6 → 7 hop:** if the index has a `tools.migrate6to7` key, fetch the codemod to a temp path (`curl -sf <baseUrl><tools.migrate6to7> -o /tmp/bryntum-migrate.js`; zip users already have `<Product>/migrate.js`; no key and no zip → skip the codemod and do the selector renames as plan Actions), run it dry first and show the summary:

   ```bash
   node /tmp/bryntum-migrate.js ./src --migrations css,fonts --exclude "node_modules/**" --dry-run
   node /tmp/bryntum-migrate.js ./src --migrations css,fonts --exclude "node_modules/**"
   ```

   Options: `--migrations css,fonts,all`, `--include "*.js"`, `--exclude "node_modules/**"`, `--dry-run`. Review its diff — it rewrites selectors in CSS **and** in JS/HTML strings, which is what you want, but check generic class names it may have touched.

4. Apply the **Actions** one by one, in plan order, ticking `- [x]` in the plan file after each. Never touch **Not applicable** items. Stop and ask on **Unsure** items that were not answered.

5. Update CSS / theme imports to the target layout (`bryntum` skill → CSS setup): structural `<product>.css` + one theme file + FontAwesome files; remove legacy single-file theme imports (`gantt.stockholm.css`) and any SASS imports. Update wrapper imports if the guide renamed them (e.g. `@bryntum/<product>-react` component or prop names).

6. Quick sanity greps before verifying: `grep -rn "Data\b" src` for leftover `*Data` props, `grep -rn "b-buttongroup\|b-timeline-subgrid" src`, and for a v8 target `grep -rni "schedulerpro\|\.umd\.js\|GridFeatureManager\|new CrudManager" src` — plus every other renamed identifier from the plan.

### 4. Verify

1. Run the project's own build / typecheck: whichever exist among `npm run build`, `tsc --noEmit`, `ng build`, `vite build`, `vue-tsc --noEmit`. Fix errors caused by the migration; report anything unrelated separately.

2. Grep for **every** removed / renamed identifier in the plan's Actions → must be zero hits.

3. Run the project's test script if one exists (`npm test`, `npm run test:unit`, …).

4. If a dev server can be started, start it and search the browser console for **`Deprecation warning`**. Bryntum calls `VersionHelper.deprecate('<removal version>', msg)` during a grace period, which logs `Deprecation warning: You are using a deprecated API which will change in v<version>. <message>` — so each warning names the version in which the API disappears; once that version ships the same call throws `Deprecated API use. <message>` instead. Add each warning to the plan as a follow-up item.

5. Report: applied items (ticked), remaining manual items, Unsure items left open, build/test status, and the reminder to check the browser console after the first run.

---

## Worked example — Gantt 6.0.3 → 7.2.1, React + Vite

Detection: `@bryntum/gantt` and `@bryntum/gantt-react` resolve to `6.0.3` in `node_modules`; `package-lock.json` → npm; target `7.2.1` (< 8, so no package rename); siblings Grid → Scheduler → SchedulerPro → Gantt. 33 releases in `(6.0.3, 7.2.1]`. Guides in range, per product (rollup `6.0.0+` sections count as one guide each):

| Product | Upgrade guides | What's-new guides |
|---------|----------------|-------------------|
| Grid | 6.1.2, 6.1.6, 6.2.0, 6.2.4, 7.0.0, 7.0.1, 7.0.2, 7.1.0, 7.2.0 | 6.1.0, 6.1.2, 6.1.4, 6.1.6, 6.1.7, 6.1.8, 6.2.0, 6.3.0, 7.0.0, 7.1.0, 7.1.1, 7.2.0 |
| Scheduler | 6.1.6, 6.1.8, 6.2.0, 6.3.0, 7.0.0, 7.1.0 | 6.0.5, 6.1.0, 6.1.1, 6.1.4, 6.1.8, 6.2.0, 6.3.0, 7.0.0, 7.2.0 |
| SchedulerPro | 6.2.0, 6.3.0, 7.0.0, 7.1.0, 7.2.0 | 6.0.5, 6.1.0, 6.1.3, 6.1.4, 6.1.7, 6.1.8, 6.2.0, 6.2.4, 6.3.0, 7.0.0, 7.2.0 |
| Gantt | 6.1.6, 6.2.0, 6.3.0, 7.0.0, 7.1.0, 7.2.0 | 6.1.0, 6.1.3, 6.1.4, 6.1.6, 6.1.7, 6.1.8, 6.2.0, 6.2.3, 6.2.4, 6.2.5, 6.3.0, 6.3.1, 7.0.0, 7.2.0 |

There is no Gantt 6.1.0 upgrade guide — 6.1.0 has a what's-new entry and a changelog only. Reading list, Gantt rows (excerpt; sibling rows follow the same shape, `versions-support` omitted — it is never read; ✱ = changelog carries `[BREAKING]` entries):

| Version | Product | Upgrade guide | What's new | Changelog sections (✱ = BREAKING) |
|---------|---------|---------------|------------|-----------------------------------|
| 6.0.4 – 6.0.5 | Gantt | — | — | features-enhancements / demos, bug-fixes (listed, not read) |
| 6.0.6 | Gantt | — | — | **api-changes**, demos, bug-fixes |
| 6.1.0 | Gantt | — | `## Gantt v6.1.0` in `whats-new/6.0.0+` | features-enhancements, demos, bug-fixes |
| 6.1.6 | Gantt | `## Gantt v6.1.6` in `upgrades/6.0.0+` | `## Gantt v6.1.6` | features-enhancements, bug-fixes |
| 6.2.0 | Gantt | `## Gantt v6.2.0` | `## Gantt v6.2.0` | ✱ features-enhancements, **api-changes**, demos, bug-fixes |
| 6.3.0 | Gantt | `## Gantt v6.3.0` | `## Gantt v6.3.0` | features-enhancements, **api-changes**, locale-updates, demos, bug-fixes |
| 7.0.0 | Gantt | `upgrades/7.0.0` | `whats-new/7.0.0` | ✱ features-enhancements, **api-changes**, **styling-changes**, locale-updates, demos, bug-fixes |
| 7.0.1 – 7.0.2 | Gantt | — (Grid has 7.0.1 and 7.0.2 guides) | — | bug-fixes (listed, not read) |
| 7.1.0 | Gantt | `upgrades/7.1.0` ("Salesforce support") | — | features-enhancements, **styling-changes**, bug-fixes |
| 7.1.1 – 7.1.3 | Gantt | — | — | bug-fixes / demos (listed, not read) |
| 7.2.0 | Gantt | `upgrades/7.2.0` ("Prevent silencing resolved scheduling issues") | `whats-new/7.2.0` | features-enhancements, demos |
| 7.2.1 | Gantt | — | — | demos, bug-fixes (listed, not read) |

One action item from the plan, taken verbatim from the Gantt 7.0.0 upgrade guide and adapted to the customer's file:

````markdown
### 7.0.0 — Gantt
- [ ] **Rename project config `autoPostponedConflicts` → `autoPostponeConflicts`** · risk: deprecation (old name still works, deprecated)
  Source: https://bryntum.com/products/gantt/docs/guide/Gantt/upgrades/7.0.0 → "Project `autoPostponedConflicts` has been renamed"
  Files: `src/components/ProjectGantt.tsx`
  **Old code**
  ```tsx
  <BryntumGantt project={{ autoPostponedConflicts: true }} />
  ```
  **New code**
  ```tsx
  <BryntumGantt project={{ autoPostponeConflicts: true }} />
  ```
````

Other 7.0.0 guide headings the plan must answer for this app: Gantt — "Assignment field picker has changed to TabPanel", "Dragging all selected tasks by default", "Tracking project settings changes by default", "Resizable task editor", "Conflicts postponing related changes" (`hasPostponedOwnConstraintConflict` → `postponedConflict`); Grid — "New themes & styling changes", "FontAwesome Free no longer built in", "Individual animation related configs deprecated", "Container layout changes", "Mask mode config removed"; Scheduler — "The DependencyMenu feature is now enabled by default", "Dragged events now snap to resources by default", "Event styles changed". From 6.3.0: "Naming simplification for project data properties" (`tasksData` → `tasks`, …). The theme replacement lands under **Unknowns for the user** (old `gantt.stockholm.css` is gone; pick one of Svalbard, Stockholm, Visby, Material3, High Contrast in `-light`/`-dark`), and the kebab-case selector rename is handled by the 6 → 7 codemod dry-run before any manual CSS edits.
