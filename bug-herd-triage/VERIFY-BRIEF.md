# BugHerd verification brief

Template for verification agents. The lead copies it to the scratchpad and fills every `<…>`.

---

You verify completed BugHerd tickets. You capture screenshots, attach them, comment, and report. You leave site code, CMS content and ticket status exactly as they are.

## Environments

- **Before** = staging: `<staging URL>`
- **After** = local: `<local URL>` (fixed, not yet on staging)
- Already shipped to staging: `<commit + what it changed>`. Where staging already shows the fix, build a **reconstructed** before (below).

## BugHerd

Project `<id>`. Load with ToolSearch: `select:mcp__BugHerd__get_task_details,mcp__BugHerd__prepare_attachment_upload,mcp__BugHerd__update_task_attachments,mcp__BugHerd__add_comment`. Read each ticket and its full comment thread first (ToolSearch `+BugHerd comment` for the list tool); its `url`, `css_selector_info` and `browser_info.window_size` give page, element and viewport.

## Screenshots

- Work in `<scratchpad>/verify/<group>/`. Use a named browser session: `playwright-cli -s=<group> …` on every command; close it at the end.
- Resize to the ticket's window size, `scrollIntoView` the target, wait ~800ms for lazy images, screenshot the containing section (not the full page).
- Read every PNG you will attach and confirm it shows what the comment claims.
- One screenshot may serve several tickets on the same section; attach it to each.

## Reconstructed befores

The BugHerd pin screenshot (`screenshot.url`) is usually an unstyled capture and sometimes the wrong section — attach it as a record, never as the only before. When staging already carries the fix, reconstruct the pre-fix render on staging:

- **CSS change**: inject the old rules (the lead supplies them) via `eval` creating a `<style>` with `!important` overrides.
- **Markup change**: rewrite the element's `innerHTML` / classes back to the old markup.
- **Added element**: hide it with an injected `display:none` rule.

Measure to confirm it took (e.g. the old box size), name the file `<id>-before-reconstructed.png`, and label it in the comment as "staging with the pre-fix CSS/markup re-injected".

## Upload recipe

1. `prepare_attachment_upload` {project_id, task_id, files:[{file_name}]} → `put_url`, `public_url` (valid 15 min).
2. `curl -s -o /dev/null -w "%{http_code}\n" -X PUT -H "Content-Type:" --upload-file <file> "<put_url>"` → 200. Send only the blank Content-Type: the ACL is signed into the URL, and an `x-amz-acl` header makes S3 return 403.
3. `update_task_attachments` {action:"add", attachments:[{file_name, url: public_url}]}.
4. `add_comment`: `Verification: <page + section, viewport>. Before (<staging | reconstructed | BugHerd pin>): <what it shows>. After (<local, pending staging push | on staging>): <what it shows>. Change: <one line>. Screenshots attached: <names>. (via Claude)`

## Verify against the ticket

The after must satisfy the ticket's words as revised by its comments (a later clarification is the current ask), not merely differ from the before: exact strings for Replace/With tickets, measured sizes for sizing tickets, counts for lists. Check the same section at 390px for overflow or overlap. A ticket that falls short gets no pass comment; report it.

## Report (≤40 lines)

`#id PASS | DISCREPANCY | BLOCKED — one line` per ticket. For each DISCREPANCY: what the ticket asked, what you see, screenshot path. Name any upload or comment that failed, and any before you judged weak.
