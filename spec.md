# Zapmaster Session Tracker — spec

A single self-contained `index.html` app for tracking electrology student procedures toward
graduation, planning/tracking school days, tracking non-clinical assignments, and generating
prefilled Google Form (SOAP charting) submissions. Runs from `file://`, stores everything in the
browser (`localStorage`), and loads Chart.js and patternomaly (colorblind-safe chart textures)
from CDN.

**Privacy:** Patient name is never stored or prefilled (HIPAA). Only the session **date** is
required; every other field is optional. A HIPAA-safe **patient identifier** can optionally be
attached to a session: the user provides a patient name + a shared salt phrase; the name is
normalized (lowercased, non-ASCII and spaces stripped), concatenated with the salt, hashed with
SHA-256 via the Web Crypto API, and the first 8 hex characters of the digest are stored as
`patientId`. The raw name and salt are never persisted.

## Files

- `index.html` — the whole app (inline CSS + JS, no build step).
- `spec.md` — this document.

## Data model (localStorage, `zap_` prefix)

- `zap_settings`: `studentName` (also used as the default technician on new sessions),
  `schoolStartDate` (ISO), `totalProcedures` (default 400), `observationProcedures` (default 12 —
  count toward the total but are never logged as sessions; shown as their own always-complete row in
  the remaining-by-type table and subtracted from the student-choice bucket; edited in the
  required-procedures-by-modality panel and included in its "student choice remainder" hint math),
  `schoolHoursNeeded` (default 400 — clinical-hours goal used by the
  Outstanding-Tracking planning check), `soapEmail` (email prefilled into the Google Form's
  email-address field; set during onboarding, editable in Settings), `untracked`
  `{ galvanic, flashThermolysis, manualThermolysis, blend, studentChoice }` (backfilled procedures
  already completed that you don't want to log as individual sessions; count toward totals / the
  "Done" column but not "Tracked"), `formId` (default `1FAIpQLSdAs88YbezDuxVPQptIynrqfVeMlQHNWoC8lcoFHmIXYTBrmg`),
  `requirements` `{ galvanic:20, flashThermolysis:46, manualThermolysis:46, blend:46 }`
  (these do **not** sum to `totalProcedures`; the remainder is a "student choice" bucket).
- `zap_view`: the last-used tab name, written by `showView`; used as the fallback target when the
  URL hash is empty or unrecognized (see **Routing**). Refresh normally restores the exact page from
  the hash; this is the default when there is none (defaults to Reports).
- `zap_sessions`: array of session objects, each with `id`, `date`, `patientId` (optional 8-char
  hex SHA-256 digest — see Privacy above), `startTime`/`endTime`
  (optional appointment start/end, stored as **UTC** ISO instants; entered and displayed in
  Seattle/Pacific time — the Sessions table and exports order by actual start instant so same-day
  sessions sort by which appointment was first), `appointmentType`
  (Consult/Service/Colleague/Self work — "Colleague" is tracked separately in-app but submitted to
  the Google Form as "Service"), `technician`, SOAP fields (`subjective/objective/assessment/plan`),
  `modalities` (map of form-modality name → procedure count), `machines[]`, `megahertz[]`,
  `filamentMaterial[]`, `filamentSize[]`, `filamentBrand[]`, `levelsUsed`, `hairDensityRemoved[]`,
  `hairColorsPresent[]`, `bonusDiscussed[]`, `adverseEventNote`, `timeSpentText`,
  `soapSubmitted` (bool), `bodyAreas[]`, `bodyAreasNote`, `myNotes` (private — never sent to the
  Google Form). Procedure count = sum of modality
  counts; **Consults always count 0**. A session is **"future" (not yet happened)** until its
  `endTime` has passed (or, if it has no `endTime`, until its `date` is past today); future sessions
  are excluded from every stat/report roll-up and from SOAP-needed tracking (see **Reports**).
- `zap_calendar`: map `YYYY-MM-DD` → `{ plannedHours, servedHours, status, note }` where `status`
  is `default | school-in | school-out`. M–F are school days unless `school-out`; weekends are
  not unless `school-in`.
- `zap_assignments`: array of non-clinical assignment objects (school coursework, unrelated to
  clinical sessions), each with `id`, `title`, `category` (free text, autocompleted from
  previously-used categories), `dueDate`, `plannedDate`, `submittedDate`, `submitted` (bool),
  `assignmentLink`, `submissionLink`, `notes`.

## Routing

Navigation is driven by the URL **hash** (`#/reports`, `#/calendar`, `#/sessions`, `#/patients`,
`#/settings`, `#/assignments`, `#/session/<id>` for a session detail page, `#/patient/<id>` for a
patient detail page, `#/day/<YYYY-MM-DD>` for a calendar day detail page, and `#/assignment/<id>`
for an assignment detail page). Hash routing is used
instead of `?query` params because the app runs from `file://`: a query-string change would force
a reload the browser can't reliably service, whereas hash changes never reload and support browser
back/forward. `navTo(route)` writes the hash (or re-runs the router if the hash is unchanged); a
`hashchange` listener and the initial boot both call `router()`, which dispatches to
`renderSessionView(id)`, `renderDayView(iso)`, `renderAssignmentView(id)`, or `showView(tab)`. An
empty/unrecognized hash falls back to the last-used tab remembered in `zap_view` (else Reports).
Navigation goes through `navTo`; **in-place refreshes** (edit-save, filtering) call the render
functions directly since the hash doesn't change.

