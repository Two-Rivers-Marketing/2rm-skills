---
type: index
timestamp: {{DATE}}
description: Ledger of known unknowns that block or degrade knowledge — not todos, not settled decisions.
tags: [open-questions, conventions]
---

# Open Questions

Known unknowns that block or degrade knowledge — not todos, not settled decisions, not pending lint
corrections (those live in the project's task/todo list and decisions log instead).

## Entry format

```markdown
- **<question, phrased so a routed answer closes it>**
  - Added: YYYY-MM-DD
  - Source: manual | wrap | lint | dispatch | skill:<name>
  - Client: <client-slug>/<project-slug>          _(optional)_
  - Context: <one-line — why this matters>        _(optional)_
```

New entries are appended at the bottom of **Open**. No `weight:` field — rank by entry age, oldest
`Added` date first.

## Feeders

- **Mid-session gaps** — any skill that notices a known unknown mid-session appends it here silently,
  with a one-line receipt to the operator ("parked: <question>").
- **`/wiki lint` tier-3 escalations** — unadjudicable contradictions land here, deduped against
  entries already open.
- **`/wrap`'s parking step** — questions the session raised and left unanswered.
- **Cross-repo bubbling** *(optional adapter — only where an operator runs a hub repo)* —
  business-level questions, not implementation detail, also drop a copy into the hub's own ledger
  via the session-wrap notify step.

This file is created on demand, not scaffolded: it exists because a real question needed
recording. If `## Open` empties out, delete the file — git history is the ledger.

## Closing — by routing, not archiving

An entry closes when its answer lands on a real surface — a wiki page, the decisions log, the todo
list — and the entry is deleted in the same commit as that answer. Deciding a question is moot closes
it the same way: delete the entry, say why in the commit message. **No "Closed" or "Archive" section,
ever** — git history is the ledger.

## Open

_(none yet)_
