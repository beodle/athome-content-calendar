# CLAUDE.md — 앳홈 콘텐츠 캘린더

This file describes the codebase structure, conventions, and workflows for AI assistants working on this repository.

---

## Project Overview

**앳홈 콘텐츠 캘린더** (ATHOME Content Calendar) is a static, single-page web application that visualizes ATHOME's content publishing schedule. It reads data from a Google Sheets CSV export and renders a monthly calendar view with channel filtering.

- **Type**: Static SPA — no build step, no framework, no package manager
- **Language**: Vanilla JavaScript + HTML + CSS
- **Audience**: Korean-language interface targeting internal use
- **Channels tracked**: LinkedIn (링크드인) and Blog (블로그)

---

## Repository Structure

```
athome-content-calendar/
├── index.html          # Entire application (HTML + embedded CSS + embedded JS)
└── athome-tokens.css   # ATHOME corporate design system tokens (shared across projects)
```

This is an intentionally minimal repository. There is no `package.json`, no build toolchain, no test framework, and no configuration files. All logic lives in `index.html`.

---

## Running Locally

Because the app fetches from Google Sheets (cross-origin), it must be served over HTTP — opening `index.html` directly as a `file://` URL will trigger CORS errors.

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

No installation step is required.

---

## Architecture

### Data Flow

