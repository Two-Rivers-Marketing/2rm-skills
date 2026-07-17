# log-time — Reference

Field contracts, the Registry schema, and gotchas. **All examples are illustrative** — the skill
learns real values per-user from that user's own WMJ data; nothing here is hardcoded.

## Tools (Workamajig)

All are MCP tools; in some harnesses they're deferred — load with ToolSearch before calling.

- `add_time_entry` — write. Single form: top-level fields. Bulk: `entries: [{...}]`.
- `get_time_entries(startDate, endDate)` — the user's own entries. Seeds the cache; the
  history/default source for job, task, and service inference.
- `get_schedule_details(job)` — task list (`taskID` + `taskName`) for a Job.
- `get_project_summary(job)` — validate a Job exists.
- `search_everything(term)` — global WMJ search; used by the calendar "Other → describe it" path to
  resolve a Job the registry doesn't cache.

## Calendar tool (external — Google Calendar)

Calendar logging reads **Google Calendar**, not WMJ (`get_calendar_events` is a different calendar
and is not the source). The Google Calendar list-events tool is **not part of this skill's WMJ
tools** — it comes from a Google Calendar connector and is **discovered at runtime via ToolSearch**
(its exact tool id isn't known until the connector is enabled).

- **Hard preflight:** if no Google Calendar list-events tool is exposed in the current environment,
  stop and tell the user to enable Calendar scope (see SKILL.md → Calendar logging). Degrade to a
  message, never a silent no-op.
- **Org-scope gap (as of 2026-07-17):** the 2RM Google connector carries Gmail scope only, so no
  calendar tool is exposed yet. Enabling Calendar scope is an org/management task. Until then the
  preflight always stops — expected, not a bug.
- **Both environments:** associates use claude.ai **and** Claude Code; a connector enabled on
  claude.ai web is not automatically exposed to the Claude Code CLI. It must be enabled in whichever
  environment the skill runs in.
- **Fields consumed per event:** title, start, end (→ duration → hours), the user's response status
  (for `skipDeclined`), and an all-day flag (for `skipAllDay`). Map to the real tool's field names
  once known.

## add_time_entry contract

| Field | Required | Notes |
|---|---|---|
| `projectNumber` | yes | the Job number, e.g. `WG-26503` |
| `hours` | yes | number |
| `taskId` | no (API) / yes (real entries) | per-Job; every real entry has one |
| `serviceName` | no (API) / yes (real entries) | the user's service code (see defaultService) |
| `workDate` | no | `YYYY-MM-DD`; defaults today |
| `comments` | no | free text |

**Never send rate/amount** — WMJ auto-computes `actualRate`/`actualAmount`.

## Resolution facts / gotchas

- **`defaultService` is per-user** — the modal `serviceCode` in that user's `get_time_entries`
  history. Never hardcode a service code.
- **`taskId` is per-Job and per-user** — resolve from the user's own history first; the schedule is
  the fallback for a never-logged Job. It is not constant across Jobs.
- **taskID float collision** — `get_schedule_details` renders IDs as floats, so `1.10` shows as
  `1.1` (collides with a real `1.1`). Always match by `taskName`.
- **Non-loggable tasks** — some tasks are flagged like "…(NO TIME ENTRY)". Never log to these.
- **Job number encodes the client prefix** (e.g. `WG-…`), but keep the client grouping explicit in
  the cache for fuzzy match.

## Registry schema — `projects.yaml`

Lives beside this skill, **gitignored, per-user**. `clients` is the WMJ cache;
`workspaces`/`jobDefaults`/`config` are the local Overlay, never overwritten by refresh.

```yaml
config:
  wmjUser: <name / email / id>     # from current user on bootstrap
  defaultService: <code>           # modal serviceCode in the user's history
  last_refresh: YYYY-MM-DD
clients:                           # WMJ cache — refresh replaces this whole section
  - name: <client>
    jobs:
      - job: <JOB-NUMBER>
        name: <project name>
        active: true               # heuristic: logged-to in the last 60 days
workspaces:                        # overlay — directory -> candidate Jobs
  - path: ~/path/to/dir
    jobs: [<JOB-NUMBER>, <JOB-NUMBER>]
    defaults: { service: <code> }
jobDefaults:                       # overlay — per-Job task / service / aliases
  <JOB-NUMBER>: { task: "<id>", service: <code>, aliases: [<alias>] }
calendar:                          # overlay — calendar-driven logging (see SKILL.md → Calendar logging)
  ignore:                          # case-insensitive title regexes; never proposed
    - "<regex>"
  rules:                           # explicit title -> Job (task/service inherited from jobDefaults[job])
    - { match: "<regex>", job: <JOB-NUMBER> }
  defaults:
    roundTo: 0.25                  # snap event duration to this fraction of an hour
    skipDeclined: true             # drop events the user declined
    skipAllDay: true               # drop all-day events
```

**Calendar matching order (per event):** `calendar.ignore` → `calendar.rules` → `jobDefaults`
aliases → fuzzy vs cached Jobs → else the AskUserQuestion "Other" fill-in. Resolving via "Other"
offers to persist a new `calendar.rules` entry (self-heal), same as adding an alias.

**Merge on refresh:** key by job number; overwrite the `clients` section (WMJ-owned), preserve
`workspaces` / `jobDefaults` / `config` (user-owned).

## Deferred to v2

- **monday.com discovery** — the forward-looking "what am I assigned but haven't logged yet?" view.
  WMJ history is backward-looking and can't surface a brand-new assignment. If added: the monday Job
  code lives in the parent item's **group title** (not the Job No. column, which is empty on some
  boards); work lives in **subitems**, not parent items; board fetches are large (process
  out-of-context, never inline); monday user id ≠ WMJ user id.
- **Workspace scaffolding** — a separate skill that reads this same `projects.yaml` to create a new
  workspace directory.
- **`primary:` Job per Workspace** — a default pick for when narrowing candidates proves tedious.
