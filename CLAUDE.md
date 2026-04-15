# CLAUDE.md — Predictive-par

This file documents the codebase structure, development conventions, and workflows for AI assistants working in this repository.

---

## Project Overview

**Predictive-par** is a single-file HTML5 Progressive Web App (PWA) used by FOH (Front-of-House) kitchen staff at **Lazy Acres Natural Market #1827, Mission Hills** to manage daily food preparation inventory pars.

- **Version:** v6.12 (embedded in print footer)
- **Target users:** Restaurant prep kitchen staff (mobile-first, touchscreen use)
- **Stack:** Vanilla HTML5 + CSS3 + JavaScript — no frameworks, no build tools, no dependencies
- **Entry point:** `index.html` (2,043 lines; entire application lives here)

The app calculates how much of each food item needs to be prepared each day by comparing on-hand inventory (entered by staff) against target par levels stored in the data layer.

---

## Repository Structure

```
Predictive-par/
├── index.html      # The entire application — HTML, CSS, and JS in one file
├── README.md       # Minimal placeholder
└── CLAUDE.md       # This file
```

There are no external dependencies, package manifests, build scripts, or test files.

---

## Application Architecture

`index.html` is organized into logical sections (not separate files):

| Lines | Section | Description |
|-------|---------|-------------|
| 1–14 | `<head>` / PWA meta | PWA manifest tags, viewport, title |
| 14–572 | `<style>` | All CSS — custom properties, layout, component styles, print styles |
| 573–1015 | `PAR_DATA` | Target par levels per item per day of week |
| 1016–1459 | `DAILY_AVG` | 3-day rolling usage averages per item per day |
| 1460–1482 | `STATIONS`, `STATION_ICONS`, item sets | Menu structure, icons, special item classifications |
| 1483–1561 | State & helpers | Global vars, `sessionStorage` functions, tier/sort helpers |
| 1562–1633 | Rendering helpers | `renderDayNav()`, `selectDay()`, toggle functions |
| 1634–1761 | `renderContent()` | Full UI render (both view and prep modes) |
| 1762–1969 | Voice input module | Aliases, fuzzy matching, Speech API wrapper |
| 1971–1976 | Init block | App startup (first `<script>` tag ends here) |
| 1978–2043 | Print module | `printList()` function in a second `<script>` tag |

---

## Data Layer

All data is embedded as JavaScript constants in `index.html`.

### `PAR_DATA` (line 573)
Nested object of target inventory levels:
```js
PAR_DATA[day][itemName] → number (float)
```
Keys are full day names (`"Monday"` … `"Sunday"`). Values are floated par levels for ~60 food items. Par is always ceiling'd at render time: `Math.ceil(PAR_DATA[day][item])`. Exception: `"Burritos"` is always fixed at `12`.

### `DAILY_AVG` (line 1016)
Same structure as `PAR_DATA` but holds 3-day rolling average usage:
```js
DAILY_AVG[day][itemName] → number
```
Displayed in view mode as a reference alongside the par level.

### `STATIONS` (line 1460)
Defines kitchen station groupings and item order:
```js
STATIONS = {
  "Breakfast": [...items],
  "Wraps & Sandwiches": [...items],
  "Protein Salads": [...items],
  "Starch & Grain Salads": [...items],
  "24oz Salads": [...items],
  "End Cap": [...items],
  "Soups": [...items]
}
```

### Special Item Sets (lines 1475–1477)
```js
FLAGGED_ITEMS   // Set — items needing UPC verification (currently: "Matzo Ball Soup")
NEAR_ZERO_ITEMS // Set — near-zero movement items ("Edamame Udon Soup", "Heart Healthy Turkey Chili")
PROXY_ITEMS     // Set — 24oz proxy/special items ("Spinach Pasta Salad", "Quinoa Detox Salad", "Kale Recharge Salad")
```

### Day/Coverage Constants (line 1479)
```js
DOW       // ["Monday", ..., "Sunday"]
DOW_SHORT // ["Mon", ..., "Sun"]
DOW_COV   // {"Monday": "Mon → Wed", ...}  — 3-day coverage windows shown in subtitle
```

---

## Core Algorithms

### Tier Classification — `getTier(par, item)` (line 1507)
Classifies each item into a display tier used for sorting and color-coding:

