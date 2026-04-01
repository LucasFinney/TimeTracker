# TimeTracker

A simple, single-file web application for logging time spent on freelance work. Track tasks with a stopwatch-style timer, review a summary of your day, and export the results to CSV.

## Features

- **Stopwatch Timer** — Enter a task name and start a running timer. The display counts up in `HH:MM:SS` format and updates every second. Stop the timer when you're done, and the session is logged automatically.
- **Task Activity Log** — Every completed timer session appears in a chronological list showing the task name and duration. A running total is displayed at the bottom.
- **Activity Summary** — A "Summarize" button opens a modal that aggregates all sessions by task name. If you worked on the same task multiple times throughout the day, the durations are combined into a single entry. The current date is displayed at the top of the summary.
- **CSV Export** — From the summary modal, click "Export CSV" to download a file. The CSV contains three columns: `Date` (YYYY-MM-DD), `Task`, and `Duration` (HH:MM:SS). The filename is automatically set to `time-tracker-YYYY-MM-DD.csv`. Task names containing commas or quotes are properly escaped per the CSV spec.
- **Clear Log** — A "Clear" button resets all recorded activities after a confirmation prompt.

## Tech Stack

This is a **zero-dependency, single-file implementation**. Everything — HTML structure, CSS styling, and JavaScript logic — lives in `index.html`. There is no build step, no framework, and no external libraries.

- **HTML** — Semantic structure with a timer section, an activity log section, and a summary modal overlay.
- **CSS** — Dark theme UI with a `#0f1117` background. Uses CSS flexbox for layout, `border-radius` for rounded cards, and `font-variant-numeric: tabular-nums` so timer digits don't shift as values change. All styles are embedded in a `<style>` block.
- **JavaScript** — Vanilla JS using `setInterval` for the timer tick and `Date.now()` for elapsed-time calculation. Activity entries are stored in an in-memory array of `{ task, ms }` objects. CSV generation uses the Blob API and a temporary download link. All user-supplied text is escaped via a `textContent`/`innerHTML` roundtrip to prevent XSS.

## How to Use

1. Open `index.html` in any modern web browser.
2. Type a task name (e.g., "Chapter 3-4 video editing") into the input field.
3. Click **Start** or press **Enter** to begin the stopwatch.
4. Work on your task. The timer counts up in real time.
5. Click **Stop** when you're done. The session is saved to the activity log.
6. Repeat for each task throughout the day.
7. When you're finished, click **Summarize** to see an aggregated breakdown.
8. Click **Export CSV** to download the summary as a `.csv` file.

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

All activity data is held in a plain JavaScript array (`entries`). Each entry is an object with two properties:

- `task` (string) — the name the user typed
- `ms` (number) — elapsed time in milliseconds

This array is the single source of truth for both the activity log and the summary/export views.

### Aggregation

The summary groups entries by exact task name and sums their durations. This means "Bug fix" and "bug fix" are treated as separate tasks. The grouped data is displayed in the summary modal and used as-is for CSV export.

### CSV Format

The exported CSV follows RFC 4180 conventions:

```
Date,Task,Duration
2026-04-01,Chapter 3-4 video editing,01:23:45
2026-04-01,Email and admin,00:32:10
```

Task names containing commas or double quotes are wrapped in double quotes, with internal quotes doubled (`""`) per the spec.

### Security

User-provided task names are escaped before being inserted into the DOM. The `escapeHTML` helper creates a temporary `<div>`, sets its `textContent`, then reads back `innerHTML`, which neutralizes any embedded HTML or script tags.
