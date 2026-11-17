# HUJI Student Exchange Platform — Project Reference

> **Last updated:** 2026-11-17  
> **Repository:** https://github.com/OmerGit2000/huji-exchange-11-2026.git  
> **Branch:** `main`  
> **Deployed as:** Static single-page HTML (GitHub Pages or any static host)

---

## 1. What This Project Is

A **fully self-contained, single-file web application** (`index.html`, ~508 KB, ~5,900 lines) that helps **Hebrew University of Jerusalem (HUJI)** students explore, compare, filter, and rank **125 international exchange destinations** for the **2027/2028 academic year**.

It is designed as an enhanced, student-friendly alternative to the official [Mobility Online portal](https://www.service4mobility.com/europe/PortalServlet?identifier=JERUSALEM01), adding features like personal ratings, top-3 preference selection, cost of living data, Jewish community info, QS world rankings, interactive maps, and cross-device sync via QR codes.

---

## 2. File Structure

```
Student exchange/
├── index.html                          # THE ENTIRE APP (HTML + CSS + JS + data)
├── HUJI_PEP_Exchange_Options.csv       # Source data (reference only, not used at runtime)
├── HUJI_PEP_Exchange_Options.xlsx      # Source data (reference only)
├── .gitignore                          # Prevents accidental secret leaks
├── Partner_Factsheets/                 # ~90 PDF fact sheets from partner universities
│   ├── Ca__Foscari_University_of_*.pdf
│   ├── Charles_University_*.pdf
│   └── ... (one per partner)
└── scratch/                            # Temporary dev scripts (NOT part of the app)
```

### Why everything is in one file

The app is intentionally a **single `index.html`** with no build step, no server, no bundler. This makes it trivially deployable (drag-and-drop to GitHub Pages) and easy for non-technical users to host. All CSS is in `<style>` tags, all JS is in `<script>` tags, and the destination data is an inline JSON array.

---

## 3. Architecture Overview

### 3.1 Data Layer

All 125 destinations live in a single JS constant near line ~2754:

```js
const RAW_DESTINATIONS = [ { id: "dest_1", name: "...", ... }, ... ];
```

Each destination object has **30 fields**:

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique ID like `dest_1` |
| `name` | string | University name |
| `city` | string | City |
| `country` | string | Country (26 unique) |
| `address` | string | Full address |
| `study_levels` | string[] | `["Bachelor", "Master", "Doctoral"]` subset |
| `programs` | string[] | Exchange program types (see §4.1) |
| `study_fields` | string[] | 87 possible study fields from Mobility Online |
| `academic_years` | string[] | e.g. `["2027/2028", "2026/2027", ...]` |
| `available_spots` | string | Human-readable quota description |
| `calendar_start` | string | Semester start info |
| `calendar_end` | string | Semester end info |
| `start_month` | string | Normalized: `"August"`, `"September"`, `"October"` |
| `end_month` | string | Normalized: `"December"`, `"January"`, `"February"` |
| `orientation_date` | string | Orientation date text |
| `exam_details` | string | Exam period info |
| `rank_info` | string | QS subject rankings summary |
| `living_cost_usd` | number | Monthly cost in USD |
| `living_cost_local` | string | Monthly cost in local currency |
| `subsidies_university` | string | Available subsidies |
| `scholarships_country` | string | Country-level scholarships |
| `has_jewish_life` | boolean | Whether city has Jewish community |
| `jewish_life_details` | string | Synagogues, kosher dining, etc. |
| `factsheet_file` | string | Path to PDF in `Partner_Factsheets/` |
| `factsheet_url` | string | External factsheet URL |
| `website` | string | University website |
| `user_rating` | string | Default rating: `"unrated"` |
| `notes` | string | Additional info about the destination |
| `general_rank` | string | QS World rank |
| `subject_ranks` | object | Per-field rankings, e.g. `{"Economics": "#51-100"}` |

### 3.2 State Management

Client-side state is stored in simple `let` variables (no framework):

| Variable | Type | Purpose |
|---|---|---|
| `activeYear` | string | Selected academic year (default: `"2027/2028"`) |
| `activeSemester` | string | Selected semester filter |
| `activeProgram` | string | Selected exchange program |
| `activeCountry` | string | Selected country |
| `activeLevelFilter` | string | `"Bachelor"`, `"Master"`, or `"Doctoral"` |
| `activeStartMonth` | string | Start month filter |
| `activeEndMonth` | string | End month filter |
| `activeRatingFilter` | string | `"green"`, `"yellow"`, `"red"`, `"unrated"` |
| `activeSort` | string | Sort order |
| `jewishOnly` | boolean | Jewish life filter toggle |
| `selectedFields` | Set | Selected study fields |
| `currentLang` | string | `"en"`, `"he"`, or `"ar"` |

**Persistent state** (survives page reloads) uses `localStorage`:

- `userRatings` — user's green/yellow/red ratings per university
- `topChoices` — user's top 3 preferences (`{1: "dest_X", 2: "dest_Y", 3: "dest_Z"}`)
- `cardNotes` — user's personal notes per card
- `preferredLang` — language preference

### 3.3 Rendering Pipeline

```
User action (filter change / search / sort)
  → renderCards()
    → getBaseFilteredDestinations()   // applies ALL filters
    → getFilteredDestinations()       // adds rating category filter
    → sort results
    → generate card HTML strings
    → inject into #cards-container
  → updateStats()                     // update stat counters & level pill counts
```

### 3.4 Key Functions

| Function | Purpose |
|---|---|
| `initApp()` | Bootstrap: set language, attach all event listeners, render |
| `renderCards()` | Main render loop: filter → sort → build HTML → inject |
| `getBaseFilteredDestinations(ignoreLevel)` | Core filtering engine |
| `destinationMatchesSemester(d, sem)` | Semester restriction logic |
| `destinationMatchesFields(d, selectedFields)` | Study field matching |
| `toggleFieldDetails(id, button, event)` | Expand/collapse all study-field badges on a card |
| `isFieldMatchExact(itemField, selectedField)` | Exact field comparison with PEP handling |
| `isPepField(field)` | Detects PEP program field name |
| `updateStats()` | Recalculates stat strip and level pill counts |
| `setLanguage(lang)` | Switch UI language (en/he/ar) + RTL |
| `setRating(destName, rating)` | Set green/yellow/red + save to localStorage |
| `setTopChoice(slot, destId)` | Set top 1/2/3 preference |
| `initMapIfNeeded()` | Lazy-init Leaflet map |
| `updateMapMarkers()` | Sync map pins with current filters |
| `openSyncModal()` / `generateSyncUrl()` | QR code cross-device sync |

---

## 4. Filters — Matching Mobility Online

The filters are designed to mirror the [Mobility Online portal](https://www.service4mobility.com/europe/PortalServlet?identifier=JERUSALEM01) as closely as possible.

### 4.1 Available Filters

| Filter | HTML ID | Mobility Online equivalent |
|---|---|---|
| Academic Year | `#filter-year` | Academic year (only `2027/2028` available) |
| Semester | `#filter-semester` | Semester (Autumn 2027 / Spring 2028 / Full year) |
| Study Fields | `#multiselect-dropdown` | Study field (87 fields, multi-select) |
| Program | `#filter-program` | Program type |
| Country | `#filter-country` | Country |
| Study Level | `#level-pills` | Study level (Bachelor/Master/Doctoral radio) |
| Start Month | `#filter-start-month` | _(not in Mobility Online — extra feature)_ |
| End Month | `#filter-end-month` | _(not in Mobility Online — extra feature)_ |
| Jewish Life | `#jewish-filter-btn` | _(not in Mobility Online — extra feature)_ |
| Rating Category | rating pills | _(not in Mobility Online — extra feature)_ |
| Free Text Search | `#search-input` | _(not in Mobility Online — extra feature)_ |

### 4.2 Program Values (matching Mobility Online exactly)

| Our value | Mobility Online value |
|---|---|
| `Bilateral Student Exchange Program - University level` | `Bilateral Student Exchange Program - University level` |
| `Business School` (matches via `.includes()`) | `Bilateral Student Exchange Program - Business School` |
| `Erasmus+` (matches via `.includes()`) | `Erasmus (SMS) - All fields` |

### 4.3 Semester Filtering Logic

The semester filter restricts by explicit semester-only constraints found in destination data:

- **Fall-only destinations** (e.g. Bocconi: `"1 BA/MBA students Fall 2027 only"`) → excluded from Spring and Full Year
- **Spring-only destinations** (e.g. Nova SBE: `"1 BA student for spring 2028 semester only"`) → excluded from Autumn and Full Year
- **Full-year-only destinations** (e.g. Kansai Gaidai: `"Full academic year exchange"`) → excluded from Autumn and Spring
- **All other destinations** → available for all semesters

The detection uses regex on `available_spots` and `notes` fields — see `destinationMatchesSemester()`.

When a semester is selected, the start/end month filters use that semester's calendar range from the destination notes. Spring ranges such as January–May, March–June, and April–September are not reused from the autumn dates. Cards also display the selected spring range instead of showing the autumn calendar.

### 4.4 PEP Field Filtering

"PEP" = **Integrative Program in Philosophy, Economics, and Politics**. In Hebrew: **פכ"מ** (פילוסופיה, כלכלה ומדע המדינה).

The field is handled by the same generic study-field matching and ranking code as Economics, Philosophy, Political Science, and every other field. There is no PEP-only filter, ranking path, or result treatment. Humanities-only agreements remain the only field-specific UX.

---

## 5. Internationalization (I18N)

Three languages are supported: **English**, **Hebrew** (RTL), **Arabic** (RTL).

Translations live in the `I18N` object (~line 3180):

```js
const I18N = {
  en: { ... },  // ~lines 3181–3360
  he: { ... },  // ~lines 3361–3520
  ar: { ... },  // ~lines 3521–3600
};
```

The `setLanguage(lang)` function (called `applyTranslations` internally) updates all UI text by element ID. Country names, city names, and study field names have their own localization lookup functions:

- `getLocalizedCountry(country, lang)`
- `getLocalizedCity(city, lang)`
- `getLocalizedField(field, lang)`

RTL is applied via `document.documentElement.dir = 'rtl'` for Hebrew and Arabic.

---

## 6. External Dependencies

| Library | CDN URL | Purpose |
|---|---|---|
| Font Awesome 6.5.1 | cdnjs | Icons throughout the UI |
| Leaflet 1.9.4 | unpkg | Interactive world map |
| QRCode.js 1.0.0 | cdnjs | QR code generation for sync |

No npm, no build tools, no server-side code.

---

## 7. Features Summary

1. **Card Grid View** — Sortable, filterable cards for each destination
2. **Expandable Study Fields** — Click “+N more” on a card to view every available field
3. **Interactive Map** — Leaflet world map with clickable pins
4. **Personal Ratings** — Green (excited) / Yellow (maybe) / Red (not interested)
5. **Top 3 Preferences** — Drag priority picks (slot 1, 2, 3)
6. **Personal Notes** — Free-text notes per destination (saved locally)
7. **Multi-language** — English / Hebrew / Arabic with full RTL support
8. **Study Field Multi-Select** — 87 checkboxes with search, quick "All"/"Clear"
9. **Cost of Living** — USD monthly estimates + local currency
10. **Jewish Community Info** — Synagogues, kosher dining per city
11. **QS World Rankings** — General + per-subject rankings
12. **Fact Sheet Links** — Direct PDF downloads from `Partner_Factsheets/`
13. **Cross-Device Sync** — QR code / share link to sync ratings between devices
14. **Application Instructions** — Collapsible HUJI-specific application guide
15. **Financial Notice Banner** — Prominent tuition/scholarship info banner
16. **Sort Options** — By rank, country, name, cost, rating, or QS rank
17. **Mobile Responsive** — Full responsive design with bottom-sheet dropdowns

---

## 8. Data Source & Provenance

The destination data was compiled from:

1. **Mobility Online Portal** (`service4mobility.com` with `identifier=JERUSALEM01`) — the official HUJI exchange portal containing partner institutions, agreements, study fields, and semester info
2. **Partner Fact Sheets** (PDF files in `Partner_Factsheets/`) — calendar dates, orientation, exam details, housing
3. **QS World University Rankings** — general and subject-level rankings
4. **Manual research** — cost of living, Jewish community details, scholarship info

The data is **static** and baked into `index.html`. To update destinations, you edit the `RAW_DESTINATIONS` array directly in the HTML file.

---

## 9. Git History & Key Commits

| Commit | Description |
|---|---|
| `aec2b98` | **Initial release**: 103 destinations, fact sheets, interactive map |
| `b05591d` | Dynamic study field tag highlighting based on active filter |
| `9156ad1` | Add 26 Faculty of Humanities exchange agreements (→ 125 total) |
| `a351f61` | Ordered field rankings with newlines + Field Rank N/A fallback |
| `4944988` | Clarify multi-agreement spot quotas by program |
| `bd47fe9` | Remove Humanities Only filter toggle |
| `66ea8ab` | Add two-way sync via QR code and smart share link |
| `75914a1` | Security: add `.gitignore` to prevent secret leaks |
| `101f33b` | Remove Humanities Faculty deadline from cards |
| `e3f7e33` | Fix Hebrew PEP translation: פפ"מ → פכ"מ |
| `4de1792` | Fix PEP filtering: strict matching, eliminate Brazil false positive |
| `e5b64d9` | Remove "PEP Only" button, standardize generic field filtering |
| _(pending)_ | Restrict year filter to 2027/2028, add semester filter |

---

## 10. How to Make Common Changes

### Add a new destination
1. Add a new object to the `RAW_DESTINATIONS` array in `index.html`
2. Follow the exact schema (all 30 fields)
3. Update the country dropdown if it's a new country
4. Add a fact sheet PDF to `Partner_Factsheets/` if available

### Add a new filter
1. Add HTML `<select>` or pill in the filter bar (~line 2580–2700)
2. Add a state variable (`let activeXxx = ''`)
3. Add filter logic inside `getBaseFilteredDestinations()`
4. Add event listener in `initApp()`
5. Add i18n keys to `I18N.en`, `I18N.he`, `I18N.ar`
6. Add translation application in `setLanguage()`

### Change study fields list
Study fields are dynamically extracted from the data:
```js
const ALL_FIELDS = [...new Set(RAW_DESTINATIONS.flatMap(d => d.study_fields))].sort();
```
Just update the `study_fields` arrays in `RAW_DESTINATIONS`.

### Update translations
Edit the `I18N` object. Each language block has the same keys. The `setLanguage()` function maps keys to DOM element IDs.

---

## 11. Development Notes

- **No API keys** are used anywhere. The app is fully static.
- **PowerShell gotcha**: Inline Python with `-c "..."` often breaks in PowerShell due to quote escaping. Always write to a `.py` file and run it.
- **Git path**: `git` may not be on the PowerShell PATH. Use `"C:\Program Files\Git\bin\git.exe"` in subprocess calls.
- **JS validation**: After editing, extract JS with `scratch/extract_js.py` and validate with `node -c scratch/extracted.js`.
- **The `scratch/` directory** contains temporary development scripts. It is NOT part of the deployed app.

---

## 12. Pending / Uncommitted Work

As of 2026-11-17, the following changes are **staged but not yet pushed**:

- Academic year filter restricted to `2027/2028` only (removed 2024/2025, 2025/2026, 2026/2027)
- New **Semester filter** added with options: All Semesters, Autumn 2027, Spring 2028, Full academic year 2027-2028
- Semester filtering logic via `destinationMatchesSemester()` function
- I18N translations for semester filter in all 3 languages
- `activeYear` defaults to `"2027/2028"` (auto-selected on page load)

**To commit and push:**
```python
# Run scratch/git_commit_push.py
```

---

## 13. Quick Reference: Line Ranges

| Section | Approximate Lines |
|---|---|
| CSS styles | 1–1950 |
| HTML structure | 1950–2750 |
| `RAW_DESTINATIONS` data | 2754 (single long line) |
| `I18N` translations | 3180–3600 |
| `setLanguage()` / `applyTranslations` | 4240–4350 |
| State variables | 4630–4650 |
| Study field multiselect logic | 4650–4780 |
| Card HTML template | 4550–4600 |
| `destinationMatchesSemester()` | 5047–5080 |
| `getBaseFilteredDestinations()` | 5081–5136 |
| `renderCards()` | 4480–4560 |
| `updateStats()` | 5150–5250 |
| Map logic | 5260–5420 |
| Sync/QR logic | 5420–5590 |
| `initApp()` event listeners | 5640–5760 |
