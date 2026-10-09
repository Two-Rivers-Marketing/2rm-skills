---
name: configure-deployments
description: Set up GitHub Actions deploys (staging on push, production by hand) from the current repo to its DigitalOcean droplet behind 2RM-Hosting-Firewall.
disable-model-invocation: true
---

# /configure-deployments

Run from inside a repo. Ends with staging deploying on push to `staging`, production deploying by hand from the default branch, one verified **first run** each, and the project's docs updated.

**The kit** — the setup script, workflow templates, and every gotcha — lives in the org repo `Two-Rivers-Marketing/.github` under `deploy/`. Its `README.md` is the single source of truth for how the pattern works and why; this skill is the process. Fetch it first and read the README before step 3:

```bash
KIT=~/.cache/2rm-deploy-kit
if [ -d "$KIT/.git" ]; then git -C "$KIT" pull -q --ff-only; else git clone -q https://github.com/Two-Rivers-Marketing/.github.git "$KIT"; fi
```

The droplet must already exist. If it doesn't, that's `/craft-droplet` first.

## Guardrails

- **SSH budget.** Droplets run ufw `LIMIT` on port 22: a 7th connection inside 30 seconds is refused, and each refused attempt restarts the window. Batch server checks into one `ssh` call; when refused, wait 60+ seconds untouched (a background `sleep`), then retry once.
- **The user runs the setup script and anything that writes `authorized_keys`.** It prompts for the DO token (hidden) and sometimes a sudo password.
- **Production deploys run only on the user's explicit go**, every time.

## Steps

### 1. Facts

Gather and show the user this table, filled:

| Fact | Where to find it |
|---|---|
| `OWNER/NAME`, default branch | `gh repo view --json nameWithOwner,defaultBranchRef` |
| Staging branch | usually `staging`; confirm it exists on the remote |
| App dir | the directory holding `craft` and `composer.json` (`app`, `application`, or `.`) |
| Deploy runbook | the project's docs (search for rsync / deploy); its excludes, build, and post-deploy steps are the spec for step 5 |
| Server IP, SSH user (default `2rmsql`) | runbook, `~/.ssh/config`, or the DO MCP droplet list |
| Staging path + hostname, production path + hostname | runbook; the server's vhost dirs |
| Existing deploy workflows, GitHub environments | `.github/workflows/`, `gh api repos/OWNER/NAME/environments` |

Ask the user once (AskUserQuestion) for whatever the repo can't answer.
**Done when** every row has a value the user has seen.

### 2. Firewall and server

1. **Droplet behind the firewall.** In the DO MCP droplet list, find the droplet by IP and confirm its tags include the firewall's tag (`2RM-Hosting-Firewall` attaches by tag `firewall`; its `droplet_ids` stays empty). If the tag is missing, stop and tell the user: attaching it limits ports 80/443 to Sucuri and the office. Check each hostname's DNS: Sucuri edge IPs are `192.124.249.x`. A hostname that resolves straight to the droplet goes office-only, and its certbot renewal fails.
2. **One SSH call** gathering, per deploy path: whether it exists and has a `.env`; the `ALLOW_ADMIN_CHANGES` line from that `.env` (that line only); `stat -c '%U:%G %a'` of `craft`, `composer.json`, `vendor`, `config/project`, and one template; and the on-server homepage check: `curl -sk -o /dev/null -w '%{http_code}' --resolve HOST:443:127.0.0.1 https://HOST/`. Also `node -v` and `composer -V` once.
3. **Permission standard** (kit README, "Permission standard"). Deploys keep it but can't create it. Where a path's owners or modes differ, the user runs the README's one-time root command on it before its first run.
4. **`ALLOW_ADMIN_CHANGES=true`** on a server means the CP can write project config there. The next deploy reverts that YAML, and `project-config/apply` drops the change from the DB. Recommend `false` on every server, so schema changes happen locally and deploy. The user decides.

An environment whose path, `.env`, or site config is missing is **new**. The setup script creates its directories (step 4). The user supplies its database, `.env`, and web server site config, before its first run in step 6 or 7.

**Done when** the droplet carries the firewall tag, every hostname is classed (Sucuri or direct), the build location is decided (on the server if it has Node, as the runbook does; on the runner if not), each environment's homepage check returns 200 or it is marked new, and each environment is at the permission standard (or the user chose to fix it later) with its `ALLOW_ADMIN_CHANGES` value shown to the user.

### 3. Dry-run

For each environment, deploy a **clean checkout** in dry-run: `git archive origin/<branch>` into a scratch dir, then `rsync -rlz --inplace --checksum --no-p --delete --dry-run --itemize-changes` with the excludes the workflow will use, against that environment's path. CI syncs a clean checkout, never the working tree, so this is what the first run will really do. With `--checksum`, only files whose content differs are listed.

Classify every `*deleting` line and every listed file:

