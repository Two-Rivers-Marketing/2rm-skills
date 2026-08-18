---
name: wiki
description: >
  Canonical project wiki curator — bootstrap, curate, and maintain a project wiki.
  Use when the user wants to set up project memory, ingest docs, log a session,
  lint wiki health, convert existing docs, or make the wiki browsable.
  Also triggers on /wiki and any /wiki subcommand.
---

# /wiki — Project Wiki Curator

One skill, six modes. The single gateway for all wiki reads and writes.

## Ground Rules

1. **Wiki skill is the only writer to curated pages.** External producers stage to `raw/<source>/`.
2. **WIKI-CLAUDE.md governs.** Read it before every write. Non-negotiable.
3. **raw/ is immutable.** Never modify files in `raw/`.
4. **Curation over completeness.** Only persist what future sessions can't derive from code.
5. **Absolute dates.** YYYY-MM-DD everywhere. Never relative.
6. **Cite sources.** Every claim references its source page.
7. **Cross-reference liberally.** Update index.md on every write.
8. **Never silently overwrite contested claims.** Surface both versions, let the user decide.
9. **File reusable synthesis.** When answering a question from wiki content and the synthesis is non-trivial, offer to file it as a new wiki page rather than re-deriving it next time.

## Write Safety

Two tiers govern every fix:

- **Auto-fix (mechanical):** missing index entries, dead links, log formatting, missing cross-refs.
- **Flag for user (semantic):** contradictions between pages, stale claims where the correct answer is unclear, decisions whose triggers may have fired ambiguously.

When authority is clear (feature demonstrably shipped, code clearly contradicts docs, no protecting decision), auto-fix semantic issues too. When ambiguous, always flag.

## Source Authority

| Source | Authority | Behavior |
|---|---|---|
| `raw/gsd/`, `raw/external/` | **Trusted** | Auto-curate with high confidence |
| `raw/research/` | **Trusted provenance** | Auto-curate but cite and qualify external claims |
| `raw/advisory/` | **Advisory** | Evaluate against wiki state: promote, adapt, flag, or skip |

Advisory drops get four outcomes: **promote** (new, compatible), **adapt** (relevant, needs reframing), **flag** (conflicts with existing wiki — surface both claims), **skip** (redundant — mark `status: skipped` with reason). The project wiki has final say.

## Wiki Detection

Resolve location in order: explicit path → `docs/wiki/WIKI-CLAUDE.md` → `wiki/WIKI-CLAUDE.md` → root `WIKI-CLAUDE.md` → structural markers (`decisions/` + `log/` without schema file). No wiki found → suggest `/wiki init`.

---

## Mode Routing

| Command | Mode |
|---|---|
| `/wiki init` | init |
| `/wiki convert` | convert |
| `/wiki ingest` | ingest |
| `/wiki log` | log |
| `/wiki lint` | lint |
| `/wiki human-version` | human-version |

**Smart routing** (no args or natural language): pending `raw/` files → ingest. Session produced commits → log. Question about wiki content → query. "wrap up" / "done" → log. "health check" → lint. "browsable" / "onboard" / "mkdocs" → human-version.

Announce the detected mode before proceeding so the user can redirect.

---

## Modes

### init — Bootstrap a new wiki

Discover project context (README, CLAUDE.md, package.json, git log), ask 2-3 setup questions (project name, exclusions, seed decisions), scaffold the canonical layout using this skill's `assets/WIKI-CLAUDE-TEMPLATE.md` plus `assets/schema-TEMPLATE.md` (as `schema.md` at the wiki root — the frontmatter/lint authority), seed index + overview + first log entry.

**Do not create `open-questions.md`.** It is created lazily, on the first real question (see Open questions, below). Scaffolding it produces an empty file that nothing feeds and a pointer that outlives its content.

**Then write the memory block into the repo's root `CLAUDE.md`** — instantiate this skill's `assets/CLAUDE-MEMORY-BLOCK.md` per its instantiation table, creating `CLAUDE.md` if absent. This is not a "pointer"; it is the repo's whole read contract. Scan for a sibling knowledge store (`docs/solutions/`, or any `docs/<dir>/` that isn't the wiki) and seed the Area read table from it. Never emit a pointer to a file that doesn't exist.

