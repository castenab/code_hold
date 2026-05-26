# Org Chart Tool — CLAUDE.md

## Background & audience

Built for a **corporate HR team** to support VP-level org planning and budgeting discussions. The audience (VPs and their staff) is non-technical — no IDE, no code changes. The deliverable must be a single `.html` file that works by double-clicking.

**Hard constraints from requirements.txt:**
- 100% local — no data leaves the browser, ever
- No cloud tools (Mermaid, draw.io, etc. are ruled out)
- Single-file HTML: no build tools, no framework, no server

## What this is

`test.html` is a fully self-contained, single-file browser app that reads Workday HR data from an uploaded Excel workbook and renders an interactive org chart. No server, no build step, no framework. Open the file in a browser and it works.

## External dependencies (CDN)

- **SheetJS** (`xlsx.full.min.js`) — parses `.xlsx`/`.xls` files locally in the browser
- **html2canvas** — captures the chart DOM as a PNG for export

## Three screens

| ID | Name | Role |
|----|------|------|
| `screen-upload` | Upload | Drag-and-drop or file picker; shown on load |
| `screen-mapping` | Column mapping | Auto-detects columns; user confirms before building |
| `screen-chart` | Org chart | Interactive tree with filter bar + PNG export |

Screen switching is done by toggling `.active` on `.screen` elements via `showScreen(id)`.

## Four expected Excel sheets

| Key | Expected tab name (fuzzy-matched) | Purpose |
|-----|-----------------------------------|---------|
| `r005` | "R005", "employee list", etc. | Main headcount — source of truth |
| `newHire` | "New hire", "starters", etc. | People joining, may not be in R005 yet |
| `departures` | "Departures", "leavers", etc. | People leaving |
| `openReqs` | "Open reqs", "requisitions", etc. | Open headcount positions |

Tab name matching uses `normalize()` (trim + lowercase) + fuzzy `.includes()`.

## Application state

Everything lives in the `STATE` object (global). Key fields:

```
STATE.workbook         — raw SheetJS workbook
STATE.sheetMap         — { r005: "R005 Employee Data", newHire: "New Hires", ... }
STATE.headers          — { r005: ["First Name", "Last Name", ...], ... }
STATE.columnMap        — confirmed field-to-column mapping; r005 also has managementChainCols[]
STATE.employeeMap      — { [id]: employeeNode } — keyed by Employee ID
STATE.nameMap          — { "sarah chen": "EMP001" } — normalized name → ID lookup
STATE.roots            — array of root nodes (no parent, has children)
STATE.orphans          — array of nodes with no resolvable parent and no children
STATE.expandedNodes    — Set of currently expanded node IDs
STATE.filters          — { l4, l5, employmentType, globalGrade, search }
STATE.rawData          — { newHires, departures, openReqs } raw parsed arrays
```

## Data pipeline (in order)

1. `processFile()` — reads binary from FileReader, hands to SheetJS
2. `detectSheets()` — fuzzy-matches tab names → `STATE.sheetMap`
3. `extractHeaders()` — reads row[0] of each sheet → `STATE.headers`
4. `buildMappingUI()` — renders Screen 2; calls `autoMatch()` for each field
5. `confirmMapping()` — reads dropdowns → `STATE.columnMap`; calls `buildOrgData()`
6. `buildOrgData()` — main orchestrator:
   - `parseSheet()` → raw row arrays for all 4 sheets
   - `buildEmployeeMap()` → `STATE.employeeMap` from R005 rows
   - `buildNameMap()` → `STATE.nameMap`
   - `detectAndCreateRootNodes()` ← **must run before** `resolveManagerships()`
   - `resolveManagerships()` → sets `parentId` + fills `children[]`
   - `enrichNewHires()` → marks `isNewHire`, creates pending nodes
   - `enrichDepartures()` → marks `isDeparting`
   - `findRoots()` → populates `STATE.roots` and `STATE.orphans`
   - `showOrgChart()` → switches to Screen 3, renders tree

## Manager resolution — 3-priority rule

For each employee, the code tries in order:
1. **Worker's Manager field** — name match via `nameMap`
2. **Manager Level 02 field** — skip-level fallback
3. **Management Chain columns** — walk from deepest level upward; first non-self match wins

## Synthetic nodes

