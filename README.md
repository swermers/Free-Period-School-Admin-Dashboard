# Free-Period-School-Admin-Dashboard
A zero-backend, privacy-first data visualization tool for school administrators and counselors.

# School Admin Dashboard — by Free Period

**A zero-backend, privacy-first data visualization tool for school administrators and counselors.**

Upload a CSV from PowerSchool (or any SIS), get instant interactive visualizations, correlations, and insights. Export as PDF or copy to clipboard for reports. No login. No server. No data leaves the browser.

---

## The Problem

School administrators and counselors sit on top of rich student data in PowerSchool — attendance, grades, demographics, behavior — but the native interface shows it as static grids of numbers. The visual layer is missing. Building Tableau-style dashboards requires IT approval, vendor contracts, and data-governance reviews that can take months. Meanwhile, you just want to understand whether attendance is correlated with GPA in your ninth-grade cohort before next week's faculty meeting.

## The Solution

A single HTML file you open locally in your browser. Drop in a CSV. See your data. Generate a report. Close the tab, everything's gone.

- **No server, no database, no auth.** Everything runs client-side in the browser.
- **Privacy by default.** Data never leaves the device. Clear cookies and it's genuinely gone.
- **Two modes.** A generalized CSV explorer and a PowerSchool-aware mode with auto-detected fields and pre-built correlation views.
- **Export-ready.** Copy the dashboard to clipboard or generate a PDF report to share with superiors.
- **Free.** Hosted as a static page on the Free Period site — or downloaded and run offline.

---

## Target Users

- **School administrators** preparing board reports, enrollment reviews, or strategic planning documents.
- **School counselors** looking at attendance-grade correlations, demographic equity gaps, or behavior-academic relationships for specific caseloads.
- **Deans and academic leaders** who need visual summaries without submitting tickets to IT.

The common thread: people with legitimate access to student data who need better tools to *understand* it, not more tools to *collect* it.

---

## Feature Specification

### Core (both modes)

1. **CSV upload** — drag-and-drop or file picker. Parses in-browser via PapaParse. No upload, no transmission.
2. **Data preview table** — first 50 rows, sortable, with column-type detection (numeric / categorical / date).
3. **Chart builder** — pick columns, pick chart type (bar, line, pie, scatter, histogram, heatmap). Live preview.
4. **Dashboard canvas** — add multiple charts to a single canvas, drag to reorder, resize.
5. **Filters** — click any chart segment to filter the whole dashboard (cross-filtering).
6. **Export**
   - *Copy to clipboard* — copies rendered dashboard as an image suitable for pasting into Google Docs, Slides, Word.
   - *Export as PDF* — full report with title, generated date, all charts, and a summary-stats appendix.
7. **Session-only memory** — everything in-memory. Refresh the page = clean slate. Optional "Save config" exports a JSON of the dashboard layout (not the data) for reuse.

### Generalized mode

- Agnostic to column names.
- User picks which columns to visualize.
- All chart types available.
- Summary statistics panel (mean, median, distribution, nulls) for any selected column.

### PowerSchool mode

- **Auto-detects common field patterns** on import. Pattern-matching rules:
  - `student_number`, `studentid`, `id` → identifier
  - `grade_level`, `gradelevel`, `grade` (integer 0–12) → grade level
  - `attendance*`, `absences`, `tardies`, `present_pct` → attendance cluster
  - `gpa`, `grade` (alphanumeric), `final_grade`, `course_grade` → academic cluster
  - `gender`, `ethnicity`, `race`, `ell`, `sped`, `iep`, `504`, `frl`, `free_reduced` → demographic cluster
  - `referral*`, `suspension*`, `detention*`, `incident*` → behavior cluster
- **Pre-built dashboard sections:**
  - *Cohort overview* — student counts by grade, demographic breakdowns.
  - *Attendance* — distribution, chronic-absence threshold (10%+) flagging, trends if date columns present.
  - *Academic performance* — GPA distribution, grade-level comparisons, failure-rate indicators.
  - *Behavior* — referral frequency, incident-type breakdowns.
  - *Correlations* — heatmap of Pearson correlations across numeric fields; highlighted pairs with |r| > 0.3.
  - *Equity lens* — performance and attendance segmented by demographic categories with visual gap indicators.
- **Drill-down** — click any chart to filter the entire dashboard to that segment. Click a demographic bar to see only that group's attendance and GPA distributions.
- **Insight callouts** — plain-language summary above each section. Example: "Ninth graders with attendance below 90% have a mean GPA 0.8 points lower than peers above 90%."

### Report generation

- **Title page** — user-editable title, optional subtitle, date.
- **Executive summary** — auto-generated 3-5 bullet points from the strongest correlations and most notable distributions.
- **Charts** — all active dashboard charts, one or two per page, with captions.
- **Appendix** — summary statistics table for every numeric column.
- **Footer** — "Generated locally. No data transmitted." with timestamp.

---

## Technical Architecture

### Stack