Reading is governed by `CLAUDE.md` alone. `WIKI-CLAUDE.md` governs writes and must never carry a session-orientation protocol — two read protocols drift, and pointer-not-copy forbids it.

If existing docs are found, suggest `/wiki convert` instead.

### convert — Reshape existing docs

Archive every existing doc verbatim to `raw/external/` (immutable), scaffold the wiki skeleton including `schema.md` — but not `open-questions.md`, which is created lazily (see init, above) — then curate each file into the appropriate section. Curated versions may restructure and cross-reference but must not lose information from the originals.

Write the memory block into the repo's root `CLAUDE.md` exactly as `init` does, replacing whatever section currently holds the wiki read instructions (`## Read this first`, `## Knowledge wiki`, `## Project Memory`). Leave `## Operating rules` and other repo-specific law alone.

Never overwrite `raw/external/`. Re-runs stage alongside as `raw/external/<folder>-YYYY-MM-DD/`.

### ingest — Process raw/ into curated pages

The core curation pipeline. Scan `raw/` for unprocessed files, process by authority level (see Source Authority above). Ingest means integrating — touching every page the source affects, not just creating one new page.

**Provenance stamping (ADR 0010 §5).** Assign `source` + `confidence` at promotion — never leave a promoted page untagged. `confidence: reported` when the raw source is faithfully captured (its own claims stay unvetted); `inferred` when the page is agent synthesis with no direct source statement; `verified` only when the operator confirms it in-session, or the claim restates one of the project's **authority surfaces** — the files that own a fact outright, declared per project (in this operator's setup: `context/priorities.md`, `decisions/log.md`, `state/<slug>.md`, `clients.yaml`). A project with no such surfaces simply has fewer routes to `verified`. Never minted automatically otherwise. Full vocabulary in the wiki's `schema.md`.

### log — Session wrap-up

Extract wiki-worthy knowledge from the current session and **write directly to curated pages** — this is the one mode that skips `raw/` staging because the agent IS the source.

Scan recent commits, conversation decisions, deferrals, architecture changes. Categorize into decisions, architecture, capability, or deferral pages. Then run a staleness sweep: do overview pages match what was built? Did this session resolve any deferred decisions? Apply Write Safety rules.

### lint — Wiki health check

Apply Write Safety: auto-fix mechanical issues, flag ambiguous semantic ones for the user.

**Scripted pass.** When the repo ships `scripts/wiki-lint.py`, run it first. The checks it actually implements, as of 2026-07-27:

| Tier 1 (mechanical) | What it catches |
|---|---|
| `frontmatter` / `type-vocab` | missing required keys; a type outside the schema's vocabulary |
| `confidence-backfill` | no `confidence:` — backfilled from whether `source:` exists |
| `dead-link` | a link target that does not exist, in **either** syntax |
| `link-case` | a link that *resolves* but to a differently-cased file — silently lands on another document on a case-insensitive filesystem |
| `index-gap` | a page no other page links to |
| `stale-path` | an absolute cross-repo path that no longer exists |
| `path-trigger` | an area-read row naming an entry point that isn't there |
| `date-drift` / `pointer-not-copy` | stale dates; a page restating a fact whose home is elsewhere |

| Tier 2/3 (judgment) | What it catches |
|---|---|
| `supersedence` | a conclusion-like line changed with no matching `## Superseded` entry |
| `wiki-contradiction` | two pages asserting opposite things, tie-broken by `confidence` |
| `op-drift` | an operational surface outliving its registry status |

Retired 2026-07-27, do not expect them in output: `line-budget` (a nudge nobody acted on), `transition-marker` (dated from git blame, too speculative), `authority-order` (ran in one repo only, widest surface for the least reach).

Append-only logs are exempt from link findings — they record what was true then and must not be edited retroactively, so a finding there is one nobody is allowed to fix.

All config derives from the wiki's `schema.md`, including the structural-exemption list:

```
python3 scripts/wiki-lint.py --report state/wiki-lint.md          # read-only
python3 scripts/wiki-lint.py --report state/wiki-lint.md --fix    # + backfill + tier-3 escalation
```

