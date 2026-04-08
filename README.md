# TimeTracker

A simple, single-file web application for logging time spent on freelance work. Track tasks with a stopwatch-style timer, review a summary of your day, and export/import the results as CSV.

## Features

- **Stopwatch Timer** — Enter a task name and start a running timer. The display counts up in `HH:MM:SS` format and updates every second. Stop the timer when you're done, and the session is logged automatically. A "Stopped at" timestamp (e.g., "Stopped at 3:12 PM, 4/8/2026") appears after stopping, so you can tell how long ago you paused and manually account for any untracked time.
- **Editable Task Names** — Task names can be changed while the timer is running (the input field stays editable) or after the fact by clicking any task name in the activity log. The "Tracking: ..." label updates live as you type.
- **Optional Tags** — Add comma-separated tags (e.g., "Video Editing, General Chemistry") to any task via the tag input field below the task name. Tags display as pills in the log and summary, and are included in CSV export/import. Tags can also be edited inline by clicking a task in the log.
- **Resume Tasks** — Each entry in the activity log has a "Resume" button. Clicking it restarts the timer for that task, and the timer display picks up from the previously accumulated time (e.g., if a task had 5 minutes logged, the display starts at `00:05:00`). When stopped, the new elapsed time is added to the existing entry rather than creating a new one. This lets you switch between tasks without cluttering the log.
- **Delete Entries** — Each entry in the activity log has an **x** button to remove it individually, without needing to clear the entire log. The button is disabled for the currently-running entry.
- **Task Activity Log** — Every completed timer session appears in a chronological list showing the task name, tags, and duration. A running total is displayed at the bottom.
- **Activity Summary** — A "Summarize" button opens a modal that aggregates all sessions by task name. If you worked on the same task multiple times throughout the day, the durations are combined into a single entry. Tags from all sessions of the same task are merged. The current date is displayed at the top of the summary.
- **CSV Export** — From the summary modal, click "Export CSV" to download a file. The CSV contains four columns: `Date` (YYYY-MM-DD), `Task`, `Duration` (HH:MM:SS), and `Tags` (semicolon-separated). The filename is automatically set to `time-tracker-YYYY-MM-DD.csv`. Fields containing commas or quotes are properly escaped per RFC 4180.
- **CSV Import** — Click "Import" in the activity log header to load a previously exported CSV file. Imported entries appear in the activity log and are fully functional — they can be edited, resumed, summarized, and re-exported. The parser handles quoted fields and the exact format produced by the export function.
- **Clear Log** — A "Clear" button resets all recorded activities after a confirmation prompt.
- **Auto-Save** — All activity data and running timer state are automatically saved to the browser's `localStorage` after every change. If you accidentally close the tab, reopening `index.html` restores your full activity log and resumes any running timer (including elapsed time while the tab was closed). Clearing the log also clears the saved state.

## Tech Stack

This is a **zero-dependency, single-file implementation**. Everything — HTML structure, CSS styling, and JavaScript logic — lives in `index.html`. There is no build step, no framework, and no external libraries.

- **HTML** — Semantic structure with a timer section, an activity log section, and a summary modal overlay. A hidden file input handles CSV import.
- **CSS** — Dark theme UI with a `#0f1117` background. Uses CSS flexbox for layout, `border-radius` for rounded cards, and `font-variant-numeric: tabular-nums` so timer digits don't shift as values change. All styles are embedded in a `<style>` block.
- **JavaScript** — Vanilla JS using `setInterval` for the timer tick and `Date.now()` for elapsed-time calculation. Activity entries are stored in an in-memory array of `{ task, ms, tags }` objects and persisted to `localStorage`. CSV generation and parsing use the Blob API, FileReader API, and a custom RFC 4180-compliant parser. All user-supplied text is escaped via a `textContent`/`innerHTML` roundtrip to prevent XSS.

## How to Use

1. Open `index.html` in any modern web browser.
2. Type a task name (e.g., "Chapter 3-4 video editing") into the input field.
3. Optionally add tags in the field below (e.g., "Video Editing, General Chemistry").
4. Click **Start** or press **Enter** to begin the stopwatch.
5. Work on your task. The timer counts up in real time. You can edit the task name while the timer runs.
6. Click **Stop** when you're done. The session is saved to the activity log and a "Stopped at" timestamp is shown.
7. To switch tasks: stop the current timer, start a new one. To go back, click **Resume** on the previous entry — time is added to the same entry.
8. Click any task name in the log to edit it or its tags after the fact.
9. When you're finished, click **Summarize** to see an aggregated breakdown.
10. Click **Export CSV** to download the summary as a `.csv` file.
11. To load a previous session, click **Import** and select a CSV file exported by this app.

## File Structure

```
TimeTracker/
├── Design Specs.txt   # Original product requirements
├── README.md          # This file
└── index.html         # The complete application (HTML + CSS + JS)
```

## Implementation Details

### Timer Engine

The timer uses `Date.now()` timestamps rather than incrementing a counter, so elapsed time remains accurate even if the browser throttles `setInterval` (e.g., when the tab is in the background). The interval fires every 1000ms to update the display.

### Data Model

All activity data is held in a plain JavaScript array (`entries`). Each entry is an object with three properties:

- `task` (string) — the name the user typed
- `ms` (number) — elapsed time in milliseconds
- `tags` (string[]) — optional array of tag strings

A `resumingIndex` variable tracks whether the current timer session is resuming an existing entry. When non-null, stopping the timer adds elapsed time to `entries[resumingIndex]` instead of pushing a new entry. A `timerOffset` variable holds the previously accumulated milliseconds so the timer display continues from where the entry left off.

### Aggregation

The summary groups entries by exact task name and sums their durations. This means "Bug fix" and "bug fix" are treated as separate tasks. Tags from all entries sharing a task name are merged into a single set. The grouped data is displayed in the summary modal and used as-is for CSV export.

### CSV Format

The exported CSV follows RFC 4180 conventions:

```
Date,Task,Duration,Tags
2026-04-01,Chapter 3-4 video editing,01:23:45,Video Editing; General Chemistry
2026-04-01,Email and admin,00:32:10,
```

Fields containing commas, double quotes, or newlines are wrapped in double quotes, with internal quotes doubled (`""`) per the spec. Tags are joined with `; ` (semicolon-space) within the Tags column.

### CSV Import

The import parser reads the same four-column format. It uses a character-by-character state machine to correctly handle quoted fields with embedded commas and escaped quotes. The `Duration` column (`HH:MM:SS`) is converted back to milliseconds. The `Date` column is read but not used — imported entries join the current session's activity log. The `Tags` column is split on `;` to reconstruct the tag array.

### Local Storage Persistence

The app saves its full state to `localStorage` under the key `timetracker_state` after every mutation (start, stop, resume, edit, delete, clear, import). The saved state includes:

- The `entries` array (all logged tasks, durations, and tags)
- The running timer state, if active: start timestamp, task name, tag input value, `resumingIndex`, and `timerOffset`

On page load, `loadState()` restores the entries and, if a timer was running, reconstructs the full timer UI. Because the original `Date.now()` start timestamp is persisted, elapsed time is calculated correctly even if the browser was closed for hours. If the stored data is missing or corrupt, the app starts fresh.

### Security

User-provided task names are escaped before being inserted into the DOM. The `escapeHTML` helper creates a temporary `<div>`, sets its `textContent`, then reads back `innerHTML`, which neutralizes any embedded HTML or script tags. This applies to all rendering paths: the activity log, the summary modal, and tag pills.
