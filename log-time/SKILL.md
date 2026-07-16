---
name: log-time
description: Log Workamajig (WMJ) time entries from natural language, auto-resolving job, task, and service from a local project registry plus the user's own WMJ history. Use when the user wants to log/track hours, "log N hours to this project", fill a timesheet, or mentions Workamajig/WMJ time entry. Directory-aware.
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

## Examples

- *(in a registered Workspace)* "log 5h, dev on the checkout" → candidates narrowed, task from history, service default, comment explicit → confirm job → file.
- "log 1h to MBD yesterday, silhouette icon UI" → fuzzy "MBD" → Job; task from history; date parsed → confirm → file.

## Not in v1

monday.com discovery ("what am I assigned but haven't logged yet?"), Workspace scaffolding, and a
per-Workspace `primary:` Job default. See [REFERENCE.md](REFERENCE.md) → Deferred to v2.