Read-only by default; `--fix` applies the confidence backfill and appends tier-3 findings to the wiki's `open-questions.md` (deduped). Commit both with a receipt (`docs(wiki): lint auto-fix — <n> pages`) — keep the words `lint auto-fix` in the subject, the next run excludes those commits from its edited-pages window so a bulk backfill doesn't look like 91 hand edits. Tier-2 findings ship a drafted correction, never auto-applied — apply the unambiguous ones yourself, and remember the direction: the operational layer wins over the wiki, and between two wiki pages the *lower*-confidence side gets corrected.

The script only reaches what's mechanizable. Work the rest by hand in the same sweep: any claim the wiki makes that one of the project's authority surfaces contradicts, and value drift in prose (a figure the registry contradicts — the script deliberately doesn't guess, since pricing prose quotes numbers for a dozen honest reasons). Anything you can't adjudicate is a known unknown; record it where the project keeps those (see the open-questions asset below) rather than burying it in the report.

*Optional adapter — an operator with a cross-project registry and operational surfaces (a project index, a decision log, per-project state files) gets the extra authority-order pass by hand; a standalone project simply has fewer surfaces to check against and skips it.*

**Multi-wiki sweep — PAUSED as of 2026-07-27.** `scripts/wiki-sweep.py` runs the same lint across every registered wiki and rolls findings per-repo into one report. Its scheduled trigger is **unloaded**: for months it reported *clean* on wikis whose primary link syntax the linter could not parse, which is a false assurance rather than a gap. The parser is fixed; the sweep stays off until the rebuilt lint has run manually for a couple of cycles and is shown to surface things worth acting on. Re-enable or delete deliberately — do not re-enable it as a side effect of other work.

Run it by hand meanwhile:

```
python3 scripts/wiki-sweep.py --report <path the repo owns>
```

Fixes apply only in the repo that owns the lint; other repos are report-only at every tier, no cross-repo writes. A non-zero exit means a repo went unswept — never call the sweep complete in that case.

### human-version — Browsable static site

Wrap the wiki in MkDocs Material so humans can browse it without AI. Set up `mkdocs.yml` with curated `nav:`, generate an onboarding landing page that **synthesizes** (not duplicates) key wiki content, and generate how-to guides grounded in **real git history** — actual commits, actual files, actual diffs.

Exclude wiki internals from nav (WIKI-CLAUDE.md, log.md, raw/). Onboarding page is always the landing page.

**Nav update protocol:** when wiki pages are added or moved after setup, flag the needed `mkdocs.yml` change to the user. Don't update nav silently. Content edits hot-reload automatically via `mkdocs serve`.

---

## Open questions — lazy creation

`open-questions.md` at the wiki root holds known unknowns that block or degrade knowledge. It is
**optional and created on demand**, never scaffolded.

- Nothing creates it until a real question needs recording. At that moment — `/wrap`'s parking
  step, a lint escalation, or a mid-session gap — instantiate `assets/open-questions-TEMPLATE.md`
  and append the entry in the same write.
- `WIKI-CLAUDE.md` mentions it only in wikis where it exists. Never point at an absent file.
- Lint treats it as an optional operational file: absent is valid, and its absence is never a
  finding. An escalation with nowhere to go creates it.
- A file whose `## Open` section has emptied out can be deleted; git history is the ledger.

Rationale: a 2026-07-27 audit found nine of thirteen copies were empty templates created to
satisfy a pointer introduced hours earlier. Nine real entries existed, in two repos. The workflow
is sound where it is used; the mandate was the defect.

## Shared Behaviors

**Every write-producing mode:** log the operation, update affected indexes, emit a receipt (files created/updated/skipped, flagged items), commit with `docs(wiki): <mode> — <summary>`.

**Frontmatter on new pages:** when the wiki has a `schema.md` at its root (ADR 0010 §8), it is the authority — required keys, closed type vocabulary, provenance tags. Where no `schema.md` exists yet, fall back to the 2026-07-07 OKF-minimal rule: every NEW curated page gets at least `type:` and `timestamp:`. Never retrofit existing pages for format alone — wikis converge through normal churn, not a migration pass.

**Staleness guard:** during any session work, catch obvious staleness inline (you just built the thing the wiki says is deferred). During `/wiki log`, run a broader sweep across all overview and decision pages.

## Anti-Patterns

- Empty placeholder pages and premature folder taxonomies
- Asking the user to fill out forms — you do the bookkeeping
- Filing decisions without updating the pages they affect
- Letting index.md drift from reality
- Re-deriving the same synthesis every query — file it the second time
