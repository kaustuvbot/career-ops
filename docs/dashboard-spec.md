# Dashboard — Interactive Application Tracker

## Overview

An interactive single-page HTML application for browsing and managing job applications. Opens directly in a browser — no build step, no server required.

**File:** `dashboard/index.html`

---

## Layout

```
+------------------------------------------------------------------+
|  HEADER: "Career Ops Dashboard" + summary stats (total, avg)    |
+------------------------------------------------------------------+
|  FILTERS BAR                                                     |
|  [Score Range ▼] [Company ________] [Role ________] [Status ▼]  |
+------------------------------------------------------------------+
|  APPLICATION LIST (scrollable table)                             |
|  # | Date   | Company | Role        | Score | Status | PDF |    |
|  ----------------------------------------------------------------|  |
|  Click row → opens Report Detail panel                           |
+------------------------------------------------------------------+
|  DETAIL PANEL (slides in from right, 400px wide)                |
|  [Report] [Cover Letter] tabs                                    |
|  --------------------------------------------------------------  |
|  Report/Cover Letter content rendered here                       |
+------------------------------------------------------------------+
```

- **Header:** Fixed height (~60px). Shows total applications count and average score.
- **Filters Bar:** Fixed below header. Four filter controls in a row.
- **Application List:** Scrollable table filling remaining vertical space.
- **Detail Panel:** Overlay panel on the right side. Hidden by default. Toggles open when a row is clicked.

---

## Components

### Filter Bar

| Control | Type | Behavior |
|---------|------|----------|
| Score Range | `<select>` dropdown | Options: All, 4.5+, 4.0-4.4, 3.5-3.9, <3.5 |
| Company | `<input type="text">` | Substring match, case-insensitive |
| Role | `<input type="text">` | Substring match, case-insensitive |
| Status | `<select>` dropdown | All, Evaluated, Applied, Interview, Offer, Rejected, Discarded, SKIP |

- Filters apply immediately on change (no submit button).
- Multiple filters combine with AND logic.
- Clearing a text input or selecting "All" removes that filter.

### Application List Table

Columns:

| Column | Source | Behavior |
|--------|--------|----------|
| # | Row number | Sequential |
| Date | `applications.md` date column | Displayed as YYYY-MM-DD |
| Company | `applications.md` company column | Plain text |
| Role | `applications.md` role column | Plain text |
| Score | `applications.md` score column | Color-coded badge |
| Status | `applications.md` status column | Editable dropdown |
| PDF | PDF emoji from `applications.md` | ✅ or ❌ |
| Report | Link icon | Opens `reports/{num}-{slug}-{date}.md` in new tab |
| URL | — | Icon button → opens job URL in new tab |

**Row colors:** Subtle background tint matching score color (lighter versions of the score colors, ~10% opacity).

**Row click:** Opens the Detail Panel with that application's report.

**Score color coding:**

| Score | Color |
|-------|-------|
| 4.5+ | `#22c55e` (green) |
| 4.0–4.4 | `#86efac` (light green) |
| 3.5–3.9 | `#eab308` (yellow) |
| <3.5 | `#ef4444` (red) |

### Status Dropdown (Inline Edit)

- Each row's Status cell is a `<select>` dropdown with all canonical states.
- On `change` event:
  1. Update the DOM immediately for responsiveness.
  2. Write the new status to `applications.md` using a simple regex replacement on the matching row.
  3. If write fails, revert the DOM change and show a brief error toast.

**Why direct edit:** The dashboard is a local interactive tool, not a batch processor. Direct edits keep the file in sync without indirection through TSV + merge-tracker.

### Detail Panel

- Width: 420px, full viewport height, positioned fixed right.
- Background: white with subtle left shadow.
- Close button (X) in top-right corner.
- Two tabs: **Report** and **Cover Letter**.

**Report Tab:**
- Reads `reports/{num}-{company-slug}-{date}.md`.
- Renders markdown as formatted HTML (simple parser: headers, bold, italic, lists, code blocks, links).
- If no report exists, shows "No report yet."