## Pages

- **Reports** — **future sessions are excluded from everything on this page** (procedures done,
  the Tracked/Done table, the cumulative-procedures chart, the proportion doughnuts, and the
  SOAP-needed count) since they haven't happened yet. Contents: remaining-per-type table with
  **Required / Tracked / Done / Remaining** columns
  (Galvanic = SN+MN; "Tracked" = logged sessions only, "Done" adds untracked backfill + observation;
  plus student-choice + total rows; Remaining cells show ✓ in green when the count reaches 0;
  numbers in this table are right aligned); 
  an **Outstanding Tracking** panel that flags past planned school
  days missing served hours (links to that month's Calendar), sessions missing SOAP notes (links to
  the Sessions page filtered to SOAP-needed), and a planning gap when planned + performed hours fall
  short of `schoolHoursNeeded` (links to the Calendar), or "all caught up" when clear; pace cards
  (procedures done/remaining; school days remaining = future days with planned-but-unserved hours,
  annotated with estimated hours performed to date [logged served hours, or planned hours as the
  estimate for past days not yet logged] and a second line showing the average hours/day needed
  across remaining planned school days to reach `schoolHoursNeeded` — highlighted in warn color
  when > 8 h/day; procedures needed per day; **Assignments remaining** = count of non-clinical
  assignments not yet marked submitted, with the earliest-due unsubmitted assignment's title/date
  as a subtitle — clicking the card navigates to Assignments filtered to unsubmitted); a **Progress over time** line chart with a
  **Procedures / School Hours** toggle in the panel header — **Procedures** (default) shows
  cumulative procedure count (baseline = observation + untracked backfill) plus a dashed goal
  line and a dashed gray **Even pace** line (what accumulation would look like if procedures were
  distributed evenly across all planned school days); **School Hours** shows two stacked area
  series — a background area for cumulative planned hours and a foreground area for cumulative
  served hours — plus the same Even pace and goal lines (goal = total planned hours across all
  school days; pace denominator = total school-day count); and proportion doughnuts for
  appointment type, modalities, machines, and **separate** filament material, brand, and size
  charts. A single **By procedure / By session** toggle in the Proportions header switches all
  doughnuts between procedure-weighted and session-count weighting (in session mode a session
  counts once per modality/value it used); hovering a legend item shows that category's count.
- **Calendar** — month grid (starts at school-start month, bounded prev; a **Today** button jumps
  back to the current month). Each day shows procedure count + planned/served hour badges + note
  dot + a 🧼 icon when any session on that date is missing its SOAP submission (past/current
  sessions only — see legend). Day types are color-coded with a legend; planned-but-unserved days
  are highlighted amber ("needs filling"). Clicking a day cell navigates
  to the **Day detail page** (`#/day/<YYYY-MM-DD>`). A pencil icon (✏, revealed on hover,
  positioned bottom-right of the cell) opens the inline edit modal for hours, status override, and
  note without leaving the calendar.
- **Day detail** (`#/day/<YYYY-MM-DD>`) — dedicated page for a single school day. Contains: a
  **mini month calendar** (school days colored green when served hours logged, amber when planned
  but unserved; each date is clickable to navigate to that day; always renders 6 rows of dates for
  consistent height; ← / → arrows in the title bar navigate to the previous/next month, bounded on
  the left by the school-start month); **← Prev / Next →** buttons that jump to the
  previous/next school day (skipping weekends and school-out overrides); a **Day Summary** panel
  (planned hours, served hours, status, session count, note) with an **Edit Day** button that opens
  the same edit modal as the calendar pencil; a **Sessions** panel listing every session for that
  date as detail cards (appointment type, time range, technician, SOAP status, modalities summary,
  machine + MHz, filament material/size/brand, levels used, body areas, private notes, View and
  Edit buttons); and a **+ New Session** button that opens the session form with the date
  pre-filled to this day.
