# School Data Dashboard — Comprehensive Architecture Plan

**Project:** Free Period — Privacy-First School Data Visualization Tool
**Author:** Shayne (LAS)
**Build Target:** One-shot Claude Code implementation
**Deliverable:** Single-file HTML application with embedded JS, zero backend, zero login

---

## 1. Project Overview

### 1.1 Vision

A privacy-first, client-side data visualization platform for school administrators and counselors. Users upload CSV files (from PowerSchool or generic sources), the tool auto-detects field types, and generates multi-perspective dashboards with statistical correlations, drill-down analytics, and exportable reports — all without login, backend infrastructure, or persistent data storage.

### 1.2 Core Principles

- **Zero backend, zero login, zero data persistence.** Everything runs in the browser; data is gone when the tab closes.
- **Massive CSV support.** Handle 100k+ rows efficiently via Web Workers and streaming.
- **Two modes:** Generalized (any CSV) and PowerSchool-optimized (auto-mapped fields, preset analytics).
- **Visual-first analytics with statistical rigor underneath.** Replace PowerSchool's grid-of-numbers with readable insights.
- **Single-file or ZIP upload.** Support combining multiple exports (attendance + grades + demographics).
- **Export pipeline.** Clipboard copy, PDF report, PNG snapshots of individual charts.

### 1.3 Target Users

School administrators, counselors, data analysts, educational leaders who need quick insights from student data without IT involvement or privacy risk.

### 1.4 Success Criteria

1. User uploads a PowerSchool export and sees a meaningful dashboard in under 30 seconds
2. User can filter, drill down, and cross-reference demographic/academic/behavioral data interactively
3. User can export a polished PDF report suitable for presenting to superiors
4. No data ever leaves the user's machine
5. Tool works offline after first load (for fully air-gapped use)

---

## 2. Data Architecture

### 2.1 Input Handling

**Supported Formats:**
- Single CSV file (recommended up to 500 MB, technical limit ~2 GB on modern browsers)
- Multiple CSVs uploaded as a ZIP archive (for multi-table school data — attendance, grades, demographics separately)
- Automatic encoding detection (UTF-8 priority, fallback to ISO-8859-1, Windows-1252)
- Delimiter auto-detection (comma, semicolon, tab, pipe)

**File Processing Strategy:**
- **ZIP extraction:** JSZip library for in-browser extraction (no server roundtrip)
- **CSV parsing:** PapaParse with streaming mode for large files
- **Web Worker offloading:** Heavy parsing happens off the main thread to prevent UI freezing
- **Chunked processing:** For files over 50 MB, parse in ~10 MB chunks with progress updates
- **Session storage:** IndexedDB for session-only persistence (cleared on tab close or manual reset)

### 2.2 Internal Data Representation

```javascript
RawData = {
  metadata: {
    fileName: string,
    uploadedAt: timestamp,
    rowCount: number,
    columnCount: number,
    dataSize: bytes,
    detectedType: "generic" | "powerschool",
    fieldMappings: { [originalName]: recognizedType },
    sourceFiles: [string]  // for ZIP uploads with multiple CSVs
  },
  rows: [
    { field1: value, field2: value, ... },
    ...
  ],
  schema: [
    {
      name: string,           // original column name
      displayName: string,    // user-friendly, editable
      type: "string" | "number" | "date" | "categorical" | "boolean",
      isNumeric: boolean,
      uniqueValues: Set,      // sample for categoricals
      range: { min, max } | null,
      missingCount: number,
      recognizedCategory: "demographic" | "attendance" | "grade" | "behavior" | "identity" | null,
      semanticRole: string    // e.g., "gpa", "attendance_pct", "grade_level"
    }
  ],
  joins: [                    // when multiple CSVs are uploaded via ZIP
    { leftTable, leftKey, rightTable, rightKey, joinType }
  ]
}
```

### 2.3 Multi-File Handling (ZIP Uploads)

When a user uploads a ZIP with multiple CSVs (typical PowerSchool export pattern):
1. Extract all CSVs in-browser
2. Show file picker: "We found `attendance.csv`, `grades.csv`, `students.csv`. Treat these as related?"
3. Auto-detect join keys (columns named `student_id`, `studentID`, `id` across files)
4. Offer join configuration UI: inner join, left join, or keep separate
5. Merge into a single unified dataset OR keep as linked datasets with cross-table querying

