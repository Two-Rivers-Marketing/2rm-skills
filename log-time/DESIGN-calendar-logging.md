# log-time — Calendar-driven logging (design)

_Date: 2026-07-17 · Status: approved design, pre-implementation_

Extend `log-time` so a user can say **"log my time for yesterday"** and the skill reads their
Google Calendar, walks each meeting, and files a time entry per meeting — resolving job → task →
comment interactively, one event at a time.

## Prerequisite (org-level gap)

Requires a **Google Calendar read tool** in the *same environment where the skill runs* (claude.ai
web **and** Claude Code CLI — associates use both). As of this writing the 2RM Google connector
(Gmail) does **not** carry Calendar scope, so no calendar tool is exposed. Enabling that scope is an
org/management task, tracked outside this skill. The skill treats the tool as a hard dependency:

**Preflight (hard gate):** discover a Google Calendar list-events tool via ToolSearch. If absent →
**stop** with: _"Calendar logging needs Google Calendar (Calendar scope) enabled in this
environment. See setup."_ Nothing else runs. Degrades to a clear message, never a silent no-op.

## Config — `projects.yaml` `calendar:` block

```yaml
calendar:
  ignore:                       # regex, case-insensitive; never proposed
    - "daily dev stand[- ]?up"
    - "standup"
  rules:                        # explicit title-pattern -> job (task/service inherited from jobDefaults)
    - match: "1:1 w/ nik"
      job: 2RM-26000
  defaults:
    roundTo: 0.25               # snap duration to 15-min
    skipDeclined: true
    skipAllDay: true
```

Overlay section (user-owned) — preserved by `refresh`, like `jobDefaults`.

## Flow

1. **Trigger** — "log my time for \<day\>"; date parsed as elsewhere in the skill (default today).
2. **Preflight** — calendar tool present, else stop (above).
3. **Pull & filter** — day's events → drop `skipDeclined`, `skipAllDay`, and `calendar.ignore` matches.
4. **Per-event loop** (chronological, **file as each resolves** — not bulk):
   1. `hours` = duration snapped to `roundTo`.
   2. Rank job candidates: `calendar.rules` → `jobDefaults` aliases → fuzzy vs cached jobs.
   3. **AskUserQuestion** — the event (title, time, hours) + up to 4 ranked jobs (best labeled
      "Recommended") + the automatic **Other**.
      - **Other** → user describes it → fuzzy-search WMJ (`search_everything`/history) → resolve job.
   4. **Task** — `jobDefaults[job].task` → modal task from user history on that job → else quick
      task-name pick (match on `taskName`, never the float id).
   5. **Service** — `jobDefaults[job].service` → else `config.defaultService`.
   6. **Comment** — proposed = meeting title; user accepts or replaces.
   7. **Dup guard** — before filing, check `get_time_entries` for the day; if a same job+task entry
      exists, flag and confirm before adding.
   8. File the single entry (`add_time_entry`) → report `timeKey` → next event.
5. **End** — summary of filed + skipped/ignored rows.

### Matching order (per event)
`ignore` → `calendar.rules` → `jobDefaults` aliases → fuzzy vs cached jobs → else Other/fill-in.

## Self-heal

When an event is resolved via **Other → fuzzy**, offer to persist a `calendar.rule`
(title → job) so it's automatic next time — mirrors the `1:1` alias pattern. **Asks before
writing** to `projects.yaml`.

## Edge cases

- **Overlapping meetings** — file both; note the overlap in the end summary (user may drop one).
- **Partial attendance** — user overrides `hours` at step 4.1.
- **Already-logged day** — dup guard (4.7) catches re-runs.
- **Empty day / all ignored** — report "nothing to log" and stop.

## Out of scope (v1)

- Auto-filing without confirmation (always interactive).
- Non-Google calendars (Outlook/M365).
- Writing events back to the calendar.
- monday.com "assigned but unlogged" discovery (already deferred in REFERENCE).

## Open items

- **[blocking, org]** Google Calendar scope on the 2RM connector — management.
- Confirm the calendar tool's exact name/shape once the scope lands (drives the ToolSearch preflight
  and the event-field mapping: title, start/end, response status, all-day flag).
