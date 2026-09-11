# Restoring on a new machine, and troubleshooting

## Restore

Prerequisites on the new machine: git, rsync, and access to the private
backup repo. Claude Code itself can be installed before or after.

```sh
git clone <private-remote> ~/projects/agents      # or wherever the user keeps it
cd ~/projects/agents

# 1. Global ~/.claude: memories, skills, CLAUDE.md, settings, plugin manifests.
#    Overwrites same-named files in an existing ~/.claude. Say so before running.
mkdir -p ~/.claude
rsync -a dotclaude/ ~/.claude/

# 2. Per-project .claude/ folders, only for projects that exist here.
for d in dotclaude-projects/*/; do
  n=$(basename "$d"); [ -d ~/projects/"$n" ] || continue
  mkdir -p ~/projects/"$n"/.claude && rsync -a "$d" ~/projects/"$n"/.claude/
done

# 3. Per-project CLAUDE.md, same rule.
for f in claude-mds/*.md; do
  n=$(basename "$f" .md); [ -d ~/projects/"$n" ] && cp "$f" ~/projects/"$n"/CLAUDE.md
done

# 4. Shell rc. Review the diff first if a rc already exists on this machine.
cp dotfiles/zshrc ~/.zshrc

# 5. Plugins: Claude Code re-installs from the mirrored manifests on next start.
#    If it does not, re-add each marketplace listed in
#    ~/.claude/plugins/known_marketplaces.json with /plugin marketplace add.

# 6. Keep backing up from the new machine.
claude-backup/bin/install-launchd        # macOS
claude-backup/bin/install-systemd        # Linux
```

If the projects directory or shell rc differs on the new machine, adjust the
paths above and set `PROJECTS_DIR` / `SHELL_RC` in the scheduler definition
as described in SKILL.md, step 3.

Memory slugs encode the absolute project path (for example
`-Users-me-projects-app`). If the new machine uses a different username or
projects root, the mirrored memories will not line up with the new paths.
Rename the directories under `~/.claude/projects/` to the new slugs, which
Claude Code derives by replacing `/` with `-` in the absolute path.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ERROR: repo is on 'feature', not 'main'` | Someone left the backup repo on another branch | `git checkout main`; the next run proceeds |
| `ERROR: local main has diverged from origin` | Commits were made on `main` both locally and elsewhere | Resolve by hand (`git pull --rebase` or merge), then re-run |
| `WARN: could not reach origin` then `ERROR: committed locally; push skipped (offline)` | Machine was offline at run time | Nothing to do; the next run pushes |
| Notification "Claude backup failed" with `push failed` | Credentials or remote changed | Run `git push` in the repo by hand to see the real error |
| No commits for weeks, log says "nothing to back up" | Nothing changed | Normal. Confirm by editing a memory and running by hand |
| No log lines at all on the scheduled day | Machine was powered off, or the job was never loaded | `launchctl print "gui/$(id -u)/local.claude-backup"` or `systemctl --user list-timers`; re-run the installer if missing |
| Job runs but `git` or `rsync` not found | Scheduler PATH lacks the tool's directory | Add it to `EnvironmentVariables.PATH` in the plist or `Environment=PATH=` in the unit |
| Smoke test fails on a fresh machine | Usually `git` identity missing or bash too old | `git config --global user.name/email`; bash 3.2 or newer is required |
| Secret found by the setup grep | An `env` block or rc export | Move it to an unmirrored file, or accept it knowing the repo is private |
