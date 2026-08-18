# CLAUDE-MEMORY-BLOCK — the standard "Project memory" section

Canonical wiki read/write contract for a repo's root agent-instruction file. `/wiki init` and
`/wiki convert` instantiate this. **Rules are identical in every repo; only paths vary.** Do not
reword the rules per project — that drift is what this file exists to prevent.

## Why it is shaped this way

Drawn from a documented retrieval failure: an agent read a project's state file, took its handful of
passing mentions of a sibling knowledge store as the complete map, and spent a session rediscovering
findings recorded three weeks earlier — including a conclusion that was recorded and *wrong*.

Four properties the block enforces, each traceable to a cause:

1. **The trigger is a path, not a topic.** Topic-scoped triggers ("sessions about content, decisions,
   or strategy") never fire on a debugging session. A judgment call demanded at the moment of least
   context is not a trigger.
2. **At least one orientation step is generated, not authored.** Authored catalogs drift — in the
   incident, an index entry pointed at the wrong document for three days. `ls`/`find` cannot drift.
3. **A summary is never the entry point.** Summaries are terminal: they answer "where does this
   stand" and kill the pull toward anything else. Catalogs are non-terminal. The state file goes last
   and is explicitly labeled partial.
4. **Memory is never authority.** The proximate cause was subagents briefed from the orchestrator's
   recall rather than a lookup. Recall is lossy, and delegation multiplies the loss.

The one instruction that *did* fire in that session named a specific file, tied itself to an
unavoidable action, and cited a real failure. Every rule below has all three properties.

## Instantiation

Substitute the repo's actual paths. Drop what doesn't exist — never emit a pointer to a missing file.

| Condition | Action |
|---|---|
| No sibling store (`docs/solutions/` or equivalent) | Drop the two-trees sentence, drop the sibling `find` from step 1, drop the Area read section. **Delegation is not part of that drop** — it lives under Authority and ships in every repo |
| Wiki uses a `log/` directory, not `log.md` | Step 3 becomes `ls docs/wiki/log/ \| sort -r \| head -10`, then Read the newest 2-3 entries |
| No log at all | Drop step 3, renumber |
| No state file — check the **repo root**, not just the wiki dir | Drop step 4, renumber, and drop the state-file sentence from Authority. A state file that lives outside the wiki still counts: keep step 4, point it at the real path |
| No `decisions/` | Drop it from the step-1 enumeration |
| Nested wiki layout (e.g. `pages/`) | Step 1 becomes `find docs/wiki -name '*.md' -not -path '*/raw/*'` |
| No `schema.md` yet | Drop it from the Write section; scaffold one from `schema-TEMPLATE.md` instead |
| An area has a README entry point | Add a concrete Area read table row. No README anywhere → keep the section, drop the table, keep the enumeration-match rule |

Replace whatever section the repo currently uses for this (`## Read this first`, `## Knowledge wiki`,
`## Project Memory`). Leave repo-specific operating rules alone.

---

## Project memory

Project memory is two trees. `docs/wiki/` holds what is true about this project. `docs/solutions/` holds engineering learnings: debugging patterns, gotchas, how a subsystem actually behaves. "Consult the wiki" does not cover the second tree.

### Orientation read — every session, no exceptions

Before your first substantive action:

1. **Enumerate.** Run this before reading anything:
   `ls docs/wiki/*.md docs/wiki/decisions/ && find docs/solutions -name '*.md'`
   This is the map. It is generated from disk and cannot drift. Nothing you read later replaces it. Never write it to a file — `docs/wiki/` is curated, never compiled.
2. Read `docs/wiki/index.md` — the catalog, with what each page is for.
3. Read `docs/wiki/log.md`, last 30 lines — what recent sessions did.
4. Read `docs/wiki/STATE.md` — current delta only. Where work stands, not what the project knows. It names a few files in passing; that is not the file list. You have the file list from step 1.

This fires for every session. Debugging, a one-line fix, code, content, strategy. No topic exempts you. If you are about to conclude "this session isn't really about project content," stop — that judgment is the documented failure mode.

### Authority — agent memory is a routing aid, never project authority

For any project-specific fact, decision, constraint, status, prior conclusion, or commitment:

- **Search the wiki before answering or acting.** Do not answer from session memory.
- **Prefer repository evidence over remembered context.** If they disagree, the repository wins and the memory is what's wrong.
- **Name the wiki page you used.** An answer with no cited page is an inference, not a fact.
- **Check freshness and supersedence** before treating a page as current.
- **If no authoritative answer exists, say so explicitly** and label any inference as inference.

Reading the state file every session does not make it the highest authority. It is read last because it is the current operational delta. **Durable pages and explicit decisions outrank it.**

**Delegation.** Never brief a subagent on documented areas from memory. Run the lookup, pass the exact paths. Recall is lossy and every spawned agent multiplies the loss.

### Area read — before editing code

The trigger is the area of code you are touching, not the topic of the conversation.

| Before editing | Read first |
|---|---|
| `<path glob>` | `<area README>` |

Editing in an area with no row above: match your area against the step-1 enumeration before you start. Add a row when an area earns an entry point.

### Write

- Session end: `/wrap` digests to the wiki. Do not hand-roll the digest.
- Before writing anything into `docs/wiki/`, read `docs/wiki/WIKI-CLAUDE.md` (this wiki's filing rules) and `docs/wiki/schema.md` (required frontmatter). They are filing rules, not orientation.
- `docs/wiki/raw/` is immutable. Never edit or delete anything under it.
- A recorded conclusion you disprove is superseded in place, never deleted and never left standing. A stale conclusion is worse than a missing one because it will be trusted.
