# WIKI-CLAUDE.md — {{PROJECT_NAME}} Knowledge Wiki Schema

Claude Code session guidance for the application-level knowledge wiki at [docs/wiki/](.). Read this before creating or editing any file under `docs/wiki/`.

This wiki's primary subject is **{{PROJECT_NAME}}** — {{PROJECT_DESCRIPTION}}. {{SCOPE_NOTE}}

## Conventions

- **Reading is not governed here.** Session orientation lives in this repo's agent-instruction file under "Project memory" (instantiated from the `/wiki` skill's `CLAUDE-MEMORY-BLOCK.md`). This file governs writes only — read it before writing, never to get oriented. Two read protocols drift; pointer-not-copy forbids it.
- **Memory is not authority.** Answer project-specific questions from this wiki, not from session recall; name the page used; if no page answers it, say so and label the inference.
- **Schema** — `schema.md` at this wiki root is the frontmatter authority (required keys, closed type vocabulary, provenance tags). Lint reads it directly; never hand-mirror its rules here.
- **Supersedence** — when a page's conclusion reverses (it asserted X, now asserts not-X, and acting on remembered X would be wrong), retire the old claim into a dated `## Superseded` block at the bottom rather than deleting it or leaving it live. Additive content, rephrasing, and always-wrong corrections are ordinary edits.
- **Pointer, not copy** — a page must not restate a fact whose canonical home is elsewhere; it links it instead.
- **Soft line budget** — curated pages target ~120 lines (a lint nudge, never a gate). `raw/` and `notes/` are exempt.
- **Open questions** — *(include this bullet only once `open-questions.md` exists)* known unknowns live in `open-questions.md` at this wiki root, fed by mid-session gaps, lint escalations, and `/wrap`. The file is created on demand by the first real question, never scaffolded empty.
- **Provenance** — stamp `source` and `confidence` at write time (`/wiki ingest` at promotion; `/wrap` and `/dispatch` default `inferred`). `verified` only when the operator confirms in-session or the claim restates an authority surface. Three values, no more: they exist to separate verified fact from reported information from inference.

Auto-memory at `{{MEMORY_PATH}}` is the **pointer layer** — `MEMORY.md` lists wiki destinations with one-sentence "why this matters" summaries. The wiki itself is the canonical store.

## Wiki operations

Three operations govern how content moves in and out.

### Ingest

Adding new content — a locked decision, an architecture distillation, a strategy brief, a deferred plan, raw source material. Every ingest must:

1. **Write/update the content file** per the file-authority map below.
2. **Update the relevant section's `index.md`** with a one-line entry pointing at the new page. (Do NOT update a global `index.md` for every ingest — section indexes are per-section to avoid co-touching shared files.)
3. **Append a new file to `log/YYYY-MM-DD-<slug>.md`** with date, operation type, scope, and 1-3 sentences of context. Filename slug should describe the ingest event (e.g., `2026-05-07-decisions-seed.md`).
4. **Add at least one cross-reference** from the new page to >=1 sibling, and ensure the new page is linked from its section's `index.md`. Markdown links only — wikilinks `[[label]]` don't render on GitHub.
5. **For external sources** (PDFs, DOCX, exported conversations, screenshots): file under `raw/<subdir>/` and reference from the wiki page that consumes them.

### Query

Using the wiki to answer a question — typically during planning, code review, or session bootstrap. Read [decisions/index.md](decisions/index.md) and the relevant section's `index.md` first to find candidate pages, then drill into them. Synthesize the answer with citations back to the specific wiki pages.

When a synthesis is library-worthy (cross-section, would inform multiple future sessions, not redundant with existing content), file it under [synthesis/](synthesis/) following the same ingest rules above. Examples of library-worthy syntheses: cross-cutting impact assessments, retrospectives that span decisions and architecture, comparative analyses. Most queries don't produce a library-worthy artifact — only file when it's clearly valuable.

### Lint

Periodic health-check. Looks for:

- **Orphans** — pages no other page links to.
- **Index gaps** — pages on disk but not in their section's `index.md`, or in `index.md` but missing on disk.
- **Stale claims** — `decisions/active/*.md` whose preconditions have changed; `architecture/*.md` whose code references no longer match.
- **Contradictions** — pages claiming conflicting facts (especially `decisions/` vs `architecture/` vs `strategy/`).
- **Missing cross-references** — pages that should link based on content overlap but don't.
- **`raw/` orphans** — source material no wiki page references.
- **Lifecycle drift in `decisions/`** — `active/` decisions whose triggers fired but never moved to `superseded/`; `deferred/` decisions whose work has shipped but never moved to `implemented/`.

**Fix policy:** Mechanical issues (index gaps, missing cross-refs, orphan linkage) are auto-fixed. Semantic contradictions are auto-fixed only when code or a locked decision clearly settles the conflict. Everything else is flagged for human review: contradictions without a clear winner, stale claims requiring judgment, lifecycle moves, orphan removals.

## Architectural ground rules

These are inviolable. If a requested change would break one of these, stop and raise it rather than silently working around it.