| Tier | Condition | Color |
|------|-----------|-------|
| `"F"` | item === "Burritos" (fixed par 12) | Purple |
| `"ZERO"` | item in `NEAR_ZERO_ITEMS` | Gray muted |
| `"FLAG"` | item in `FLAGGED_ITEMS` | Warning / amber |
| `"H"` | par >= 10 (High) | Green |
| `"M"` | par >= 4 (Medium) | Orange |
| `"L"` | par < 4 (Low) | Gray |

Tier order for sorting: H → M → L → F → FLAG → ZERO (defined in `sortByTierAndPar`, line 1518).

### Prep Mode Need Calculation — `updateNeedCell(item)` (line 1617)
```
par  = item === "Burritos" ? 12 : Math.ceil(PAR_DATA[currentDay][item])
need = par - haveValues[item]
```
Display:
- `+N` (green) — need to make more
- `-N` (red) — overstocked
- `✓` (gray) — exactly on par
- `—` (empty) — no "have" value entered yet

### Voice Matching — `matchVoiceItem(phrase)` (line 1835)
Three-tier priority matching against `VOICE_ALIASES` (line 1762):
1. **Direct alias match** — exact key in `VOICE_ALIASES`
2. **Substring alias match** — alias is substring of spoken phrase (longer aliases checked first)
3. **Fuzzy word-overlap score** — 4pts exact word match, 2pts prefix match, 1pt contains match; returns best score ≥ 4

---

## State Management

### Global Variables (lines 1483–1486)
```js
let currentDay = null;           // Active day string ("Monday"…"Sunday")
let prepMode   = false;          // Toggle: view mode vs. prep input mode
const collapsedStations = new Set(); // Station names currently collapsed
const haveValues = {};           // {itemName: number} — current on-hand counts
```

### sessionStorage Persistence (lines 1488–1504)
Data persists for the browser session only (cleared on close or via "Clear" button).

- **Key format:** `"foh_have_" + day + "_" + itemName`
- **Helper:** `ssKey(day, item)` — generates the storage key
- **Load:** `loadHave(day)` — called on day switch; repopulates `haveValues` from storage
- **Save:** inside `onHaveInput(item, val)` — writes to `haveValues` and `sessionStorage` simultaneously
- **Clear:** `clearAll()` — removes all keys for `currentDay` from both `haveValues` and `sessionStorage`

---

## UI Rendering

### `renderDayNav()` (line 1562)
Rebuilds the day-of-week pill selector. Called on day change. Active pill uses the `active` CSS class.

### `selectDay(day)` (line 1568)
Main day-switch handler:
1. Sets `currentDay`
2. Calls `loadHave(day)`
3. Calls `renderDayNav()`
4. Calls `renderContent()`
5. Updates coverage label

### `renderContent()` (line 1634)
Regenerates the entire station/item grid. Behavior differs by `prepMode`:

**View mode (`prepMode === false`):**
- Shows par level and 3-day average for each item
- Items grouped by station, sorted by tier then descending par
- Collapsible station headers

**Prep mode (`prepMode === true`):**
- Shows a numeric input for "have" quantity, plus a calculated "need" cell
- Input fields get `enterkeyhint="next"` for iOS keyboard UX
- Enter key advances to the next input via `_prepInputOrder`

### `updateNeedCell(item)` (line 1617)
Surgically updates a single need cell without a full re-render. Called after every `onHaveInput()`.

---

## Voice Input Module (lines 1762–1969)

**Only available in Prep mode.** Uses the browser `SpeechRecognition` API (Web Speech API).

- **`VOICE_ALIASES`** (line 1762) — hand-curated map of spoken phrases → canonical item names (e.g., `"noodles" → "Sesame Noodles"`)
- **`startVoice()` / `stopVoice()`** (lines 1911, 1933) — toggle microphone; language hardcoded to `'en-US'`
- **`processVoiceResult(transcript)`** (line 1939) — parses transcript, matches items, calls `onHaveInput()`
- **`parseVoiceTranscript(transcript)`** (line 1881) — tokenizes spoken text; converts written numbers ("twenty-five") to digits before matching
- Results are shown via `showVoiceToast()` (line 1960) — a non-blocking overlay toast

---

## Print Module (lines 1980–2043)

`printList()` generates a print-optimized view written to `<div id="print-view">`. The CSS `@media print` rule hides the nav bar and formats for letter-size paper using CSS Grid multicolumn layout. Soups station is excluded from the print view (`if (stn === "Soups") continue`).

---

## Code Conventions

