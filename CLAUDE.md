# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Three static, self-contained HTML files that make up the Columbia Association (CA) Membership Marketing Dashboard. There is no build system, package manager, server, or test suite — each file is plain HTML/CSS/vanilla JS, deployed by being served/opened as-is (e.g. via GitHub Pages or similar static hosting).

- `index.html` — the main Membership Marketing Dashboard (leads pacing, CPL, channel/facility breakdowns, promotions calendar, UTM taxonomy reference).
- `sales.html` — the Membership Acquisition (sales) dashboard: current-month unit/cash PAR tracking, composition donuts, fiscal-year bar chart, rollforward/flow charts.
- `playbook.html` — "Membership Acquisition Operating Model": a static reference document (sidebar + scrollspy) explaining team structure, resource allocation (75/25 execution vs. build), the 3 campaign streams, Cat 0/1/2 project categories, and the 5-layer promotions framework. Pure content page, no data fetching.

There is no README, no CI, and no test/build tooling in this repo — treat each HTML file as the full unit of deployment.

## Working on this codebase

- **No build/lint/test commands exist.** To "run" any page, just open the `.html` file in a browser (or serve the directory with any static file server). Verify changes by opening the file directly and checking the console for fetch/render errors.
- Each file is a single monolithic document: `<style>` block, then body markup, then a single `<script>` block at the bottom with all JS. Keep edits localized to the relevant section rather than restructuring — these are hand-maintained dashboards, not a componentized app.
- CSS custom properties (`--navy`, `--gold`, `--green`, `--red`, etc.) are defined per-file in `:root` and reused throughout that file's markup/JS (e.g. color-coding PAR status as ahead/behind). `index.html` and `playbook.html` share the same navy/gold palette and Barlow + DM Mono font pairing; `sales.html` uses a similar but independently-defined palette — check the right file's `:root` before adding colors.
- Fonts (Barlow, Barlow Condensed, DM Mono) and Chart.js (only used in `index.html`) are loaded from CDNs (Google Fonts, cdnjs) via `<link>`/`<script>` tags in `<head>` — there's no bundling, so any new library must be added the same way.
- `sales.html` builds its own SVG donut/bar/line charts by hand (`buildDonut`, `fy-bars`, `rf-bars`, etc.) rather than using a charting library; `index.html` uses Chart.js (`charts['name'] = new Chart(...)`, with a `dc('name')` helper to destroy/recreate a chart before redrawing it).

## Data architecture (important — read before touching data logic)

Both `index.html` and `sales.html` pull live data client-side on page load and fall back to hardcoded demo/placeholder data if the fetch fails — there is no backend in this repo.

**`index.html`**:
- Fetches a single published Google Sheet as CSV (`SHEET_URL`, a `docs.google.com/.../pub?output=csv` link) in `fetchAndInit()`.
- The sheet is one CSV with multiple logical tables concatenated vertically, each preceded by a `SECTION N ...` marker row (e.g. `SECTION 3` = current-month meta/KPIs, `SECTION 10` = current-month spend, `SECTION 12` = fiscal year, `SECTION 16` = rolling CPL). `findSection(rows, n)` locates a marker; `getSectionRows(rows, sectionIdx)` reads rows until the next `SECTION` marker or blank row, skipping known header/note rows (see the `skip` list in `getSectionRows`).
- `parseSummaries(rows)` walks each section in turn and builds the `LD` object, which is `Object.assign`'d into the global `D` state object (`D.meta`, `D.daily_all`, `D.rolling_12`, `D.fy`, `D.cpl_daily`, `D.spend_mtd`, `D.promo_months`, etc.).
- If the fetch fails, the hardcoded `D` object already declared in the script (with 2026-06/demo values) is used as-is and a warning banner (`showBanner`) is shown — this is intentional graceful degradation, not a bug.
- **When adding a new data field**: add it to the correct `SECTION N` block in the Google Sheet, extend the corresponding parsing block in `parseSummaries`, add a matching default in the hardcoded `D` fallback object, and wire it into the relevant `build*View()` function.
- The month picker, view tabs (`switchView`), and FY tabs (`switchFY`) all re-render from `D`/`selectedYM` rather than re-fetching — data is fetched once per page load.

**`sales.html`**:
- Fetches four separate CSV "sheets" (`members`, `goals`, `rollforward`, `pricing`) in parallel via a Cloudflare Worker proxy (`DATA_API = 'https://ca-data.timothy-pinel.workers.dev/?sheet='`) in `loadData()`, each parsed by the generic `parseCSV` (header-row → array-of-objects), not the section-marker format `index.html` uses.
- "Current month" is **not** wall-clock time — `setDateFromData()` derives `CURRENT_MONTH`/`CURRENT_DAY`/`CURRENT_YEAR` from the latest `date` found in the `members` sheet, since the dashboard should reflect data currency, not today's date.
- `getRolling3()` computes the 3 completed months immediately preceding `CURRENT_MONTH` dynamically (handles year wraparound).
- Membership revenue is estimated via `getPrice()` / `PRICING` lookup keyed by `type|residency|size`, not from actual transaction amounts.

**`playbook.html`** has no data fetching — all content is static markup. Its only script is a scrollspy `IntersectionObserver` that highlights the active sidebar link.

## Cross-file conventions

- `index.html` links to `sales.html` (top-right nav) and to `playbook.html` via the "Marketing Plans" dropdown (pinned "Operating Model" link); `sales.html`/`playbook.html` link back to `index.html`. Keep these relative links intact if renaming files.
- The `MARKETING_PLANS` array near the top of `index.html`'s script is a hand-maintained list of monthly plan links (Notion URLs) shown in the "Marketing Plans" nav dropdown — update this array (and flip `status: 'draft'` → `'published'`) when a new monthly plan is published; don't wire this to any data source.
- UTM taxonomy (`UTM_TAXONOMY` in `index.html`) and channel/category color maps (`CHANNEL_COLORS`, `CATEGORY_TAG_COLORS`) are the canonical source-of-truth reference for lead attribution — keep these in sync with actual UTM tagging conventions if channels are added/deprecated (see the "Usage Notes" callouts in the UTM Taxonomy view for deprecation status of channels like `prospect_wizard`).
- Fiscal year runs **May 1 – April 30** (not calendar year) — reflected in both `index.html`'s FY view/tabs (`FY2027`/`FY2026`) and `sales.html`'s `FY_MONTHS`/`FY_QUARTERS` ordering. Any FY-related logic must preserve this month ordering.
- PAR ("pace-adjusted rate"/linear pacing) is the recurring cross-dashboard concept: `goal × (elapsed time / total time)`, used to judge whether actuals are ahead/behind pace for the month. It's computed independently in each file's own JS rather than shared.