---

## 3. Field Recognition System

### 3.1 PowerSchool Pattern Matching (Specific Version)

The dashboard auto-detects PowerSchool fields using fuzzy matching on column names. Patterns are case-insensitive, match common variations.

**Attendance Fields:**
- Patterns: `*attend*`, `*absence*`, `*present*`, `*tardy*`, `*unexcused*`, `*excused*`
- Mapped to: `attendance_pct`, `absences_count`, `tardies_count`, `attendance_rate`

**Academic Fields:**
- Patterns: `*gpa*`, `*grade*`, `*score*`, `*mark*`, `*percent*`, `*final*`, `*semester*`
- Mapped to: `gpa`, `current_grade`, `cumulative_grade`, `subject_grade`, `semester_mark`

**Demographic Fields:**
- Patterns: `*grade*level*`, `*year*`, `*cohort*`, `*gender*`, `*sex*`, `*ethnicity*`, `*race*`, `*sped*`, `*iep*`, `*504*`, `*ell*`, `*esl*`, `*lunch*status*`, `*frl*`, `*ses*`
- Mapped to: `grade_level`, `gender`, `ethnicity`, `special_ed`, `language_learner`, `socioeconomic`

**Behavioral Fields:**
- Patterns: `*behavior*`, `*conduct*`, `*discipline*`, `*incident*`, `*referral*`, `*suspension*`, `*expulsion*`, `*detention*`
- Mapped to: `behavior_grade`, `discipline_incidents`, `suspensions`, `conduct_rating`

**Identity Fields:**
- Patterns: `*student*id*`, `*student*number*`, `*dcid*`, `*name*first*`, `*name*last*`, `*email*`
- Mapped to: `student_id`, `first_name`, `last_name`, `email` (anonymized by default for reporting)

**Enrollment/Dates:**
- Patterns: `*enroll*`, `*entry*date*`, `*exit*date*`, `*dob*`, `*birth*`
- Mapped to: `enrollment_date`, `exit_date`, `date_of_birth`, `age`

### 3.2 Generic Mode

For non-PowerSchool CSVs:
- User manually tags columns via UI: numeric, categorical, date, identifier, text
- System infers smart defaults:
  - Anything with 2–20 unique values → categorical
  - 90%+ unique values → identifier
  - Pure numeric → numeric
  - Parseable as ISO date → date
- User can override any auto-detection

### 3.3 Field Override UI

Before dashboard generation, user sees a confirmation screen:
- Each column with its detected type and category
- Dropdown to override type
- Checkbox to hide from dashboard
- Option to rename display label
- "Accept all" or "Review each" modes

---

## 4. Analytics Engine

### 4.1 Automatic Metrics (All Data)

Computed on load for every field, cached for reuse:

