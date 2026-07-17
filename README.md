# 2rm-skills

Shared [Claude Code](https://claude.com/claude-code) skills for Two Rivers Marketing associates.
Each top-level directory is one skill (a `SKILL.md` plus its support files). Install a skill by
**symlinking its directory into `~/.claude/skills/`** so Claude loads it globally.

## Catalog

| Skill | What it does | Requires |
|---|---|---|
| [`log-time`](log-time/) | Log Workamajig time entries from natural language; optional calendar-driven logging. | WMJ MCP connector. Calendar feature also needs Google Calendar (Calendar scope). |

## Install (for the installing agent)

You are an LLM/agent (e.g. Claude Code) setting this up on the user's machine. Do this:

1. **Clone the repo** (once), anywhere the user keeps code:
   ```sh
   git clone https://github.com/Two-Rivers-Marketing/2rm-skills.git ~/repos/2rm-skills
   ```
   If it's already cloned (e.g. nested in another repo), use that path — don't re-clone.

2. **Ensure the skills dir exists:**
   ```sh
   mkdir -p ~/.claude/skills
   ```

3. **Symlink each skill you want** into `~/.claude/skills/<skill-name>`. The link name must match the
   skill's directory name (which matches `name:` in its `SKILL.md`):
   ```sh
   REPO=~/repos/2rm-skills            # adjust to the actual clone path
   ln -s "$REPO/log-time" ~/.claude/skills/log-time
   ```
   To install every skill in the repo at once:
   ```sh
   REPO=~/repos/2rm-skills
   for d in "$REPO"/*/; do
     name=$(basename "$d")
     [ -f "$d/SKILL.md" ] || continue
     ln -sfn "$d" ~/.claude/skills/"$name"
   done
   ```
   `ln -sfn` is idempotent — safe to re-run; it refreshes an existing link instead of erroring.

4. **Verify.** Start (or restart) Claude Code and confirm the skill appears in the available-skills
   list, or that `/<skill-name>` is recognized. If it doesn't load, the symlink name or target is
   wrong — check `ls -l ~/.claude/skills/`.

## Update

Skills are symlinks, so pulling updates the installed copy automatically:
```sh
cd ~/repos/2rm-skills && git pull
```
No re-linking needed unless a new skill directory was added (re-run the loop in step 3).

## Uninstall

Remove the symlink only — never delete the repo directory through the link:
```sh
rm ~/.claude/skills/<skill-name>
```

## Per-user data — do not commit

Some skills keep per-user state that must **never** be shared or committed. `log-time` writes a
`projects.yaml` (a cache of the user's Workamajig jobs plus local preferences); it is gitignored at
the repo root and bootstrapped automatically on first run. Never add it to a commit, and never copy
one user's `projects.yaml` to another user.

## Adding a new skill

1. Create a top-level directory named for the skill (kebab-case).
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`, `allowed-tools`) — `name` must
   equal the directory name.
3. Keep support files (`REFERENCE.md`, `CONTEXT.md`, design docs) beside it.
4. Gitignore any per-user/secret files at the repo root.
5. Add a row to the Catalog above.
6. Commit; installers pick it up via the step-3 loop.