- **Single HTML file.** Everything inlined — HTML, CSS, JS — for true portability.
- **No build step.** Written to be editable by anyone with a text editor.
- **Libraries (via CDN or inlined):**
  - [PapaParse](https://www.papaparse.com/) — CSV parsing.
  - [Chart.js](https://www.chartjs.org/) — core charts (bar, line, pie, scatter).
  - [D3.js](https://d3js.org/) — heatmaps and custom visualizations.
  - [html2canvas](https://html2canvas.hertzen.com/) — clipboard image export.
  - [jsPDF](https://github.com/parallax/jsPDF) — PDF generation.
  - [simple-statistics](https://simplestatistics.org/) — correlations, summary stats.

### File structure

```
free-period-dashboard/
├── index.html              # single-file app (generalized + PowerSchool modes)
├── README.md               # this file
├── sample-data/
│   ├── generic-example.csv
│   └── powerschool-mock.csv
└── screenshots/
    └── dashboard-preview.png
```

### Data flow

```
CSV file
   ↓ (PapaParse, in-browser)
Parsed rows + inferred schema
   ↓ (mode detection: generalized | PowerSchool pattern-match)
Internal data model { rows, columns, types, clusters }
   ↓ (user interactions: select columns, add charts, filter)
Dashboard state (JS object in memory)
   ↓ (Chart.js / D3 render)
Rendered dashboard
   ↓ (html2canvas | jsPDF on user action)
Clipboard image | PDF download
```

No network requests after initial page load. Verifiable with browser dev tools.

### Privacy and security properties

- **No `fetch` or `XMLHttpRequest` calls after load** beyond CDN library fetches (which can be removed by inlining for a fully offline build).
- **No `localStorage` or `IndexedDB`** unless user explicitly exports a layout config.
- **No analytics, telemetry, or error reporting.**
- **Content Security Policy header** in the HTML to block any accidental external calls.
- The page can be saved locally (`File > Save As`) and run fully offline from a USB drive or network share.

---

## Claude Code Build Plan

Hand this to Claude Code as the build brief. Phases are ordered so you get a working tool after Phase 1 and layer capability on top.

### Phase 1 — Skeleton and CSV ingestion (generalized mode only)

- Single `index.html` with layout: header, upload zone, data preview table, empty dashboard canvas.
- PapaParse integration with drag-and-drop and file picker.
- Column-type inference (numeric, categorical, date, boolean).
- Data preview table showing first 50 rows with column-type badges.
- Minimal styling — clean, neutral, works in any browser.

### Phase 2 — Chart builder

- "Add chart" flow: pick chart type → pick columns → preview → add to dashboard.
- Chart types: bar, line, pie, scatter, histogram.
- Charts render via Chart.js into the dashboard canvas.
- Each chart has a header (title, edit, delete) and a resize handle.
- Basic drag-to-reorder on the canvas.

### Phase 3 — Filtering and summary stats

- Click a chart segment → adds a filter chip → all other charts re-render filtered.
- Summary stats panel — pick a column, see mean / median / stdev / min / max / null count / distribution sparkline.
- Clear-all-filters button.

### Phase 4 — PowerSchool mode

- Auto-detect toggle that runs pattern-matching on column names.
- If matches found, show "PowerSchool mode available" banner with one-click activation.
- Pre-built dashboard templates for the five sections (Cohort / Attendance / Academic / Behavior / Correlations / Equity).
- Correlation heatmap via D3.
- Insight callout text generator — template-driven strings pulled from computed stats.

### Phase 5 — Export

- Copy to clipboard — html2canvas renders the dashboard area, copies PNG to clipboard via the Clipboard API.
- Export PDF — jsPDF composes title page + executive summary + charts + appendix.
- Executive summary generator — finds top 3 correlations and top 2 distribution anomalies, templates them into sentences.

### Phase 6 — Polish and packaging

- Responsive layout (works on laptop, not designed for mobile).
- Inline all CDN libraries into `index.html` for a fully offline single-file build.
- Sample CSVs in `sample-data/`.
- Screenshot for README.
- Test with a real or realistic PowerSchool export.

---

## Known Limitations and Open Questions

- **Large files.** Browser performance will degrade past ~50K rows. Acceptable for most school sites; worth noting.
- **PDF chart fidelity.** jsPDF rasterizes charts; vector export would require a different approach.
- **PowerSchool exports vary.** The field-detection rules are based on common PowerSchool column conventions but will miss custom fields. An explicit "map columns" UI fallback is needed in Phase 4.
- **No multi-file joins.** If you want attendance and grades from two separate exports, you'd need to join them in Excel first. Future work.
- **Statistical depth.** Pearson correlations only in v1. No regression, no significance testing. Intentionally kept simple to avoid implying causal claims.

---

## Roadmap After v1

- **Column-mapping UI** — manual override for auto-detected fields.
- **Multi-CSV joins** — upload two CSVs, pick a join key, merged dataset available for analysis.
- **Saved dashboard templates** — export/import layout JSON so a district counselor team can share a standard dashboard.
- **Cohort comparisons** — two-group comparison view (this year vs. last year, grade 9 vs. grade 10).
- **Accessibility pass** — keyboard navigation, screen-reader labels on charts.
- **Offline installer** — package as a downloadable ZIP with `index.html` and all libraries inlined for air-gapped environments.

---

## Distribution

- Hosted at a subpath of `freeperiod.xyz` — e.g. `freeperiod.xyz/dashboard`.
- Also published as a GitHub release with the standalone `index.html` for offline use.
- Linked from the Free Period YouTube channel as a companion resource for any video on school data.

---

## Positioning Notes (for Free Period)

The product story writes itself: *"Your school data, visualized in your browser, without sending it anywhere."* That's the hook. Every administrator has been told "no" by IT when asking for a better dashboard. This is the tool that doesn't need permission because it doesn't touch the network.

Suggested launch content:
- YouTube: *"I built a PowerSchool dashboard that runs entirely in your browser — no login, no server"*
- Short demo: upload a CSV → two clicks → exported PDF report.
- Explicit callout of the privacy model as the differentiator vs. Tableau, PowerBI, or any hosted tool.
