# WIW Visualizer

Single-file, client-side dashboard that turns "When I Work" (WIW) Excel exports into an
interactive staffing schedule. No build step, no backend — open `index.html`
directly in a browser.

**Keep it a single file.** Portability (email it, drop it on a USB stick, open it with
zero setup) is a core requirement, not an oversight. Do not split this into separate
JS/CSS files or add a build step unless explicitly asked.

## Stack (all via CDN, no package.json)

- React 18 + ReactDOM (UMD dev builds)
- Babel Standalone — compiles the in-browser `<script type="text/babel">` JSX at page load
- Tailwind CSS via the play CDN (`cdn.tailwindcss.com`)
- SheetJS (`xlsx`) — parses `.xlsx` in-browser
- Phosphor Icons web font (`ph-*` / `ph-fill ph-*` / `ph-bold ph-*` classes)

Everything lives in one `<script type="text/babel">` block starting around line 100.
There is no `npm install` / dev server — editing is just editing the HTML file and
reloading it in a browser.

## File map (line numbers as of initial context pass, will drift as edits land)

- `1–99`: `<head>` — CDN script tags, Tailwind, custom CSS (`.glass-panel`, `.glass-header`,
  custom scrollbar, `grid-accordion` expand/collapse trick, slide-up animation).
- `102–109`: `GUIDE_IMAGES` — base64 `data:` URIs for the How-To guide's screenshots
  (exported from When I Work). Images are embedded inline, not linked as separate files,
  to preserve single-file portability. Source PNGs are kept in
  `docs/screenshots/guide/` and `docs/screenshots/` for regenerating this block if the
  guide's screenshots ever need updating (re-encode with
  `base64 -w0 <file>.png` and paste into the relevant `GUIDE_IMAGES` entry).
- `111–247`: `GUIDE_STEPS` (step content) and `HowToGuide` (the modal component) — the
  first-visit walkthrough, reopenable via the header's "How to Use" button.
- `249–259`: `Toast` component.
- `261–...`: `App` — state, file upload/drag-drop handlers, `processFiles` (the
  XLSX → dashboard data pipeline).
- `419–512`: filter/search state + `filteredEvents`, `dailyViewData`,
  `categoricalViewData` memos.
- `515–632`: `StaffRow` — one roster row (assigned staff or an open-shift placeholder).
- `634–730`: `EventCard` — one site/event card (staffing bar, notes, roster).
- `733–1120`: `App`'s JSX return — sticky header/uploader, filter bar, Daily View,
  Categories View.
- `1123–1153`: scroll listener outside React that hides/shows the sticky header and
  repositions the filter bar and date headers on scroll (manipulates DOM directly by
  element id — `main-header`, `filter-bar`, `date-header-{i}`).

## Data model

**Input files** (both `.xlsx`, user-provided, never bundled):
- *Schedule export*: one or more sheets named `Schedules - <name>`. Rows have columns
  like `First Name`, `Last Name`, `Position`, `Site`, `Shift Start Date`,
  `Shift Start Time`, `Shift End Time`, `Notes`, `OpenShift Count`.
- *Users roster*: sheet with `First Name`, `Last Name`, `Phone Number`, `Email`,
  `Positions`.

**Pipeline** (`processFiles`, `index.html:241`):
1. Parse Users sheet → `usersMap` keyed by `"firstname lastname"` (lowercase).
2. Parse every `Schedules - *` sheet → flat `allShifts` array, tagged with
   `_sourceSchedule`.
3. For each shift row: look up the staff member in `usersMap`, normalize the date
   (Excel serial number OR `YYYY-MM-DD` string OR other), classify as an **open
   shift** (`!fname || openShiftCount > 0`) vs **assigned**, and bucket into an
   `eventsMap` keyed by `` `${date}::${site}` ``.
4. `isAccount` flag is set when the source schedule name matches one of the hardcoded
   `accountKeywords` (currently `matsuhisa`, `eddie`, `capital`, `mojo`, `hudson`) — this
   drives the Categories view split between "Main Parties" and "Corporate Accounts".
   **This keyword list is a business rule, not a generic feature — update it in place
   when accounts are added/renamed rather than generalizing it into config unless asked.**
