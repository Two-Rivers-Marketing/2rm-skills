---
name: log-time
description: Log Workamajig (WMJ) time entries from natural language, auto-resolving job, task, and service from a local project registry plus the user's own WMJ history. Use when the user wants to log/track hours, "log N hours to this project", fill a timesheet, or mentions Workamajig/WMJ time entry. Directory-aware.
allowed-tools:
  - Read
  - Write
  - Edit
  - ToolSearch
  - AskUserQuestion
  - mcp__claude_ai_WMJ__get_time_entries
  - mcp__claude_ai_WMJ__add_time_entry
  - mcp__claude_ai_WMJ__get_schedule_details
  - mcp__claude_ai_WMJ__get_project_summary
  - mcp__claude_ai_WMJ__search_everything
  # Calendar logging reads GOOGLE Calendar, not WMJ. Its list-events tool is
  # discovered at runtime via ToolSearch (name unknown until the org enables
  # Calendar scope). Add its exact tool id here once known. Do NOT use WMJ's
  # get_calendar_events — different calendar, not the source.
---

# log-time

Turn "log 2h to western global, fixed the capacity page" into a valid Workamajig time entry.
Runs entirely on **WMJ** plus a local **Registry**. Terms: [CONTEXT.md](CONTEXT.md). File schema
and field contracts: [REFERENCE.md](REFERENCE.md).

**Principle: the Registry accelerates, it never gates.** A stale or empty cache degrades to a
live WMJ lookup — logging never fails because the cache is behind.

## First run (bootstrap)

If there's no `projects.yaml` beside this skill:
1. Identify the WMJ user (`get_time_entries` defaults to the current user).
2. Seed `clients` from the last **4 months** of `get_time_entries` — the distinct Client + Job set.
3. Infer `config.defaultService` = the modal `serviceCode` in that history.
4. Write `projects.yaml` (schema in REFERENCE). `workspaces`/`jobDefaults` start empty — filled as the user works.

## Resolution chain

Fill every field from a source; confirm only **inferred** ones.

1. **job** —
   - *Directory:* match `cwd` (walk up; longest `workspaces[].path` wins) → candidate Jobs → drop
     inactive → fuzzy-match the comment against candidates' names + task history → propose the clear
     winner, else ask (list candidates with each one's last-logged date).
   - *Phrase:* fuzzy-match the user's words + `aliases` against all cached Jobs. Ambiguous → ask.
   - *Miss* (not cached): live WMJ lookup → confirm → log → **append the Job to `clients`** (self-heal).
2. **taskId** — `jobDefaults[job].task` → else the last `taskId` used on that Job (history) → else a
   `get_schedule_details(job)` task named like "…Technology Solutions"/"Dev" → else ask. Match on
   `taskName`, never the float-rendered number.
3. **serviceName** — `jobDefaults[job].service` → else `config.defaultService`. Inferred → confirm.
4. **hours / comments** — from the user; ask for the comment if absent (only they know it).
5. **workDate** — parse "yesterday"/"today"/explicit date → `YYYY-MM-DD`. Default today.

Several activities in one ask → build multiple entries, file in one bulk `add_time_entry`.

## Confirm policy

Show the resolved table. If any field was **inferred** (job by fuzzy match, task, or service),
require a "yes". If the job resolved unambiguously (single active candidate or exact alias) and
hours+comment were explicit, file without asking. Always report the returned `timeKey`.

```
| job | date | hours | task | service | comment |
```

## Register / refresh

- **Register a Workspace:** add a `workspaces` entry (`path` + candidate `jobs`) so the directory
  resolves next time.
- **Refresh** (`log-time refresh`; nudge if `config.last_refresh` is >30 days old): re-pull the
  4-month window, add new Jobs, recompute `active`. Overwrites `clients` only — Overlay untouched.

## Calendar logging — "log my time for \<day\>"

Read the user's Google Calendar for a day, walk each meeting, and file one entry per meeting.
Config schema (`calendar:` block) and the tool contract are in [REFERENCE.md](REFERENCE.md).

**Preflight (hard gate).** Discover a Google Calendar list-events tool via `ToolSearch`. If none is
exposed in this environment, **stop** — do not fall back to WMJ's calendar (it's a different
calendar). Say exactly:

> Calendar logging needs the Google Calendar connector with **Calendar scope** enabled in this
> environment (claude.ai and/or Claude Code). Enable it, then rerun. (Gmail scope alone is not
> enough.)

**Pull & filter.** Fetch the day's events → drop, in order: events matching a `calendar.ignore`
regex (case-insensitive), then `calendar.defaults.skipDeclined` (declined by the user) and
`skipAllDay` (all-day events).

**Per-event loop** — chronological, **file each entry as it resolves** (not bulk):

1. `hours` = event duration snapped to `calendar.defaults.roundTo` (e.g. 0.25).
2. Rank job candidates: `calendar.rules` (title→job) → `jobDefaults` aliases → fuzzy vs cached Jobs.
3. **AskUserQuestion** — show the event (title, time, `hours`) and offer up to 4 ranked Jobs (best
   first, labeled "Recommended"); the tool's built-in **Other** lets the user describe it instead.
   - **Other** → user describes the work → fuzzy-search WMJ (`search_everything` + history) →
     resolve the Job (confirm if not obvious).
4. **taskId** — resolve per the standard chain (§Resolution chain step 2): `jobDefaults[job].task` →
   modal `taskId` from the user's history on that Job → else ask. Match on `taskName`.
5. **serviceName** — `jobDefaults[job].service` → else `config.defaultService`.
6. **comment** — proposed = the meeting title; the user accepts or replaces it.
7. **Dup guard** — before filing, scan `get_time_entries` for that day; if a same job+task entry
   already exists, surface it and confirm before adding (prevents double-logging on a rerun).
8. File the single entry (`add_time_entry`) → report the `timeKey` → move to the next event.

End with a summary of what was filed, skipped (declined/all-day), and ignored.

**Edge cases.** Empty day or everything ignored → say "nothing to log" and stop. Partial
attendance → let the user override `hours` at step 1. Overlapping meetings → file both; note the
overlap in the summary so the user can drop one.

**Self-heal.** When an event resolves via **Other → fuzzy**, offer to append a `calendar.rules`
entry (`match:` title → `job:`) so it's automatic next time — same pattern as adding an alias.
Ask before writing to `projects.yaml`.

## Examples

- *(in a registered Workspace)* "log 5h, dev on the checkout" → candidates narrowed, task from history, service default, comment explicit → confirm job → file.
- "log 1h to MBD yesterday, silhouette icon UI" → fuzzy "MBD" → Job; task from history; date parsed → confirm → file.
- "log my time for yesterday" → calendar preflight → filter (drop the ignored dev standup) → per meeting: pick Job (or Other→fuzzy), task, comment → file each → summary.

## Not in v1

monday.com discovery ("what am I assigned but haven't logged yet?"), Workspace scaffolding, and a
per-Workspace `primary:` Job default. See [REFERENCE.md](REFERENCE.md) → Deferred to v2.
