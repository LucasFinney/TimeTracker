# TimeTracker

A single-file web application for logging time spent on freelance work. Track tasks with a stopwatch-style timer, annotate each day with a description, browse the month at a glance, and export a printable monthly report or a detailed CSV log.

## Features

### Timing & activity log
- **Stopwatch Timer** — Enter a task name and start a running timer. The display counts up in `HH:MM:SS` and updates every second. Stopping the timer logs the session automatically and shows a "Stopped at" timestamp (e.g., "Stopped at 3:12 PM, 4/8/2026") so you can tell how long ago you paused.
- **Editable Task Names** — Task names can be changed while the timer is running, or after the fact by clicking any task name in the activity log. The "Tracking: ..." label updates live as you type.
- **Optional Tags** — Comma-separated tags (e.g., "Video Editing, General Chemistry") can be attached to any task. Tags display as pills in the log and can be edited inline.
- **Resume Tasks** — Each entry has a **Resume** button. Clicking it restarts the timer for that task with the previously accumulated time pre-loaded (e.g., a task with 5 minutes resumes at `00:05:00`). When stopped, the new time is added to the existing entry rather than creating a new one.
- **Delete Entries** — Each entry has an **×** button to remove it individually. Disabled for the currently-running entry.
- **Activity Log** — Every completed session appears in the log for the day being viewed, with a running total at the bottom.

### Days, descriptions, and the month view
- **Day View / Month View toggle** — A button in the header switches between viewing a single day's activities and a month-at-a-glance view.
- **Save Day** — A **Save day** button opens a modal showing a per-task summary, a total, and a free-text "Description for the day" (e.g., *"General Chemistry editing; Q28 slide review"*). Saving marks the day with a ✓ in the month view.
- **Month View** — Shows one row per day in the selected month with date, total hours, and the saved description. Click any row to jump into that day. Arrow buttons in the header step through months. Today is highlighted.
- **Date Warning** — When viewing a past or future day, a banner reminds you that new tracked time will be logged to that date and offers a **Jump to today** shortcut.
- **Clear Day** — Wipes all entries and the saved description for the day currently being viewed, after a confirmation prompt.

### Import / export
- **Detailed Log CSV Export** — From the month view, downloads a CSV with one row per `(date, task)`, columns: `Date`, `Task`, `Duration (hours)`, `Duration (HH:MM:SS)`, `Tags`, `Day Description`, plus a trailing **Month Total** footer row. Filename: `time-tracker-detailed-YYYY-MM.csv`.
- **Per-Day CSV Export** — From inside the Save Day modal, exports just the current day in the same column format.
- **CSV Import** — From the month view, loads a previously exported detailed-log CSV. The importer:
  - **Auto-populates day descriptions** from the `Day Description` column for days that don't already have one (existing descriptions are never overwritten).
  - **Skips duplicates** — entries are aggregated by `(date, task)` and compared at 2-decimal-hour precision against what's already tracked; matching rows are skipped and reported in a summary alert.
  - **Ignores the Month Total footer row** so re-importing your own export won't add a phantom entry.
- **Formatted Report (PDF)** — A **Report (PDF)** button on the month view renders a styled report into a hidden iframe and opens the browser's print dialog. Choose "Save as PDF" to get a file named `Time Report - <Month> <Year>.pdf`. The report contains:
  - **Page 1** — Cover with the month total, plus a Monthly Summary table (Date, Hours, Description, with unique-tag pills per day).
  - **Page 2+** — Detailed Log, one block per day: date heading, italic description, and a Task / Duration / Hours table with a day-total row. Tags are omitted from the detailed-log section to keep it readable.
  - If any day has a description but **zero tracked time**, the user is asked to confirm before exporting (this usually means logs were lost or the description is a manual note).

### Persistence
- **Auto-Save** — All entries, per-day descriptions, the selected date, the current view, the month cursor, and any running timer state are persisted to `localStorage` after every change. Reopening `index.html` restores everything, including a timer that was running when the tab closed (elapsed time keeps accruing in the background).

## Tech Stack

A **zero-dependency, single-file implementation**. HTML, CSS, and JavaScript all live in `index.html` — no build step, no framework, no external libraries.

