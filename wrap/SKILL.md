---
name: wrap
description: Wrap the session — digest to the project's wiki, update state, log a breadcrumb. Use when the user types /wrap, or is clearly ending a work session.
---

# Wrap

## Steps

### 1. Resolve where you are

- Repo = current working directory's git root. Slug = repo folder name.
- Wiki = the project wiki if one exists. Resolve it the way the `wiki` skill does: check `docs/wiki/WIKI-CLAUDE.md`, then `wiki/WIKI-CLAUDE.md`, then root `WIKI-CLAUDE.md`. No wiki → skip step 4, STATE.md only.

### 2. Review (conditional — code sessions only)

Run this BEFORE the digest so findings appear in the digest's Next steps.

Check whether the session produced code changes: run `git diff --name-only "$BASE_SHA"...HEAD` (where `BASE_SHA` is passed by the caller; default to `HEAD~5` for interactive sessions) and look for non-doc, non-planning source files (`.ts`, `.tsx`, `.js`, `.py`, `.rb`, `.go`, `.css`, `.html`, `.svelte`, `.vue`, `.swift`, `.rs`, etc. — not `.md`, not `.planning/`).

**If code changes exist** and a code-review skill is available, run it and fold its findings into the digest's Next steps.

**If no code changes** (research, decisions, planning): skip silently.

### 3. Distill the session

Build a digest from the conversation so far. Include any code review findings from step 2 in the **Next steps** section. Sections, all required (write "none" rather than omitting):

- **Accomplishments** — what shipped/changed, with file paths
- **Decisions** — calls made and why, one line each; flag any that belong in a decisions log
- **Struggles** — where we churned, corrected, or burned time. Include tooling friction, not just code problems.
- **Next steps** — checkbox list, concrete enough to resume cold. Include code review findings if any.
- **Open questions** — anything needing the user's call

Substance gate: if the session was trivial (a question or two, no work product), say so and stop — don't log noise.

### 3b. Research capture (conditional)

If the session was research-shaped — an external video/article/transcript discussed, a competitor or concept compared to the user's approach, strategic thinking-out-loud — capture it into the wiki's research area: drop a `research-YYYY-MM-DD-<gist>.md` under the wiki's `raw/research/` for later ingest promotion (per the `wiki` skill).

If the session had no research flavor, skip silently.

### 4. Write to the wiki

Invoke `/wiki log` with the digest content.

### 5. Update state

Write/update `{repo}/STATE.md`: bump the Updated date and Status, then Accomplishments / Current Status / Next Steps matching the existing format. Thin — state is a springboard, not a transcript. Use a clear status (e.g. `Active` / `Blocked` / `Waiting` / `Paused` / `Done`) and absolute dates.

### 6. Breadcrumb

Append one line to `~/.claude/session-wrap.log`:
`<ISO date> | <slug> | digest: <path or "skipped"> | state: <path>`

### 7. Close

Report: files written, decisions that should be logged (suggest the wiki's decisions log), and whether a git commit makes sense. Suggest the commit; don't push without being asked.

## Rules

- Write directly to the current repo's wiki via `/wiki log`. Never write outside the current repo. Cross-project context stays in the digest.
- Honest struggles. A digest that says "everything went great" when the session churned for an hour poisons any later review.
- Don't duplicate: if the session already wiki-logged its work (e.g. a `/wiki log` ran), note that in the breadcrumb and only fill gaps.
