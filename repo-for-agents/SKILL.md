---
name: repo-for-agents
description: Restructure a repo into the standard agent/deploy split — app/ is the application, the root is agent context + CI tooling.
disable-model-invocation: true
---

# repo-for-agents

Restructure a repo so what an agent reads and what a server runs never mix. `app/` holds the
entire application; the root holds agent context and CI tooling. Deploy ships `app/` minus
`.deployignore`.

Pick the mode from the repo's state: empty/new → **Scaffold**; existing code → **Migrate**.

## The layout

```
<repo>/
├── CLAUDE.md, .claude/, docs/                        # agent context — repo-only
├── .github/, .gitignore, .deployignore, README.md    # repo/CI tooling — read from root
└── app/                                              # the application — self-contained
    ├── .ddev/                                        # local dev env rides WITH the app
    ├── .gitignore                                    # the app's OWN ignore rules
    └── …                                             # source, config, deps, runtime
```

Three buckets, one rule each:

- **`app/` — the running application.** Complete and self-contained: source, config, deps, its
  own local dev tooling (`.ddev`), and its own `.gitignore`. It stays a normal project of its
  type; moving it in changes nothing inside it. Boot with `cd app && ddev start` (or the app's
  own dev command).
- **Root repo/CI tooling** — `.github/`, a minimal root `.gitignore`, `.deployignore`; read
  from the repo root.
- **Root agent context** — `CLAUDE.md`, `.claude/`, `docs/`, planning/state docs; repo-only.

**Deploy = `app/` minus `.deployignore`.** Agent context and docs are excluded by structure;
dev tooling that must sit inside `app/` (`.ddev`) is excluded by the `.deployignore` list.

Scaffold files live in `~/repos/repo-template` — the single source of truth. Copy from there.

## Scaffold (new repo)

1. Copy the `~/repos/repo-template` structure in; fill the `<PLACEHOLDER>`s in `CLAUDE.md` /
   `README.md`.
2. Put the application in `app/`.
3. Wire deploy (see **Deploy**).

## Migrate (existing repo)

Move the **whole working directory** in place, on a branch. Not `git mv`, not a worktree —
both fail here (see **Why in place**). `mv` catches tracked, untracked, and gitignored files
alike, so the app's **runtime** (`vendor/`, `node_modules/`, `.env`, `storage/`, build output)
travels with it and nothing strands at root.

1. **Classify + make safe.** List every top-level entry including hidden and gitignored ones:
   `ls -A`. Sort each into keep-at-root or move-to-app by the rule below; ask the human on
   ambiguous ones (`README`, `scripts/`, stray docs). Then commit or stash all pending work —
   this is the safety net and clears any dirty-tree collision. Branch.
   *Done when every entry — tracked AND untracked/ignored — is classified and the tree is
   committed or stashed.*

2. **Move.** `mkdir app`, then `mv` every top-level entry except the keep-set (and `.git`, and
   `app` itself) into `app/`. Move whole directories, so no empties are left and no `rm` is
   needed (`rm` is policy-blocked anyway).
   ```bash
   mkdir -p app; shopt -s dotglob
   for e in *; do case "$e" in
     .git|app|CLAUDE.md|.claude|docs|.github|.deployignore|CONTEXT.md|STATE.md) ;;  # keep at root
     *) mv "$e" app/ ;;                                                              # everything else
   esac; done
   ```
   The app's existing `.gitignore` moves into `app/` with it (it's the app's rules).
   *Done when `ls -A` at root shows only the keep-set plus `app/` — nothing else. This check is
   the guard against stranded runtime; do not skip it.*

3. **Record + scaffold root.** `git add -A` (git detects tracked moves as renames; ignored
   files re-ignore under `app/.gitignore`). Copy from `~/repos/repo-template`: `.deployignore`,
   a minimal root `.gitignore`, and `.github/workflows/deploy.yml` **only if deploying via
   GitHub** (see **Deploy**). Keep `CLAUDE.md`/`.claude/`/`docs/` at root; add a root `README`
   if missing.

4. **Verify it boots from `app/`.** For a DDEV repo, `.ddev` moved into `app/`, but DDEV's
   global list still registers the project at the **old repo root** — a plain `ddev start`
   collides. Clear the stale registration first, then start from `app/`:
   ```bash
   ddev stop --unlist <projectSlug>   # projectSlug = name: in app/.ddev/config.yaml
   cd app && ddev start
   ```
   Otherwise run the app's own dev command / build.
   *Done only once the app serves or builds from `app/`.*
   If the old root tracked `.ddev` and DDEV runs during the later fast-forward, a stray ignored
   `.ddev/` (e.g. `addon-metadata`) can regenerate at root — harmless (untracked, never
   deployed); clear it with `find .ddev -depth -delete` (`rm` is blocked).

5. **Commit + hand off.** Commit on the branch; the human fast-forwards their main branch. Name
   any deploy reconfiguration the human still owns.

### Keep vs move

- **Keep at root:** agent context (`CLAUDE.md`, `AGENTS.md`, `.claude/`, `docs/`, planning/state
  docs like `CONTEXT.md`/`STATE.md`) and root tooling (`.github/`, `.deployignore`).
- **Move to `app/`:** everything else — all application source, config, dependencies, build
  tooling, the app's local dev env (`.ddev`), its `.gitignore`, AND its untracked/gitignored
  runtime (`vendor/`, `node_modules/`, `.env`, `storage/`, build output). `mv` brings them all.

### Why in place (not git mv / not a worktree)

- **`git mv` moves only tracked files.** Craft/Node repos are mostly untracked/gitignored
  runtime — `git mv` leaves it stranded at root, which reads as "everything duplicated."
- **A preview worktree is a fresh checkout of `HEAD`**, so it never contains that runtime and
  always looks clean — it validates the one thing that can't break and hides the one that does.

## Deploy

Ask which mechanism the repo uses; wire only that one. Do not impose a full `rsync --delete`.

- **GitHub Actions** (remote is GitHub): copy `deploy.yml` — it rsyncs `app/` minus
  `.deployignore` on push to `main`. Set repo secrets `DEPLOY_HOST`, `DEPLOY_USER`,
  `DEPLOY_SSH_KEY`, `DEPLOY_PATH`; optional var `DEPLOY_SOURCE` for build output.
- **Beanstalk / platform with a repo subpath:** no workflow. Set the deployment's *path in
  repository* to `app`. Flag it for the human.
- **Manual / selective ssh:** no workflow. Deploy the touched files under `app/` to the server
  path (e.g. `app/…` → `/var/www/…/`). Keep the human's existing selective flow; don't replace
  it with a wholesale sync.

`.deployignore` lists paths inside `app/` that must never ship — `.ddev/` first. Add dev-only
paths as they appear.