1. **Markdown only.** Wiki content is prose markdown. Structured data (JSON, YAML config) lives in the codebase, not the wiki. The wiki references code files; it does not duplicate them.
2. **`decisions/active/` is immutable for substantive changes.** To revise a locked decision in a way that changes scope, technical content, or "do not relitigate" trigger semantics, move the existing file to `decisions/superseded/` (preserving its filename) and write a new file in `active/` that references it. Edit-in-place is permitted ONLY for: (a) typos and cross-ref updates, (b) framing clarifications that don't change scope or technical content. When in doubt, prefer supersession; flag the edit in the next `log/` entry so the move can be audited.
3. **`log/` is append-only.** One file per ingest event. Never reorganize, never edit retroactively. To re-classify a past entry, append a new entry that references the old one.
4. **One ingest = one log entry.** Each commit that adds/moves wiki content appends exactly one file to `log/`. Multi-section ingests get one log entry summarizing all affected sections.
5. **`raw/` is read-only.** Source material is preserved as-is. Never edit files under `raw/`. To annotate or correct raw content, write a wiki page that references the raw file and provides the corrected interpretation.

{{PROJECT_SPECIFIC_RULES}}

## File-authority map

Each path is authoritative for a specific class of fact. When the same topic appears in multiple places, the authoritative file wins.

| Path | Authoritative for |
| --- | --- |
| `decisions/active/<date>-<slug>.md` | A single locked decision: scope, rationale, "do not relitigate without" trigger condition |
| `decisions/deferred/<date>-<slug>.md` | A planned change that's been deliberately deferred (with deferral reason + revisit trigger) |
| `decisions/implemented/<date>-<slug>.md` | A formerly-deferred plan that has been executed; preserved with the original date and an `Implemented:` line |
| `decisions/superseded/<date>-<slug>.md` | A formerly-active decision that has been replaced by a later active decision (preserves history) |
| `architecture/<slug>.md` | "How does X work today" — distilled architectural knowledge |
| `overview/<slug>.md` | High-level project context, goals, and domain orientation |
| `strategy/<slug>.md` | Commercial, market, and positioning context — supporting reference for decisions |
| `plans/<slug>.md` | Proposed phase plan or feature design before it becomes a locked decision |
| `research/<date>-<slug>.md` | Investigations, technical exploration, market/domain research |
| `synthesis/<date>-<slug>.md` | Cross-section analyses and reusable query results |
| `sources/YYYY-MM-DD-<slug>.md` | Ingested source summaries with provenance metadata |
| `<section>/index.md` | Per-section catalog: every page in that section with a one-line description |
| `index.md` (top-level) | Catalog of catalogs: links to each section's `index.md` + brief description of what each section holds |
| `log/<date>-<slug>.md` | One ingest event |
| `raw/conversations/<date>-<slug>.md` | Exported Claude Code transcripts that fed an ingest |
| `raw/external/<file>` | PDFs, DOCX originals, and other external source material (binary or text; preserved as-is) |
| `raw/advisory/<file>` | Staged content dropped by an external dispatcher for later ingest promotion |
| `raw/research/<date>-<slug>.md` | Web research staged by `/research-to-wiki` for ingest promotion |

## Decision page template

Every file under `decisions/active/`, `decisions/deferred/`, `decisions/implemented/`, `decisions/superseded/` follows this shape:

```markdown
# <Title>

**Status:** active | deferred | implemented | superseded
**Date:** YYYY-MM-DD
**Scope:** <what work this decision governs>
**Supersedes:** <link to prior decision, if any>
**Superseded by:** <link to replacement, when status=superseded>

## Decision

<1-3 sentence statement of what was chosen.>

## Rationale

<Why this choice over alternatives. Include trade-offs accepted.>

## Cross-references

- <link to architecture/strategy/synthesis pages this decision relates to>

## Do not relitigate without

<The trigger condition that would justify revisiting. E.g., "after Phase 1 delivers" or "if requirements change to require X.">
```

## Cross-reference conventions

- **Markdown links only.** Wikilinks `[[label]]` are NOT used here (they don't render on GitHub).
- **Repo-relative paths** from the linking file: `[some page](../architecture/some-page.md)`, not absolute paths.
- **Every new page links to >=1 sibling** and is linked from its section's `index.md`.
- **`raw/` references** from wiki pages should use a `Sources:` section near the top so future audits can find them quickly.

## Boundary with auto-memory

Three classes in `{{MEMORY_PATH}}`, three rules:

1. **`project_*` memories become pointers.** Two-line shape:
   ```
   See [Title](/absolute/path/to/wiki/file.md).

   One-sentence reason this matters at session start: <e.g. "active iteration; locks scope through date; do not relitigate.">
   ```
2. **`feedback_*` and `user_*` memories stay as-is.** About the user, not project knowledge.
3. **Ephemeral session context stays in auto-memory exclusively.** Wiki = committed knowledge; auto-memory = in-flight state.

## When this wiki does NOT apply

If a requested change would be a better fit elsewhere, push back rather than silently filing it here.

{{EXCLUSION_TABLE}}
