# Attendance error discovery & full error-type coverage (design)

**Date:** 2026-07-21
**Status:** Approved for planning

## Context

The user asked whether `shaon` can be used to clear the "משימות" (tasks) /
"שגויים לטיפול" (errors to handle) backlog shown on their real Hilan
homepage, so their attendance reporting stays accurate and their salary
isn't affected by unresolved errors. Their real homepage showed **15
pending errors across 4 distinct error types** — but `shaon attendance
errors --month 2026-07` reported **zero**, against the same real account,
in the same session.

Investigating live (via a CDP-attached browser already logged into the real
account, plus direct dry-run/execute calls against `shaon`'s own CLI)
established the actual cause and the current state of support per error
type:

| Error type (Hebrew) | `errorType` | Count on homepage | Status |
|---|---|---|---|
| חסר דיווח ליום עם תקן (missing report) | 63 | 2 | Already documented and tested (`PROTOCOL.md`, existing `fix_error_day` tests) |
| חסרה יציאה (missing exit) | 2 | 4 | Same generic form-replay path as 63; not yet execute-tested but structurally identical |
| חסרה כניסה (missing entry) | 1 | 1 | Same as above |
| יש לדווח פרוייקט אחד לפחות ליום זה (project reporting) | 131 | **8** | Dry-run payload builds successfully, but a live `--execute` test was rejected by Hilan: `"יש לבחור סוג דיווח אחר"` ("you must choose a different report type") — genuinely needs different handling |

All four route through the same `EmployeeErrorHandling.aspx?date=...&
reportId=...&errorType=N` endpoint `shaon` already automates via
`fix_error_day` (`attendance.rs:1596`) — confirmed by inspecting the
Angular-era Hilan homepage's task-resolution modals, which render this
exact legacy ASPX page inside an iframe rather than a native Angular form.

## Root cause of the zero-vs-fifteen discrepancy

`build_overview` (`hr-core/src/use_cases.rs:117-134`) builds its
`error_days` list by filtering `calendar.days` for `day.has_error == true`
**first**, then attaches any matching `fix_targets` (discovered via
`get_error_tasks()` → `HHomeTasksApiapi/GetData`, `api.rs:273`) only to
days that already passed that filter. `read_calendar`'s calendar-grid
error-marker scraping was only ever built and tested against the
`errorType=63` case, so it never sets `has_error = true` for the other
three types — their correctly-discovered `fix_targets` get computed and
then silently discarded before reaching `attendance errors`, `overview`,
or the MCP tools built on top of them.

Critically, this bug **does not affect `attendance resolve`** — that
command calls `provider.fix_targets(month)` directly
(`shaon-cli/src/lib.rs:799`), bypassing the broken merge entirely. This is
why the dry-run/execute tests against real `errorType=131`/`1`/`2` data
worked (in the sense of producing correct payloads / real server
responses) even while `attendance errors` reported nothing.

## Fix 1: union fix_targets into error_days (low risk, no open questions)

In `build_overview`, change `error_days` from "calendar-flagged days that
happen to also have a fix_target" to "the union of calendar-flagged days
and any date with a discovered fix_target." Concretely: iterate
`fix_targets_by_date` in addition to `calendar.days.filter(has_error)`,
constructing an `OverviewErrorDay` for any date present in either set,
sourcing `day: CalendarDay` from the calendar when available and
synthesizing a minimal placeholder day (matching the `FixTarget`'s own
`date`) when the calendar didn't flag it.

This alone makes `attendance errors`/`overview` and the MCP `errors`/
`overview` tools correctly report all four error types with no risk to
correctness elsewhere — `attendance resolve` already proved live that the
underlying discovery and fix payload for types `1`, `2`, and `131` are
either already correct (`1`, `2`) or produce a real, informative server
error (`131`) rather than corrupting anything.

**Testing:** pure/unit-testable, same shape as the `TASK-181`
(`apply_selected_day_enrichment`) fix reviewed earlier this session — feed
synthetic `CalendarDay`/`FixTarget` fixtures through the union logic and
assert both calendar-flagged-only and fix-target-only dates appear in the
result.

## Fix 2: errorType=131 (project reporting) — needs real wire capture first

This is the majority error type (8 of 15 on the real account) but the one
genuinely uncertain piece. The live `--execute` test proved that resending
the wizard's pre-filled defaults (entry/exit unchanged, `Symbol.SymbolId =
0` i.e. "work day") is rejected server-side with `"יש לבחור סוג דיווח
אחר"`. This means either:
- a different attendance-type/symbol code needs to be selected for this
  error class, and/or
- the project-selection autocomplete field (`Comment` /
  `DummyAutoCompleteText`, currently submitted unchanged as the Hebrew
  placeholder "הקש ערך לחיפוש" — "type to search") must actually be
  populated with a real project value, which likely requires driving an
  autocomplete search-and-select interaction (its own small AJAX exchange)
  before the final save postback.

Per this project's established discipline (see the MFA wire-capture task
and the SSRS-report investigation earlier this session): **do not guess at
this — capture the real browser interaction first.** Use the CDP-attached
browser to manually resolve one real `errorType=131` item through Hilan's
own UI while recording the network requests (the autocomplete search
call, the value it sends on selection, and the final save payload), then
extend `AttendanceSubmit`/`replay_submit_with_fields` to reproduce it.

## What's usable today, unchanged

`errorType=1` (missing entry) and `errorType=2` (missing exit) — 5 of the
15 real items — can already be resolved via
`shaon attendance resolve <date> --type <code> --execute` today, with no
code changes, once the entry/exit time to submit has been confirmed
correct for that day. Fix 1 is what makes them (and the `errorType=131`
items, once Fix 2 lands) visible through `attendance errors`/`overview`
rather than requiring the date to already be known.

## Verification plan

1. Fix 1: unit tests against synthetic fixtures (no live account needed).
2. Fix 1, live: re-run `attendance errors --month 2026-07` against the
   real account and confirm all 15 items now appear with correct
   `report_id`/`error_type`/date.
3. Fix 2: capture real wire data for one `errorType=131` resolution via
   the CDP-attached browser (read-only observation of the user completing
   it manually), implement, then dry-run and — only with explicit
   confirmation, same as this session's live test — `--execute` against
   one real item to confirm Hilan accepts it.

## Explicitly out of scope

- Any change to the classic calendar's own `has_error` marker detection —
  Fix 1 works around its incompleteness rather than fixing it, since the
  fix_targets union is authoritative and cheaper than teaching the visual
  parser to recognize three more marker patterns it was never designed for.
- Bulk/automatic resolution of all 15 items in one command — this design
  only covers making them correctly discoverable and resolvable one at a
  time via the existing `attendance resolve` command; a bulk-fix command
  is a separate, later decision once single-item resolution is proven
  correct for all four types.
- The SSRS report-based calendar-reading redesign discussed earlier in
  this session (`CalculateAttendanceData918NEW` / `AttendanceStatusReportNew2`)
  — unrelated to this error-wizard work, tracked separately.