If a manager appears in MC Level 03 or Level 04 columns but has no R005 row, a synthetic placeholder node is created so the tree has something to connect to. Rendered with a dashed border (`node-card.synthetic`). Must be created before `resolveManagerships()` runs so their names are in `nameMap`.

## Node card color coding

Background color driven by `employmentType` via `getNodeColour()`:

| Type | Color |
|------|-------|
| Regular | `#AED6F1` (blue) |
| Contract | `#A9DFBF` (green) |
| Student / Co-op | `#F9E79F` (yellow) |
| Contingent / Other | `#D5D8DC` (grey) |
| Unknown | `#ffffff` (white) |

## Status badges (priority order, only one shown)

1. **Departing** — red; shows departure date if available
2. **On leave** — orange
3. **New hire / Pending** — dashed border; shows start date if available

## Connector lines

Drawn by `drawConnectors()` after each DOM paint using `requestAnimationFrame`. Uses `getBoundingClientRect()` relative to `#chart-canvas`. Draws elbow-shaped SVG `<path>` elements on the `#connector-svg` overlay (pointer-events: none).

## Filter behavior

- Filters live in `STATE.filters`; updated by `onFilterChange()`
- `nodeMatchesFilters(emp)` — checks L4, L5, employment type, grade, free-text search
- `subtreeHasMatch(empId)` — recursive; keeps parent nodes visible if any descendant matches
- In filter mode, children containers auto-expand to reveal matches; `STATE.expandedNodes` is ignored

## PNG export

`exportPNG()` uses html2canvas on `#chart-canvas` (not the whole page). Converts data URL → Blob → `blob://` URL to avoid corporate browser security blocks on data: URLs. Output: `org-chart-YYYY-MM-DD.png` at 2× scale.

## Code organization

JS is divided into 22 numbered sections in `<script>` at the bottom of the file, each with a block comment explaining purpose and approach. CSS is all inline in `<style>` in `<head>`, also thoroughly commented.

## Implementation status vs. requirements

### Built ✓
- Excel upload (drag-and-drop + file picker)
- Dynamic column/sheet detection via fuzzy alias matching
- Column mapping UI (Screen 2) with auto-match
- Org chart tree with expand/collapse (Screen 3)
- Employment type color coding (Regular/Contract/Student/Contingent)
- Status badges: Departing (red + date), On Leave (orange), New Hire (dashed + date)
- Headcount pills on manager nodes
- Synthetic nodes for managers in MC columns but absent from R005
- 3-priority manager resolution (Worker's Manager → Manager L02 → MC columns)
- Pending new-hire nodes (not yet in R005, placed under inferred L5 manager)
- Filter bar: L4, L5, Employment type, Global Grade, free-text search
- Filter-aware subtree reveal (parent visible if any descendant matches)
- PNG export via html2canvas (Blob URL approach for corporate browser compat)
- "Upload different file" reset

### Not yet built — pending from requirements
- **SVG export** — requirements specify `.svg` export; currently only PNG exists
- **Zoom controls** — requirements call for interactive zoom on the chart
- **Summary Table** — a full second panel covering:
  - Org snapshot: L4/L5 names, headcount, as-of date
  - Headcount by employment type (count + % of total)
  - Headcount by direct report / team size
  - What's Changing: joining/leaving/open reqs bucketed by 30/90/90+ days
  - Open positions: total, by team, by grade; flag 3+ reqs in 1 team; flag req where hiring manager is departing
  - Notes & Flags: employees with comments; contractor concentration >30%; departure with no backfill req; new hire whose manager is departing
- **Summary Table dynamic scoping** — must recalculate when L4/L5 filter changes
- **Cross-report unresolvable match warning** — surface "Could not verify [field] — manual check recommended" in Notes & Flags when a cross-report join fails

### Cross-report matching rules (from requirements)
- R005 ↔ New Hire / Departures: join on **Employee ID** (name match only if ID unavailable in both)
- Open Reqs "Hiring Manager" ↔ Departures "Resource Name": name-based (no ID in Open Reqs)
- All name matching: normalize (trim + lowercase) before comparing
- Each report analyzed independently; cross-report only for explicit flags

## File naming

The file is currently named `test.html` — treat it as the production artifact, not a throwaway. The project is built incrementally in sections as the user adds them.
