# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file IT PMO Kanban board (`index.html`) for a fictitious bank, built as an internal demo/training tool. All markup, CSS (`<style>`) and JS (`<script>`) live in that one file.

## Running

There is no build, lint or test tooling. Open `index.html` directly in a browser (double-click, or `Start-Process index.html` from PowerShell). No server is required.

## Hard constraints (from the original spec — keep them)

- **Stack:** vanilla HTML/CSS/JS only. No frameworks, libraries, bundlers, npm, CDNs, web fonts or image files. Use a system font stack, and Unicode glyphs or inline SVG for icons.
- **Single file:** everything stays in `index.html`.
- **No persistence:** do not use localStorage, sessionStorage, IndexedDB or cookies. A page refresh is meant to reset the board to the seed data, and the header note tells users so.
- **Only network call:** `notifyNewTask()` POSTs JSON to FormSubmit's AJAX endpoint. Nothing else leaves the page.
- **Branding:** neutral "IT PMO" text wordmark with a corporate blue palette. Do not use any real bank logo or trademark, and do not imitate an official system. The `UOB-ITPM-####` task-ID format and `[UOB IT PMO]` email subject were specified by the user and are kept as text-only config constants.
- **No `alert()` / `confirm()`:** use inline errors, the inline "Delete? Yes / No" row, and toasts instead.
- **CSS:** no `!important`. Palette and spacing come from the custom properties in `:root`.

## Architecture (inside `<script>`)

- **Config at the top:**
  - `FORMSUBMIT_ENDPOINT` is the one place to swap the notification email.
  - `ID_PREFIX`, `SUBJECT_PREFIX` and `NOTIFY_TIMEOUT_MS` sit next to it.
  - The HTML comment just above `<script>` documents FormSubmit's one-time activation email.
- **Single source of truth:** `state = { tasks, filters, ui }`.
  - `ui.moveMenuId` and `ui.pendingDeleteId` hold transient card UI (the open "Move ▸" menu and the delete confirmation). Keeping them in state lets `renderBoard()` stay the only code that writes card markup.
  - `idCounter` drives `nextId()`, which zero-pads IDs to 4 digits.
- **Render path:**
  - `renderBoard()` runs `applyFilters(state.tasks)` (pure), fills each column's `[data-list]` with `renderCard()` HTML strings, updates the `[data-count]` badges, then calls `renderSummary()`.
  - Column counts reflect the filtered tasks. The header summary reflects all tasks.
  - Every user-supplied string must go through `escapeHtml()` before it is inserted into HTML.
- **Mutations:** `addTask`, `moveTask` and `deleteTask` change `state` and then call `renderBoard()`. Never patch card DOM directly.
- **Events are delegated once on `#board`** (click, plus the dragstart/dragover/dragleave/drop/dragend handlers), so re-renders never re-bind listeners.
  - Card buttons are dispatched by their `data-action` attribute in `handleBoardClick`.
  - After a re-render, focus is restored with `focusCardControl()` or `focusColumnHeading()`, which keeps keyboard use working.
- **Columns are static HTML:** the four `<section class="column" data-status="...">` elements are written in the markup. Their `data-status` values must match the `STATUSES` array.
- **Dates:**
  - Dates are ISO `YYYY-MM-DD` strings compared as strings.
  - `todayISO()` uses the local date, not UTC.
  - A task is overdue when `dueDate < today` and its status is not Done.
  - Seed data uses `daysFromToday(n)`, so the demo always has overdue cards.
- **Add Task submit (`handleSubmit`):**
  1. `validateForm()` checks the input; `showFieldErrors()` writes the inline errors.
  2. The task is added optimistically.
  3. `notifyNewTask()` runs in try/catch with an AbortController timeout. On failure the card stays and a warning toast shows. A FormSubmit failure must never break the board.
  - `notifyNewTask()` also treats an HTTP 200 reply with `success: "false"` as a failure; FormSubmit sends that before the address is activated.
  - The modal stays open showing "Sending…" and closes itself afterwards, unless the user has started typing another task (`formDirtySinceSubmit`).

## Known caveat

FormSubmit may reject requests from a page opened as a local file (`file://`, Origin `null`). Notifications are most reliable when the page is served from a real origin; the failure path already handles this with the warning toast.