- **Descriptive Statistics:** mean, median, mode, std dev, variance, min, max, range, quartiles (Q1, Q3, IQR)
- **Distribution Analysis:** histogram buckets (Sturges' rule for bin count), percentiles (deciles, quartiles)
- **Categorical Summaries:** value counts, top N categories, missing value analysis, cardinality
- **Trend Detection:** linear regression slope and R² for time-series fields (when a date column exists)
- **Outlier Detection:** values outside 1.5 × IQR flagged for review

### 4.2 Correlation Analysis (PowerSchool Version)

**Pearson Correlation Matrix:**
- Computed for all numeric field pairs
- Output: correlation heatmap with diverging color scale (-1 red → 0 white → +1 blue)
- Annotations on strong correlations (|r| > 0.5)
- Significance indicator (p-value from t-test on correlation coefficient)

**Spearman Rank Correlation:**
- Available as toggle for non-linear monotonic relationships
- Better for ordinal data (e.g., grade levels 9-12)

**Segmentation Correlations:**
- Subset data by demographic group (grade level, gender, ethnicity, SES)
- Recalculate correlation matrix per segment
- Compare side-by-side: "How does attendance→GPA correlation differ for 9th vs 12th graders?"

**Cross-Demographic Analysis:**
- Attendance by grade level → grouped bar chart
- GPA distribution by demographic → violin plots or faceted histograms
- Discipline incidents by gender → stacked bar chart
- Performance tiers (top/mid/bottom quartiles) by demographic → contingency table + mosaic plot

### 4.3 Filtering & Drill-Down

- **Interactive Filters:** checkbox, slider, and dropdown filters on any categorical or numeric field
- **Linked Brushing:** selecting a bar in one chart filters all other charts
- **Drill-Down Hierarchy:** click grade level → see students in that grade → click student → see individual record detail
- **Comparative View:** side-by-side comparison of two demographic subsets (e.g., boys vs girls attendance patterns)
- **Saved Views:** user can bookmark filter combinations as "views" within the session

### 4.4 Statistical Rigor — What We Don't Do

Explicitly not in scope (to keep tool honest and accessible):
- Causal inference (correlation ≠ causation messaging prominent)
- Multi-variate regression beyond simple linear
- Machine learning predictions
- Inferential hypothesis testing beyond simple t-test
- Time-series forecasting

Tool is explicitly framed as **descriptive and exploratory**, not predictive.

---

## 5. Visualization Module

### 5.1 Chart Library Strategy

- **Primary:** Chart.js (lightweight, 60 KB gzipped, no dependencies, fast for <10k data points)
- **Secondary:** Plotly.js (for correlation heatmaps, violin plots, and advanced layouts)
- **Fallback:** Native HTML canvas for extreme performance edge cases (100k+ point scatter)

All loaded via CDN with local caching for offline support after first load.

### 5.2 Chart Types

**V1 Launch Set:**
1. **Bar Charts** — distribution, group comparisons, categorical breakdowns
2. **Line Charts** — trends over time, progression metrics
3. **Pie/Donut Charts** — composition (e.g., % of students by grade level)
4. **Histograms** — distribution of numeric values (GPA, attendance %)
5. **Correlation Heatmap** — Pearson r matrix with color scale and annotations
6. **Scatter Plots** — two-variable relationships (attendance vs GPA) with optional trend line
7. **Summary Stat Cards** — headline numbers (total students, avg GPA, avg attendance)

**V1.1+ Additions:**
8. **Box Plots** — quartile distribution, outlier detection
9. **Violin Plots** — density distributions by group
10. **Stacked Bar Charts** — multi-category composition
11. **Grouped Bar Charts** — side-by-side comparisons across demographics
12. **Mosaic Plots** — two-way categorical breakdown visualization
13. **Choropleth** — if geographic data is present (e.g., by zip code or district)

### 5.3 Chart Configuration Schema

Each chart is configured by:
- **Data Source:** which rows (filter state), which fields (x, y, grouping)
- **Axes:** field mapping, scaling (linear/log), format (% vs raw)
- **Grouping/Coloring:** by demographic or custom segment
- **Labels:** title, axis labels, legend, data labels, tooltips
- **Styling:** color palette (accessible, colorblind-safe — ColorBrewer schemes by default)
- **Export:** PNG snapshot, CSV of underlying data, clipboard copy

### 5.4 Dashboard Layout

- **Responsive 12-column grid** using CSS Grid
- **Draggable/resizable chart tiles** (user can rearrange)
- **Preset layouts** for PowerSchool mode: "Overview," "Academic Deep-Dive," "Behavioral Analysis," "Demographic Comparison"
- **Print-friendly stylesheet** (triggered automatically for PDF export)
- **Dark/light mode toggle**

---

## 6. Code Organization

### 6.1 File Structure

```
school-data-dashboard.html   (single file, no build step required)

├── <head>
│   ├── <script src="papaparse.min.js">
│   ├── <script src="chart.min.js">
│   ├── <script src="plotly.min.js">
│   ├── <script src="jszip.min.js">
│   ├── <script src="html2canvas.min.js">   // for PNG export
│   ├── <script src="jspdf.min.js">          // for PDF export
│   └── <style> (Tailwind via CDN + custom dashboard CSS)
│
└── <body>
    └── <script>
        ├── DATA_LAYER
        │   ├── FileUploadHandler       — CSV/ZIP parsing, encoding detection
        │   ├── DataValidator           — row/column validation, type inference
        │   ├── SchemaBuilder           — field detection, PowerSchool pattern matching
        │   ├── FieldMapper             — generic vs PowerSchool mode logic
        │   └── DataStore               — IndexedDB session management
        │
        ├── ANALYTICS_LAYER
        │   ├── DescriptiveStats        — mean, median, std dev, percentiles
        │   ├── CorrelationAnalyzer     — Pearson/Spearman matrix generation
        │   ├── SegmentationEngine      — group-by operations, filtered stats
        │   ├── TrendDetector           — linear regression, time-series analysis
        │   └── OutlierDetector         — IQR-based outlier flagging
        │
        ├── VISUALIZATION_LAYER
        │   ├── ChartFactory            — creates Chart.js instances from config
        │   ├── HeatmapRenderer         — Plotly for correlation matrices
        │   ├── TooltipEngine           — interactive hover details
        │   ├── LegendManager           — color mapping, categorical encoding
        │   └── ResponsiveLayout        — grid-based chart positioning
        │
        ├── FILTER_LAYER
        │   ├── FilterUI                — checkbox, slider, dropdown generators
        │   ├── FilterLogic             — AND/OR logic, numeric ranges, categorical sets
        │   ├── LinkedBrushingEngine    — cross-chart filtering
        │   └── DrillDownManager        — hierarchy navigation state
        │
        ├── EXPORT_LAYER
        │   ├── ClipboardExporter       — copy dashboard HTML to clipboard
        │   ├── PDFExporter             — generate PDF with charts + stats
        │   ├── PNGExporter             — snapshot individual charts
        │   ├── ReportGenerator         — multi-page narrative report
        │   └── DataTableExporter       — filtered data to CSV
        │
        ├── UI_LAYER
        │   ├── ModalManager            — upload, config, drill-down modals
        │   ├── SidebarControls         — filter controls, chart selection
        │   ├── DashboardGrid           — responsive 12-column layout
        │   ├── ProgressIndicator       — file upload, parsing, chart render
        │   └── ThemeManager            — dark/light mode, accessibility
        │
        ├── MODE_SELECTOR
        │   ├── PowerSchoolMode         — auto-field mapping, preset templates
        │   ├── GenericMode             — manual field tagging, flexible charts
        │   └── ModeConfig              — shared defaults, switching logic
        │
        └── WORKERS
            ├── csv-parser-worker.js    — offload parsing to background thread
            ├── correlation-worker.js   — compute correlation matrix async
            └── aggregation-worker.js   — group-by on large datasets
```

### 6.2 Module Communication Pattern

- **Pub/sub event bus** for cross-module communication (e.g., `filter:changed`, `data:loaded`, `chart:export`)
- **Central state store** (simple object with subscription pattern) — holds current filter state, active dataset, dashboard config
- **Pure functions** for analytics — no side effects, easy to test
- **Render layer** subscribes to state changes and re-renders affected charts only

### 6.3 Build Approach

- **No build step required** for V1 — single HTML file, all dependencies via CDN
- **Optional V2:** esbuild bundle for truly offline use (single self-contained HTML, ~2 MB)

---

## 7. Performance Optimization Strategy

### 7.1 Large File Handling (100k+ Rows)

**Parsing:**
- PapaParse with `worker: true` (runs in Web Worker)
- Stream mode: process in chunks, emit progress events
- Progress callback: update UI every 10k rows parsed
- Cancel button: user can abort long parses

**Storage:**
- Raw data stored in IndexedDB (async, doesn't block main thread)
- In-memory: only the current filter subset (typically 1k–5k rows for charts)
- Virtual scrolling for data tables (render only visible rows)

**Computation:**
- Correlation matrix computed in Web Worker
- Aggregations (group-by, mean-by-group) done via Web Worker for >10k rows
- Results cached keyed on filter state hash
- Debounced recalculation on filter changes (300ms)

### 7.2 Rendering Performance

- **Chart sampling:** for scatter plots with >10k points, sample to 5k for rendering (full data still used for statistics)
- **Canvas over SVG** for high-point-count charts
- **RequestAnimationFrame** batching for chart updates
- **Lazy chart initialization:** charts below the fold only render when scrolled into view

### 7.3 Memory Management

- Clear unused chart instances on filter change (prevent memory leaks from Chart.js instances)
- Offer "reset all" button to clear IndexedDB and free memory
- Warn user if dataset exceeds 200 MB ("Large dataset — consider sampling")

---

## 8. Export & Reporting

### 8.1 Clipboard Export

- Copy entire dashboard as HTML to clipboard
- User pastes into email, Google Docs, Word, or any rich-text editor
- Charts converted to embedded PNGs via html2canvas
- Summary statistics included as formatted text

### 8.2 PDF Report

- Multi-page layout with cover page (dataset summary, generation date, filter state)
- One chart per section, with accompanying statistical summary
- Correlation analysis section (PowerSchool mode)
- Top findings auto-generated: "Strongest correlation: attendance × GPA (r = 0.67)"
- Footer: "Generated locally — no data transmitted"
- Branded with Free Period logo (optional toggle)

### 8.3 PNG Snapshots

- Per-chart PNG download
- High-resolution (2x DPI) for presentation use
- Transparent background option

### 8.4 CSV Export

- Filtered dataset as CSV
- Aggregated results (e.g., "avg GPA by grade level") as CSV
- Correlation matrix as CSV

### 8.5 Report Templates (PowerSchool Mode)

Preset report structures:
1. **Executive Summary** — 1 page, headline stats + 2 charts
2. **Academic Performance Report** — GPA distribution, trends, correlations with attendance
3. **Attendance Analysis** — patterns by demographic, grade level, trends over time
4. **Behavioral Insights** — discipline patterns, correlations with academic/demographic factors
5. **Full Dashboard Snapshot** — everything currently displayed

---

## 9. Privacy & Security Architecture

### 9.1 Data Handling Guarantees

- **No network requests with user data.** All processing is local. (CDN loads for libraries are cached on first load.)
- **IndexedDB session-only.** Data cleared when tab closes OR when user clicks "Reset."
- **No telemetry, no analytics, no tracking.** Explicitly stated in UI and documentation.
- **Offline capable** after first load (service worker caches all libraries).

### 9.2 Anonymization Options

- Toggle: "Anonymize student names" — replaces names with `Student_001`, `Student_002`, etc.
- Toggle: "Strip identifiers before export" — removes student IDs from any PDF/CSV export
- Drill-down on individual students shows only if anonymization is off

### 9.3 Compliance Posture

- GDPR-friendly: no data processing happens on any server
- FERPA-friendly: student data never leaves the administrator's machine
- Explicit privacy notice on upload screen: "Your data stays on your computer. Nothing is transmitted."

---

## 10. User Experience Flow

### 10.1 First-Time User Journey

1. **Landing** — one-page intro: "Upload a CSV. Get instant insights. Nothing leaves your computer."
2. **Mode selection** — "PowerSchool data" or "Any CSV" (with brief description of each)
3. **Upload** — drag-and-drop zone or file picker; support for CSV or ZIP
4. **Parse progress** — progress bar with row count
5. **Field confirmation** — "We detected these fields — look right?" (editable)
6. **Dashboard auto-generated** — default layout based on detected fields
7. **Explore** — filter, drill down, rearrange
8. **Export** — clipboard, PDF, or PNG

### 10.2 Returning User (Same Session)

- Data persists in IndexedDB across tab reloads (until "Reset" clicked)
- Filter state and custom layouts restored on reload
- "Clear all" button prominent in header

### 10.3 Error States

- **Malformed CSV:** show row number and error, offer to skip bad rows
- **Empty file:** "This file appears empty. Try another."
- **Unsupported encoding:** offer manual encoding selection
- **No numeric fields detected:** disable correlation features, explain why
- **Huge file warning:** "This file is 300 MB. Processing may take 1-2 minutes. Continue?"

---

## 11. Implementation Roadmap

### 11.1 One-Shot Build Scope (Initial Delivery)

**In the one-shot Claude Code build:**
- Single HTML file with embedded JS and CSS
- Generic mode fully functional (upload, parse, visualize, export)
- PowerSchool mode with full field auto-detection
- All V1 chart types (bar, line, pie, histogram, heatmap, scatter, summary cards)
- Filtering + linked brushing
- Clipboard, PDF, and PNG export
- IndexedDB session persistence
- Web Worker for CSV parsing
- Responsive dashboard grid

**Explicitly deferred to V1.1:**
- Multi-CSV ZIP join logic (simple append only in V1)
- Box/violin/mosaic plots
- Drag-resize of chart tiles (fixed grid in V1)
- Custom report template editor
- Service worker for full offline mode

### 11.2 Post-Launch Iteration

- **V1.1** — advanced chart types, ZIP multi-file joins
- **V1.2** — custom report templates, saved dashboard presets
- **V1.3** — service worker offline mode, PWA install
- **V2** — optional local-first sync via Yjs for team collaboration (without server)

---

## 12. Technical Dependencies

All loaded via CDN with integrity hashes for V1. Totals roughly 1.5 MB gzipped.

| Library | Purpose | Size (gzipped) |
|---|---|---|
| PapaParse | CSV parsing | 12 KB |
| Chart.js | Primary charts | 60 KB |
| Plotly.js | Heatmaps, violins | 800 KB |
| JSZip | ZIP extraction | 30 KB |
| html2canvas | HTML → PNG | 45 KB |
| jsPDF | PDF generation | 350 KB |
| Tailwind CSS | Styling | 40 KB (via CDN, purged) |

---

## 13. Free Period Integration Notes

- Hosted at `freeperiod.xyz/tools/dashboard` (or subdomain like `dashboard.freeperiod.xyz`)
- Linked from main Free Period landing as a flagship free tool
- No account needed, no email capture on the tool itself
- Optional footer: "Made by Free Period — ed-tech for teachers building with AI" with link back
- Shareable via direct URL (tool itself has no sharing, but link to it is shareable)
- Open-sourceable: code is self-contained, MIT licensable for community contributions
- Analytics on the tool: only privacy-respecting page-view analytics on the landing, none on the tool itself

---

## 14. Testing Strategy

### 14.1 Test Data Sets

Before building, prepare synthetic test CSVs:
1. **Small generic** — 100 rows, 10 columns, mixed types
2. **Medium PowerSchool-like** — 5,000 rows with realistic field names
3. **Large PowerSchool-like** — 50,000 rows, full demographic/academic/behavioral data
4. **Edge cases** — quoted fields with newlines, missing values, mixed encodings, single-column files

### 14.2 Validation Checklist

- [ ] Uploads CSV of 100k rows without crashing
- [ ] Uploads ZIP of 3 CSVs successfully
- [ ] Auto-detects all PowerSchool field categories
- [ ] Generates valid correlation matrix
- [ ] PDF export includes all visible charts
- [ ] Clipboard paste into Google Docs preserves formatting
- [ ] Data cleared on "Reset" and tab close
- [ ] Works offline after first load (with service worker)
- [ ] Keyboard-navigable (WCAG AA)
- [ ] Colorblind-safe palette by default

---

## 15. Open Questions for Implementation

1. **Which Plotly bundle?** Full bundle is 3 MB — can use `plotly-basic` (1 MB) if we drop 3D/mapbox
2. **PDF fidelity vs size:** Higher-DPI charts = larger PDFs. Default to 2x, offer 1x for email-sized exports
3. **Student identity defaults:** Should anonymization be ON by default for PowerSchool mode? (Recommendation: yes)
4. **ZIP join logic in V1:** Append-only (stack files as separate tables) vs inner join on auto-detected keys. (Recommendation: append-only with join as V1.1 feature)
5. **Offline-first via service worker in V1?** Adds complexity but significantly improves UX on re-use. (Recommendation: defer to V1.1)

---

## 16. Claude Code Prompt (for One-Shot Build)

When ready to build, the prompt to Claude Code should be:

> Build a single-file HTML dashboard called `school-data-dashboard.html` based on the architecture in `school-data-dashboard-plan.md`. Implement all V1 scope (sections 11.1). Use CDN links for all libraries. Ensure CSV parsing runs in a Web Worker. Support both Generic mode and PowerSchool mode with auto field detection per section 3.1. Include all V1 chart types. Implement clipboard, PDF, and PNG export. No backend, no build step, no dependencies beyond CDN. Test with a 50k-row synthetic CSV before declaring done.

---

**End of Architecture Plan**