| Convention | Rule |
|-----------|------|
| **Indentation** | 2 spaces |
| **JS naming** | `camelCase` for functions and variables |
| **CSS naming** | `kebab-case` for classes |
| **Data constants** | `ALL_CAPS` object names (`PAR_DATA`, `DAILY_AVG`, `STATIONS`, etc.) |
| **CSS section comments** | `/* ── Section Name ─── */` |
| **Inline UX explanations** | Numbered circle markers: `<!-- ① comment -->` or `// ① comment` |
| **CSS custom properties** | All design tokens defined in `:root` (lines 17–36): `--brand`, `--accent`, `--bg`, `--card`, `--green`, `--orange`, `--red`, `--purple`, `--gray`, etc. |
| **No semicolons in CSS** | N/A — standard CSS semicolons used |
| **Error handling** | `try { … } catch(e) {}` wraps all `sessionStorage` calls silently |

---

## Development Workflow

### Running Locally
No build step required. Open `index.html` directly in a browser, or serve it:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

### Browser Requirements
- ES6+ (arrow functions, template literals, `Set`, `Map`, destructuring)
- `sessionStorage` API
- `SpeechRecognition` API (optional — voice input degrades gracefully without it)
- CSS Grid and `backdrop-filter` support (modern browsers)

### Testing
There is no automated test suite. Testing is manual:
1. Open the app in a browser
2. Select different days and verify par/avg values render correctly
3. Enter "Have" values in Prep mode and confirm "Need" calculations
4. Use the "Print" button to verify print layout
5. Test on a mobile device or browser DevTools mobile emulation (primary use case)

### PWA Installation
The app includes PWA meta tags for iOS "Add to Home Screen":
- `apple-mobile-web-app-capable` — fullscreen mode
- `apple-mobile-web-app-status-bar-style` — status bar styling
- SVG app icon embedded inline in `<link rel="apple-touch-icon">`

---

## Git Workflow

### Branches
- `main` — stable production code
- `claude/add-claude-documentation-djOje` — documentation feature branch (this branch)

### Commit Message Style
Previous commits use descriptive imperative messages:
```
Add files via upload
Rename FOH_Prep_Pars_v6_12.html to index.html
docs: add CLAUDE.md with codebase documentation
```

### Version Tracking
App version is tracked manually:
- In the print footer: `"FOH Predictive Par System v6.12"`
- Historically via filename: `FOH_Prep_Pars_v6_9.html` → `v6_10` → `v6_11` → `v6_12` → `index.html`

---

## Making Changes

### Editing Data (PAR_DATA / DAILY_AVG)
- `PAR_DATA` starts at line 573, `DAILY_AVG` at line 1016
- Both are plain JS object literals — edit values in place
- Float values are fine; the UI always applies `Math.ceil()` at render time
- Burritos par is hardcoded to `12` in the render logic, not from `PAR_DATA`

### Adding a New Item
1. Add the item name string to the relevant station array in `STATIONS` (line 1460)
2. Add a par entry for all 7 days in `PAR_DATA` (line 573)
3. Add a daily average entry for all 7 days in `DAILY_AVG` (line 1016)
4. Optionally add voice aliases in `VOICE_ALIASES` (line 1762)
5. If special classification needed, add to `FLAGGED_ITEMS`, `NEAR_ZERO_ITEMS`, or `PROXY_ITEMS` (lines 1475–1477)

### Adding a New Station
1. Add a key + item array to `STATIONS` (line 1460)
2. Add an emoji icon to `STATION_ICONS` (line 1470)
3. Ensure all items have entries in `PAR_DATA` and `DAILY_AVG`

### Modifying UI Styles
All styles are in the `<style>` block (lines 14–572). CSS custom properties in `:root` (lines 17–36) control colors and radius — prefer editing those over hardcoding values.

### Print Layout Changes
Print styles begin around the `@media print` block within the main `<style>` section. The `printList()` function (line 1981) controls what data appears; the CSS controls layout.

---

## Known Constraints

- **No backend:** Data exists only in `sessionStorage` — cleared when the browser/tab is closed
- **No persistence across devices:** No database or API; each device session is independent
- **No test suite:** All verification is manual
- **No linting or formatting tools:** Code style is maintained manually
- **Monolithic file:** All code in one 2,000+ line file — search carefully before adding code to avoid duplicate logic
- **No module system:** All functions and variables are in global scope; avoid name collisions
- **Voice input browser support:** `SpeechRecognition` is a Chrome/Edge API; it is unavailable in Firefox and Safari (the app handles this gracefully)