1. On `DOMContentLoaded`, `loadEvents()` fetches a public CSV export from Google Sheets
2. [PapaParse](https://www.papaparse.com/) (loaded via CDN) parses the CSV into row objects
3. Rows are mapped to a normalized event shape and stored in `allEvents[]`
4. `renderAll()` calls `renderCalendar()` and `renderUpcoming()` to build the DOM

### Key State Variables

| Variable | Type | Purpose |
|---|---|---|
| `currentYear` | `number` | Year being displayed |
| `currentMonth` | `number` | Month being displayed (0-indexed) |
| `allEvents` | `array` | All parsed events from Google Sheets |
| `activeFilter` | `string` | Channel filter: `'all'`, `'링크드인'`, or `'블로그'` |

### Event Object Shape

CSV columns map to this object (column headers are in Korean):

```javascript
{
  date:    string,   // "YYYY-MM-DD" from column '시기'; empty string if unscheduled
  title:   string,   // Content topic from column '주요 주제'
  channel: string,   // '링크드인' or '블로그' from column '채널'
  note:    string,   // Tooltip content from column '비고'
  url:     string,   // Link from column 'URL', 'url', or '링크' (tries all three)
}
```

### Render Functions

| Function | Purpose |
|---|---|
| `renderAll()` | Orchestrator — calls calendar + upcoming stats |
| `renderCalendar(year, month, events)` | Builds the 7-column CSS grid calendar |
| `renderUpcoming(events)` | Renders the summary stat cards (total / LinkedIn / Blog counts) |
| `makeCard(ev)` | Creates a single event card DOM element |
| `getThisWeekRange()` | Returns `{ mon, sun }` Date objects for current-week highlighting |
| `loadEvents()` | Async: fetches CSV and returns normalized event array |

### Google Sheets Configuration

The data source is hardcoded in `index.html`:

```javascript
const SHEET_CSV_URL =
  'https://docs.google.com/spreadsheets/d/1bnLQypIuMb0gw0RxzwYVkqGBdN8KSA4_TXSuogaOWak/export?format=csv&gid=658998951';
```

The linked spreadsheet must be **publicly accessible** (published to web or shared with "anyone with the link can view") for the CSV export to work without authentication.

---

## Design System — `athome-tokens.css`

This file is a **shared corporate design system** used across multiple ATHOME projects. It provides CSS custom properties (variables) derived from the company's visual identity.

### Key Tokens

**Colors:**
```css
--color-bg:           #E3E4E0;  /* cool gray page background */
--color-bg-white:     #FFFFFF;  /* card/content background */
--color-text-primary: #0D0D0D;  /* body text */
--color-accent:       #FF5D00;  /* orange — primary highlight/KPI color */
--color-dark:         #1A1A1A;  /* dark sections, LinkedIn card background */
--color-gray-mid:     #CCCCCC;  /* borders, secondary text */
--color-gray-light:   #F2F2F0;  /* card backgrounds */
```

**Typography:**
```css
--font-display:  'Neue Haas Grotesk', ...;   /* headings, titles */
--font-body-ko:  'Pretendard Variable', ...;  /* Korean body text */
--size-body:     15px;
--size-caption:  11px;
--weight-bold:   700;
```

**Layout:**
```css
--space-page-x:  clamp(40px, 5vw, 80px);  /* horizontal page padding */
--space-page-y:  clamp(32px, 4vh, 60px);  /* vertical page padding */
--radius-card:   12px;
--border-divider: 1px solid #0D0D0D;
```

**Utility classes provided:** `.text-accent`, `.text-caption`, `.bg-gray`, `.bg-dark`, `.bg-cool-gray`, `.border-top`, `.border-bottom`, `.page-x`, `.page-y`

When modifying visual styles in `index.html`, prefer using these tokens over hardcoded values. When adding new color or spacing values, check if an existing token fits before introducing new ones.

---

## CSS Conventions in `index.html`

CSS is embedded in a `<style>` block inside `<head>`. Key conventions:

- **Naming**: BEM-influenced kebab-case — `.cal-grid`, `.event-card`, `.btn-filter`, `.site-header`
- **Sections**: Separated by Korean comment headers in the format `/* ── 섹션명 ── */`
- **Tokens**: Use `var(--token-name)` for colors, spacing, and typography wherever tokens exist
- **Event card colors**: LinkedIn cards use `var(--color-dark)` (dark); Blog cards use `var(--color-accent)` (orange)
- **State classes**: `.today`, `.this-week`, `.other-month`, `.past`, `.has-url`, `.active`

---

## JavaScript Conventions in `index.html`

JavaScript is embedded in a `<script>` block before `</body>`. Key conventions:

- **No framework** — plain DOM manipulation via `document.createElement`, `appendChild`, `innerHTML`
- **Async/await** for data fetching (no `.then()` chains)
- **Section comments**: Use `/* ── 기능명 ── */` to separate logical sections
- **Date handling**: Dates stored as `"YYYY-MM-DD"` strings; parsed with `new Date(dateStr)` when comparison is needed
- **Past event detection**: An event is "past" if its date is before today at midnight (time zeroed with `setHours(0,0,0,0)`)
- **Filter logic**: Always apply `activeFilter` inside render functions, not at load time — `allEvents` always holds the full unfiltered set

---

## External Dependencies (CDN only)

| Library | Version | Purpose |
|---|---|---|
| [PapaParse](https://www.papaparse.com/) | 5.4.1 | CSV parsing |
| [Pretendard Variable](https://github.com/orioncactus/pretendard) | 1.3.9 | Korean web font |

Both are loaded via `cdn.jsdelivr.net`. There is no `package.json` or lock file — dependency versions are pinned directly in the HTML.

---

## Commit History Context

| Commit | Summary |
|---|---|
| `e1e1a7a` | Initial project — full content calendar SPA |
| `5509445` | Updated accent color to `#FF5D00` |
| `57dd3ab` | Removed unscheduled items section |
| `3874d7d` | Reverted removal of unscheduled section |
| `3c7dc7a` | Removed unscheduled items section again (current HEAD) |

The `renderUnscheduled()` function still exists in the JavaScript (it references a `#unscheduled-list` element that is no longer in the HTML). This is dead code — the unscheduled section has been intentionally removed from the UI.

---

## What to Avoid

- **Do not introduce a build system** unless explicitly requested — the simplicity is intentional
- **Do not add frameworks** (React, Vue, etc.) without explicit direction
- **Do not inline new hardcoded colors** — use existing `--color-*` tokens from `athome-tokens.css`
- **Do not modify `athome-tokens.css` casually** — it is a shared file used across other ATHOME projects
- **Do not break the monolithic structure** — keeping everything in `index.html` is a deliberate choice for this project's deployment context
- **Do not add environment variable handling** — there are no secrets; the Google Sheets URL is intentionally public

---

## Extending the Application

### Adding a new channel

1. Add a new `.btn-filter` button in the HTML filter group with the appropriate `data-filter` value (Korean channel name)
2. Add a CSS class for the new channel's card color
3. In `makeCard()`, update the `cls` logic to handle the new channel name
4. Add a stat card in `renderUpcoming()` for the new channel count

### Changing the data source

Update `SHEET_CSV_URL` in the `<script>` block. The sheet must be publicly accessible and have the same column headers (시기, 주요 주제, 채널, 비고, URL).

### Adding new event card fields

Add a new property to the mapping in `loadEvents()`, then render it in `makeCard()`.