- **Sessions** — summary table (columns: SOAP-status flag first — green ✓ when submitted, amber
  circled-`!` badge when not (a neutral `—` for future sessions whose end time hasn't passed); the
  flag is a button — click it to open a small dialog to set SOAP-submitted on/off, which saves and
  refreshes the table in place — then Patient (identicon SVG + 8-char hex ID, or `—` if none),
  Date, Time (Pacific start–end range), Type, Proc., Modalities rendered one-per-line with the
  count as a leading badge, Machines, and a **Filament** column-group header spanning Material /
  Size / Brand) + a **Filter** modal (free-text search across SOAP/notes/technician/
  areas, plus type / modality / machine / filament material·size·brand / SOAP-status / date-range
  filters, and a **Patient** section — enter an 8-char ID directly, or generate one from name + salt
  via "Generate & Apply"; the SOAP "not submitted" option excludes future sessions, matching the
  Reports SOAP-needed count). Machine and the three filament dropdowns also offer a "— Missing / none —" option to
  surface sessions with gaps in that field. When a filter is
  active the table shows a banner summarizing it with a Clear-filter button; filter state is
  in-memory only. Plus New/Edit Session form (patient section at top: name + salt → "Generate ID"
  shows identicon preview + 8-char hash; for existing IDs: "Regenerate…" reveals inputs, "Remove"
  clears after confirmation — ID is never directly editable; **new sessions only:** changing the
  appointment type to "Self work" auto-fills the `patientId` from the most recent Self work session
  that has one (shown with an "Auto-filled" note); switching away from "Self work" undoes the
  auto-fill unless the user manually generated or edited the ID in the meantime; per-modality
  counts auto-sum to a live total; auto-generated editable "Time spent per area" text; checkbox
  groups for machines/megahertz/filaments/body areas). Each row's View page shows all fields (including patient
  identicon + ID when set) and a **Submit to Google Form**
  button; the **Date of Service** field is a link that navigates to the Day detail page for that date.
- **Patients** — summary table of every distinct `patientId` seen across sessions (rows are
  clickable). Columns:
  Patient (identicon + 8-char hex ID), Sessions (count), Procedures (sum across those sessions),
  SOAP needed (count of past/current sessions for that patient lacking a submitted SOAP note),
  Last session (most recent session date) — sorted by last session date, most recent first. Clicking
  a row opens the **Patient detail page** (`#/patient/<id>`): a summary panel (session count,
  procedure count, SOAP-needed count, first/last session date) plus a table of that patient's
  sessions (SOAP status, Date, Time, Type, Proc., Modalities), each row's View button navigating to
  the existing **Session detail page** (`#/session/<id>`).
- **Settings** — General (student name [doubles as default technician], school-start date, total
  procedures, school hours needed); required-procedures-by-modality (the four modality requirements
  + observation, with a live "student choice remainder" = total − these); untracked/backfilled
  procedures (per-modality counts + student choice); Google Form ID + SOAP-submission email; and a Data panel with JSON
  backup/restore (Save/Load Data) and an Export ▾ menu for CSV/TSV of the sessions table.
- **Assignments** — tracker for non-clinical coursework, independent of session/procedure data. A
  summary line above the table shows "N of M submitted — N remaining" (counts are unaffected by any
  active filter).
  Table columns (Title, Category, Due, Planned, Submitted) are click-to-sort (toggling asc/desc),
  or a **Sort** dropdown applies a named preset (Due soon first; Unsubmitted first, then due date;
  By category, then due date; Recently planned). A **Filter** modal matches free text (title/
  category/notes) plus category, submitted status, and due/planned date ranges; an active filter
  shows a summary banner with a Clear-filter button. **Export CSV** / **Import CSV…** round-trip the
  table (headers: Title, Category, Assignment Link, Submission Link, Submitted, Notes, Planned
  Date, Submitted Date, Due Date — case-insensitive on import, and only columns present in the CSV
  are touched so a partial re-import doesn't blank other fields); dates in a variety of formats are
  normalized to ISO on import. Importing rows whose title matches an existing assignment opens a
  conflict-resolution modal (per-row or bulk: Update existing / Add as new duplicate / Skip). **+
  New Assignment** and each row's **Edit** button open a form (Title *required*, Category
  [autocompleted from existing categories], Due/Planned/Submitted dates, Submitted checkbox,
  Assignment/Submission links, Notes). Each row's title links to an **Assignment detail page**
  (`#/assignment/<id>`) showing all fields with Edit/Delete buttons.

## Google Form prefill

`buildFormUrl(session, includeSoap)` maps fields to `entry.<id>` params (checkbox groups repeat
the param per value; the date uses `entry.<id>_year/_month/_day`). When `settings.soapEmail` is
set it also prepends the top-level `emailAddress=` param, which prefills the Google Form's
"Collect email addresses → Responder input" field. The **Submit to Google Form**
button opens the URL **in a new tab**. If the URL exceeds ~8000 chars, it rebuilds without the
Subjective/Objective/Assessment/Plan params and warns that those four must be transferred manually.

## Export / import

- **Save Data** / **Load Data** buttons = JSON full backup `{settings, sessions, calendar,
  assignments}` (download + FileReader import that replaces all data after confirmation). The
  first-run onboarding modal also offers **Load a data backup (.json)** to restore instead of
  entering values (fresh setup, so it skips the replace confirmation).
- **Export ▾** dropdown = CSV / TSV of the sessions table (array fields joined with `; `).
- Assignments have their own **Export CSV** / **Import CSV…** buttons on the Assignments page
  (separate from the sessions Export ▾ menu and the Settings Save/Load Data backup) — see
  **Assignments** above.
