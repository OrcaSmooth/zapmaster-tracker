# Zapmaster Session Tracker — Claude Context

## What this is

A single-file web app (`index.html`) for tracking electrology student procedures toward
graduation, plus a separate tracker for non-clinical assignments. No build step, no dependencies
to install. Open it with a browser from `file://`. All state is in `localStorage`. Chart.js and
patternomaly load from CDN.

## Files

- `index.html` — the entire app (inline CSS + JS, ~2050 lines)
- `spec.md` — detailed feature spec; read this before adding anything substantial
- `zapmaster-sessions.json` — exported data backup (not loaded automatically)
- `todo.md` — feature backlog; checked items are done

## Architecture in one paragraph

Everything is in one `<script>` block. `S` is the in-memory state object (settings,
sessions, calendar, assignments, UI flags). `persist()` saves to localStorage. Views are rendered
by `renderReports()`, `renderCalendar()`, `renderSessions()`, `renderSettings()`,
`renderAssignments()`. Navigation is hash-based (`#/reports`, `#/calendar`, `#/sessions`,
`#/settings`, `#/assignments`, `#/session/<id>`, `#/day/<YYYY-MM-DD>`, `#/assignment/<id>`);
`navTo(route)` sets the hash, `router()` dispatches to `showView()`, `renderSessionView(id)`,
`renderDayView(iso)`, or `renderAssignmentView(id)`. Modals are built with `openModal(html)`
/ `closeModal()`. Charts (Chart.js) are tracked in `S.charts` and destroyed when leaving
Reports (also destroyed when entering the Assignments pseudo-views).

## localStorage keys (all prefixed `zap_`)

| Key | Contents |
|---|---|
| `zap_settings` | `studentName`, `schoolStartDate`, `totalProcedures`, `observationProcedures`, `schoolHoursNeeded`, `formId`, `soapEmail`, `requirements{galvanic,flashThermolysis,manualThermolysis,blend}`, `untracked{…}` |
| `zap_sessions` | Array of session objects; optional `patientId` field = 8-char SHA-256 hex (see spec) |
| `zap_calendar` | Map `YYYY-MM-DD → {plannedHours, servedHours, status, note}` |
| `zap_assignments` | Array of assignment objects: `{id, title, category, dueDate, plannedDate, submittedDate, submitted, assignmentLink, submissionLink, notes}` |
| `zap_view` | Last active tab name; fallback when hash is empty |

## UI state flags (on `S`, not persisted)

| Flag | Values | Purpose |
|---|---|---|
| `S.propMode` | `'proc'` (default) / `'sess'` | Proportions doughnut weighting |
| `S.progMode` | `'proc'` (default) / `'hours'` | Progress over time chart mode |
| `S.assignmentSort` | `{key,dir}` or `{preset}` | Assignments table sort column/direction or named preset (see `ASSIGNMENT_PRESETS`) |
| `S.assignmentFilter` | filter fields object | Active Assignments filter (text/category/submitted/date ranges) |

## Key helper functions

- `schoolDays()` — calendar entries with `plannedHours > 0`, sorted chronologically
- `schoolDaysThrough(dateStr)` — count of school days on or before a date (used for pace line)
- `totalPlannedHours()` — sum of all `plannedHours` across school days
- `patientIdFromName(name, salt)` — async; normalizes name (lowercase, strip non-ASCII/spaces),
  SHA-256 hashes `normalized+salt` via Web Crypto, returns first 8 hex chars
- `patientIdSVG(hexId, size)` — returns inline SVG string for a 5×5 symmetric identicon using the
  Wong colorblind-safe palette (`IDENTICON_COLORS`); safe to embed directly in HTML template strings
- `patientSectionHTML(existingId)` — returns the patient section HTML for the session form; when
  `existingId` is truthy renders read-only identicon + Regenerate/Remove buttons; otherwise renders
  name+salt inputs + Generate button
- `soapNeededOnDate(iso)` — whether any non-future session on that date lacks a SOAP submission;
  drives the Calendar's 🧼 day indicator
- `sortAssignments(list)` / `matchesAssignmentFilter(a)` — apply `S.assignmentSort` /
  `S.assignmentFilter` to the assignments array
- `parseCSV(text)` / `csvRowsToAssignments(rows)` / `importAssignmentsCSV(file)` — CSV import
  pipeline for backfilling assignments; header names are matched case-insensitively via
  `CSV_HEADER_MAP`, and only columns present in the CSV are written so a partial re-import doesn't
  blank other fields. Title collisions route through `openCSVConflictModal` (update / duplicate /
  skip, per-row or bulk).

## Key invariants

- **Future sessions** (end time not yet passed, or date not yet past) are excluded from all
  stats, Reports roll-ups, and SOAP-needed counts.
- **Consults** always count 0 procedures regardless of modality inputs.
- Times are stored as UTC ISO instants; entered and displayed in Seattle/Pacific (`America/Los_Angeles`). Use `pacificToUTC` / `utcToPacific` — never touch `new Date(wall)` directly.
- **Patient name is never stored** (HIPAA). Only session date is required. The optional
  `patientId` field is a one-way hash (SHA-256, first 8 hex chars) of the normalized name + a
  user-provided salt — the raw name cannot be recovered from the stored ID. On new sessions,
  switching the appointment type to "Self work" auto-fills `patientId` from the most recent Self
  work session that has one; switching away undoes it unless the user manually generated a different
  ID (tracked by `autoFilledPidValue` closure variable in `openSessionForm`).
- The "Colleague" appointment type maps to "Service" on the Google Form (`APPT_FORM_MAP`).
- Galvanic requirement = SN + MN combined. Observation procedures count toward the total but
  are pre-logged (not individual sessions); they subtract from the student-choice bucket.

## How to test

Open `index.html` directly in a browser (`file://`). Use the "Load Data" button to import
`zapmaster-sessions.json` if you need real data. No server needed.

## Editing conventions

- All HTML is built as template strings in JS — use `esc()` on any user-supplied value before
  interpolating into HTML.
- Add new nav-tab views by: adding a `<section>` in the HTML skeleton, a `renderXxx()` function,
  an entry in `showView`'s `map`, and a nav tab button. Pseudo-views (like session detail and day
  detail) skip `showView` — they manage the `.active` class directly and highlight the nearest
  parent tab manually.
- `buildMiniCalendar(selectedISO, viewISO)` renders the mini month calendar on the day detail
  page. `selectedISO` is the day being viewed (highlighted with `mc-selected`); `viewISO` controls
  which month is displayed (defaults to `selectedISO`'s month). Always outputs 42 cells (6 rows)
  by padding with filler `&nbsp;` cells. Month nav arrows (`data-minical-to`) are handled via
  event delegation on the view container so re-renders don't break wiring.
- The `spec.md` is authoritative for behavior. Update it when behavior changes.
- Check `todo.md` for open items before adding something that may already be planned.
