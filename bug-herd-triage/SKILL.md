---
name: bug-herd-triage
description: Work a BugHerd project's open feedback end to end — triage, fix in batches, ship to staging, verify each ticket with before/after screenshots, move verified tickets to Review.
disable-model-invocation: true
---

# BugHerd triage

The full loop for a BugHerd board: every open ticket ends as *verified in Review*, *blocked* with a named missing input, or *waiting* on a named decision. The repo's own agent docs (AGENTS.md / CLAUDE.md, wiki) own the project facts — staging URL, local URL, deploy path, DB push — read them; this skill owns the loop.

BugHerd tools are deferred: load them with ToolSearch before first use.

## 1. Orient

Run the repo's orientation protocol. Find the BugHerd project with `list_projects` (match by name or site URL) and note its status columns from `get_project_details`.

Done when you can name: project id, staging URL, local URL, how code ships, how content ships.

## 2. Build the ledger

`list_project_tasks` for every non-done status (compact, `description`). Then dispatch one subagent to run `get_task_details` and fetch the full **comment thread** (ToolSearch `+BugHerd comment` for the list tool) on every open ticket, and write a markdown **ledger** to the scratchpad: id, page path, element (selector tail + ≤120-char html summary), window size, full description, comment thread (author, date, full text, oldest first), attachments, screenshot URL. Raw task JSON stays out of the lead's context.

Done when the ledger has one row per open ticket and every row carries its comment thread, or "no comments" after the fetch came back empty.

## 3. Triage

Group the ledger by page × issue kind. A ticket's ask is its description as revised by its comments: a later comment that clarifies, narrows, answers a question, or withdraws the request is the current ask. Comments signed "(via Claude)" record earlier runs; read them as history of what was already changed and verified.

For each ticket decide:

- **ready** — the ticket plus the repo/wiki/asset library hold everything needed. Look before calling anything missing: search the wiki for named facts (cert numbers, claims), the asset library and repo for named files (logos, badges).
- **decision** — more than one reasonable reading ("match icon style" — which style?), or the request conflicts with a recorded fact (cite the wiki page).
- **blocked** — needs an input nobody here has (a file "from Jeff", client copy).

Present the grouping with proposed batches, then ask every decision in one `AskUserQuestion` call. Record the answers; they are the session's rules.

Done when every ticket is ready, decided, or blocked with its missing input named.

## 4. Fix in batches

Batch by what changes together (one page's copy, one asset family, one CSS pattern). Per batch:

- Content through the CMS's API/MCP; code through templates and styles. Follow the repo's conventions for both.
- When a ticket exposes a pattern (one icon box wrong → every icon box), fix the pattern and sweep for siblings of the same defect.
- Build, then exercise each fix in the browser at the ticket's window size and at 390px. Screenshot and look.
- When the user asks for provenance (which assets are client-supplied vs made by us), write it into the wiki per its filing rules.

Done when every ready/decided ticket in the batch is fixed locally and seen in the browser.

## 5. Ship

Order matters: **staging still shows the before** until content ships, and code deploys can overwrite befores too.

1. Commit only files this session changed; report anything else in `git status` rather than touching it. Push per the repo's flow and watch the deploy run to green.
2. Hold the content push until verification has captured its befores (step 6).
3. Content push: first check staging for edits since the last import; carry them into local or confirm with the user that losing them is fine. Then run the repo's push path (it must back up the target first).

Done when the code deploy is green and the content push is either done or explicitly queued behind step 6.

## 6. Verify

Before = staging, after = local dev (the user's default; ask if they want staging-after-push instead).

1. **Prove the recipe** on one ticket yourself: screenshot, upload, attach, comment — following [VERIFY-BRIEF.md](VERIFY-BRIEF.md).
2. Write the brief for this run: copy VERIFY-BRIEF.md to the scratchpad and fill in the environment URLs and what already shipped.
3. Dispatch page-grouped agents (~8–10; group tickets sharing a page). Each prompt: the brief path, a group name, and per ticket the exact change made and the expected after (exact strings, sizes, counts). Run them in the background.
4. **Triage every report.** Open the images behind any weak before. A before is weak when it is an unstyled pin capture, shows the wrong section, or is staging already in its fixed state — replace it with a **reconstructed** before (see the brief) and add a follow-up comment correcting the record. A DISCREPANCY is either a real defect (fix → re-verify) or a wrong expectation in your prompt (correct the comment).

Done when every fixed ticket has a before that shows the problem, an after that shows the fix, and a comment that matches both.

## 7. Move to Review

`update_tasks` with the project's review status (≤50 ids per call) for every verified ticket. Review, never done: done is the reviewer's call. If BugHerd syncs to GitHub, any later done-move goes BugHerd first, GitHub second.

## 8. Report

Re-list open tasks (new tickets often land mid-run). Report: moved to Review, blocked (with the missing input), decisions waiting on the client (AI-written copy, claims that conflict with the wiki), new tickets and which are ready, unexpected repo/environment state.