**Cover Letter Tab:**
- Looks for `cover-letters/{num}-{company-slug}-{date}.md`.
- Same markdown rendering.
- If no cover letter exists, shows "No cover letter yet."

**Empty state:** If neither exists, panel shows a placeholder message.

### Header Summary Stats

- **Total:** Count of all applications.
- **Avg Score:** Arithmetic mean of all numeric scores, displayed as `X.X`.
- Updates reactively when filters change (shows filtered count / filtered avg).

---

## Data Sources

| File | Purpose |
|------|---------|
| `applications.md` | Main tracker table — read on load and on status change |
| `reports/*.md` | Evaluation report content |
| `cover-letters/*.md` | Cover letter content (optional) |

**applications.md parsing:** Parse the markdown table on load. Cache parsed rows in memory. Re-parse only after a status write succeeds.

**Report/Cover Letter loading:** Fetch via synchronous XMLHttpRequest (same-origin, no CORS since all files are local). Parse filename from the report link in the tracker row to construct the path.

---

## Interaction Flows

### Initial Load
1. Parse `applications.md` from disk.
2. Render table rows.
3. Compute and display header stats.

### Filter Change
1. Apply all active filters to cached rows.
2. Re-render visible rows.
3. Update header stats to reflect filtered set.

### Row Click → Open Detail
1. Extract report path from row's report link.
2. Fetch `reports/{num}-{slug}-{date}.md`.
3. Parse markdown to HTML.
4. Populate Detail Panel Report tab.
5. Attempt to fetch cover letter (may not exist — show empty state gracefully).
6. Slide panel in from right.

### Status Change
1. Capture new value from dropdown.
2. Read `applications.md` as text.
3. Find the row by matching `num` (first column) and current date/company.
4. Replace old status with new status in that row's columns (status is column 5, score is column 6).
5. Write back to `applications.md`.
6. Update cached row data.
7. Recompute header stats.

### Close Detail Panel
1. Slide panel out to right.
2. Clear panel content.

---

## Responsive Behavior

- Minimum supported width: 1024px (laptop screens).
- Below 1024px: horizontal scroll on table (no column hiding).
- Detail panel narrows to 360px on screens 1024–1280px.
- Filter bar wraps to two rows if needed on narrow windows.

---

## Technical Stack

- **HTML5** single file (`dashboard/index.html`).
- **Vanilla JavaScript** (ES6+, no framework).
- **CSS** embedded in `<style>` tag (no external dependencies except fonts).
- **Markdown parsing:** Lightweight custom parser (~50 lines) handling headers (#, ##), bold (**text**), italic (*text*), lists (-, *), code blocks (```), inline code (`), and links ([text](url)).
- **File access:** XMLHttpRequest for local file reading. File write via `fs` module only available in Node.js context — for the browser-only version, status edits require the user to grant filesystem access via the File System Access API where supported, or fall back to copy-to-clipboard with instructions.

**File System Access API fallback:** If `window.showSaveFilePicker` is available (Chrome/Edge), use it to save the modified `applications.md`. If not (Firefox, Safari), copy the new file content to clipboard and show a toast: "Status updated in clipboard — paste into applications.md manually."

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| applications.md not found | Show error message in table area: "Tracker not found. Run career-ops to add applications." |
| Report file not found | Show "Report file not found" in panel |
| Cover letter not found | Show empty state: "No cover letter yet." |
| Status write fails | Revert DOM change, show error toast |
| Markdown parse error | Display raw text as preformatted block |

---

## File Structure

```
career-ops/
  dashboard/
    index.html    # Complete single-file app
  applications.md  # Referenced data file
  reports/         # Referenced directory
  cover-letters/   # Referenced directory
```

---

## Open from Browser

Double-click `dashboard/index.html` or open via `file://` URL:

```
file:///Users/kaustuv/WorkStation/Repos/career-ops/dashboard/index.html
```

Requires no HTTP server. All file access is via same-origin XMLHttpRequest to sibling files in the parent directory.