- **HTML** — Semantic structure: timer section, activity log (day view), month grid (month view), Save Day modal, and a hidden printable iframe for the PDF report.
- **CSS** — Dark theme (`#0f1117` background), CSS grid for the month rows, `font-variant-numeric: tabular-nums` for stable digit widths, and a separate print-only stylesheet inlined into the report iframe (Letter portrait, page counters, alternating row tint).
- **JavaScript** — Vanilla JS. `setInterval` drives the display tick; `Date.now()` is the source of truth for elapsed time. State is held in two top-level structures (`entries`, `days`) and serialized to `localStorage`. CSV is generated/parsed in-place with a custom RFC 4180-aware parser. The PDF report is produced by writing styled HTML into a hidden `<iframe>` and calling `contentWindow.print()`.

## How to Use

1. Open `index.html` in any modern web browser.
2. Type a task name (e.g., "Chapter 3-4 video editing") and, optionally, tags.
3. Click **Start** (or press **Enter**) to begin the stopwatch.
4. Click **Stop** when you're done. The session is saved to the activity log.
5. Use **Resume** on any entry to keep accumulating time on that task.
6. Click any task name to edit it or its tags.
7. When the day's done, click **Save day**, review the summary, write a description, and **Confirm**. (Or **Export day to CSV** straight from the modal.)
8. Switch to **Month View** to browse days, jump into any of them, **Import** a past CSV, export the **Detailed Log** CSV, or generate the **Report (PDF)**.

## File Structure

```
TimeTracker/
├── Design Specs.txt   # Original product requirements
├── README.md          # This file
└── index.html         # The complete application (HTML + CSS + JS)
```

## Implementation Details

### Timer Engine
The timer uses `Date.now()` timestamps rather than incrementing a counter, so elapsed time stays accurate even if the browser throttles `setInterval` (e.g., when the tab is in the background). The interval fires every 1000 ms to update the display.

### Data Model
Two top-level structures hold all state:

- `entries: { id, date, task, ms, tags }[]` — every logged session. `date` is a local `YYYY-MM-DD` string so days don't shift with timezone changes.
- `days: { [YYYY-MM-DD]: { description, savedAt } }` — per-day descriptions and the timestamp the day was saved.

The currently-running timer also tracks `resumingId` (the entry being resumed, if any), `timerOffset` (the pre-loaded ms when resuming), and `timerEntryDate` (the date the running session will log to, which can differ from the day currently being viewed).

A migration pass on load rolls any pre-existing per-entry `description` field (legacy data) into the corresponding day's description so older saved state isn't lost.

### CSV Format (Detailed Log)
```
Date,Task,Duration (hours),Duration (HH:MM:SS),Tags,Day Description
2026-05-10,Social Post 5 editing,3.20,03:12:00,Video Editing; Social,General editing pass
2026-05-10,Email and admin,0.53,00:32:10,,General editing pass
,Month Total,3.73,03:44:10,,
```

Fields containing commas, quotes, or newlines are wrapped in double quotes with internal quotes doubled (RFC 4180). Tags are joined with `; ` within the Tags column. The trailing `Month Total` row has no date and is recognized and skipped by the importer.

### Import Deduplication
On import, both the existing `entries` array and the imported rows are aggregated by `(date, task_lowercase)`. For each key present in both, the totals are rounded to 2 decimal hours (the export's resolution) and compared; if they match, every imported row for that key is skipped. The user gets a summary alert with the import/skip counts.

This means re-importing your own detailed-log CSV is idempotent — no phantom duplicates, and existing entries are preserved as-is.

### PDF Report
The report is built as a complete HTML document (inline `<style>` with `@page` rules for Letter portrait, page counters, and `break-before: page` to put the detailed log on page 2). The document is written into a hidden, zero-size `<iframe>`, the iframe's title is set to `Time Report - <Month> <Year>` (browsers use this as the default print-dialog filename), and `contentWindow.print()` is called. CSS `break-inside: avoid` keeps each day's block from being split across pages where possible.

### Local Storage Persistence
The full app state is saved under the `timetracker_state` key after every mutation. The saved state includes `entries`, `days`, `selectedDate`, `view`, `monthCursor`, and any running timer (start timestamp, task name, tag input value, `resumingId`, `timerOffset`, `timerEntryDate`). On load, a migration function tolerantly upgrades legacy shapes; on corrupt data, the app starts fresh.

### Security
User-provided text (task names, tags, day descriptions) is escaped before being inserted into the DOM via a `textContent`/`innerHTML` roundtrip in `escapeHTML()`. This applies to every rendering path: the day log, the month grid, the Save Day modal, and the printed report.