- **Intended**: already removed from git (stale on the server). Fine.
- **Server-managed**: gitignored but needed on the server (`config/license.key`, uploads, caches). Needs an exclude, preferably in `.deployignore`.
- **Uncommitted work**: the server copy matches the local working tree, not the branch. Ask the user: commit it first, or let the deploy revert it.
- **Behind**: an older version of a committed file. Fine; the deploy catches it up.

**Done when** every line is in one of those classes, and every server-managed file has an exclude.

### 4. Setup script

Have the user create a DO token (scopes per the kit README) and run, from the office network:

```bash
~/.cache/2rm-deploy-kit/deploy/setup-deploy.sh --repo OWNER/NAME --server IP \
  --staging-root PATH --staging-site HOST --prod-root PATH --prod-site HOST \
  --checkout "$(git rev-parse --show-toplevel)" --app-dir APP_DIR
```

Add `--staging-server IP` / `--prod-server IP` when an environment lives on a different droplet; drop the `--prod-*` pair to set up staging only. Read the output with the user. A "can't confirm the droplets" warning only means the token lacks `tag:read`; step 2 already confirmed the tag.

**Done when** the script's server section prints `✓ <path> checked` for every deploy path, with each `!` warning either fixed or recorded against a new environment, and `gh` shows: repo secrets `SSH_PRIVATE_KEY`, `DO_API_TOKEN`, `DO_FIREWALL_ID`; on each environment, secrets `SERVER_IP` and `SSH_KNOWN_HOSTS`, variables `DEPLOY_PATH` and `SITE_HOST`, and a branch rule (`staging` and the default branch respectively); and `_deploy.yml`, `deploy-staging.yml`, and `deploy-production.yml` are in `.github/workflows/`.

### 5. Fit `_deploy.yml` to the project

Resolve every `TODO(project)` against the runbook and steps 2–3:

- **Build**: keep the runner build steps, or delete them and build in the post-deploy session (with the build output excluded from rsync).
- **Rsync**: keep the template's flags (`--inplace --checksum --no-p --chmod`, receiver umask 002). They hold the server's permission standard (kit README, "Permission standard") while deploying as `2rmsql`. Change only the excludes: `--exclude-from=.deployignore` if the project has one, plus every server-managed file from step 3.
- **Post-deploy**: the runbook's sequence, in one SSH session. Migrations always run with `--no-backup` (`migrate/all` or `craft up`: Craft's own backup auto-restores on failure, and that restore corrupts DBs here). Keep `db/backup` for production only, and use targeted cache clears.

**Done when** no `TODO(project)` remains, the YAML parses, and the post-deploy remote command, expanded locally with real `ROOT`/`SITE` values, passes `bash -n`.

### 6. First staging run

Commit the workflows and any `.deployignore` change on the staging branch, push, and `gh run watch`.

**Done when** the run succeeds; its `*deleting` lines match step 3's list exactly; files it changed kept their owner and mode on the server (`stat`, against step 2); the log shows `Homepage: 200`; the DO firewall's inbound rules hold no leftover `/32` on port 22; and the public staging URL returns 200 (cache-busted).

### 7. Production

On the user's go:

1. Fast-forward the default branch to the staging branch (`git push origin origin/<staging>:<default>` once `git merge-base --is-ancestor` confirms the fast-forward).
2. Re-run step 3's dry-run for production against the merged commit and show the user the result.
3. The user triggers it (Actions → Deploy production, or `gh workflow run deploy-production.yml --ref <default>`). Watch the run.

**Done when** step 6's checks pass for production and the log names the `db/backup` file (the rollback point). Remind the user to purge Sucuri if the deploy changed anything visible.

### 8. Record it

Through the project's own docs process (its CLAUDE.md says which: `/wiki`, `/wrap`, …): a decision recording CI deploys, and the runbook updated so CI is the normal path and manual rsync the office-only fallback. Include the workflows, environments, the server steps, the first run IDs, and anything step 3 added to the excludes. Hide superseded workflows with `gh workflow disable`.

**Done when** someone reading only the project's docs can tell how a deploy happens, where its settings live, and how to deploy by hand.

### 9. CLAUDE.md

Add (or replace) a short `## Deployments` section in the repo's root `CLAUDE.md`, filled with this repo's branches. Keep it this brief; details belong in the runbook from step 8.

```markdown
## Deployments

- **Staging** auto-deploys on push to `<staging>` (GitHub Actions).
- **Production** deploys manually: `gh workflow run deploy-production.yml --ref <default>` (or Actions → Deploy production).
- No permission prompt needed for either. When the user says to merge or push to a deploy branch, that is the go-ahead for the deploy it triggers.
- Runbook: <link to the step 8 runbook>.
```

**Done when** `CLAUDE.md` has the section with real branch names and a working runbook link, and it's committed alongside step 8's docs.