5. New schedule upload **replaces** events for any date present in the new file, and
   **preserves** events on dates not touched by the upload (`currentEvents.filter(e =>
   !newDatesSet.has(e.date))` then concat). A users-only upload re-maps phone/email/roles
   onto existing events without touching event/shift data.
6. Sets (`allTags`) get flattened to `allTagsArray` before being pushed into
   `parsedData` because `localStorage`/`JSON.stringify` can't round-trip a `Set`.

**Persisted state (localStorage keys)**:
- `wiw_dashboard_data` — `{ events, availableFilters, usersMap }`, the whole dashboard.
- `wiw_staff_notes` — private per-shift notes, keyed by
  `` `${eventId}::${staff.fullName}::${staff.startTime}` `` (see `StaffRow`,
  `index.html:539`). Note this key silently breaks if a staff member's start
  time changes between syncs (their old note becomes orphaned under the old key).
- `wiw_guide_seen` — presence alone (any value) means the How-To guide has been
  dismissed once and won't auto-open on load again; the header's "How to Use" button
  reopens it manually regardless of this flag.

## View modes

- **Daily** (`viewMode === 'daily'`): events grouped by date, sorted chronologically,
  collapsible per-day accordions, "Fully Staffed" badge when no open shifts that day.
  Past days hidden by default (`showPastEvents` toggle).
- **Categories** (`viewMode === 'categories'`): events split into Main Parties vs
  Corporate Accounts via `isAccount`, each independently collapsible.

Both share `EventCard` for rendering individual events and `filteredEvents` (search +
active role/schedule filter pills) as their common data source.

## Conventions / style already in place

- Dark "Liquid Glass" theme: zinc-950 background, yellow-400 accent, glass-panel
  blur+border utility classes defined in `<style>`. Keep new UI consistent with this
  palette (zinc/yellow, rose for warnings/open-shifts, emerald for fully-staffed/success,
  blue for corporate accounts).
- Phosphor icon classes: `ph-fill ph-*` for filled/solid icons, `ph-bold ph-*` for
  bold-weight utility icons (carets, plus, x), plain `ph ph-*` sparingly.
- No external state management — plain `useState`/`useMemo`/`useEffect`, no context, no
  router. Keep additions consistent with this (don't introduce Redux/Zustand/etc.).
- No semicolon-less style debates needed — existing code uses semicolons throughout;
  match it.
- Tailwind utility classes inline, no separate CSS files aside from the `<style>` block
  for things Tailwind can't express (backdrop blur combos, custom scrollbar, the
  grid-accordion trick, keyframes).

## Known rough edges (context for future changes, not necessarily bugs to fix unasked)

- `removeFile()` (`index.html:227`) resets `parsedData` to
  `{ events: [], availableFilters: [] }` — missing `usersMap`, which will make
  `Object.keys(parsedData.usersMap)` throw if this path is hit after data already
  exists. Worth fixing together with any nearby upload-flow change.
- Date parsing (`index.html:300–318`) handles Excel serial numbers and
  `YYYY-MM-DD` strings explicitly; anything else falls through to `new Date(rawDate)`
  which is locale/format fragile.
- The scroll-driven header/filter-bar repositioning (`index.html:1123–1150`)
  hardcodes pixel offsets per breakpoint (`160px`/`200px`, `100px`/`120px`) rather than
  reading actual element heights — if header content changes height, these need manual
  re-tuning.
- `accountKeywords` is a hardcoded array of lowercase substrings, not user-configurable
  in the UI.

## Working in this repo

- Git repo, hosted at `JaysValetSS/wiw-list` on GitHub and served live via GitHub Pages
  (repo is public — required for free Pages hosting; private Pages needs a paid plan).
- `index.html` is the only file that matters at runtime — it's both the app and the page
  GitHub Pages serves at the repo root. Keep it that name; don't reintroduce a
  differently-named copy.
- Commits here are made under the `natesheridan` GitHub identity (day-to-day changes);
  `jaysvalet`/JaysValetSS is the org-owner/admin account, not the one pushing routine work.
