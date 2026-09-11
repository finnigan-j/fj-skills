---
name: claude-backup
description: Sets up, runs and restores an unattended weekly backup of Claude Code memories, skills, CLAUDE.md files and settings into a private git repo. Use whenever the user wants to back up, mirror, snapshot, preserve, migrate or restore ~/.claude or their Claude Code memory, skills or config; worries about losing memories if a laptop dies or is replaced; wants the same Claude Code setup on a new machine; or asks to schedule or automate a backup of their Claude setup, even without saying "backup".
---

# Claude Backup

## Overview

Claude Code keeps its most valuable state on one disk: per-project memories in
`~/.claude/projects/<slug>/memory/`, skills, `CLAUDE.md` files and settings.
This skill gives the user a small bash script plus a scheduler job that mirror
the durable parts of that state into a private git repo every week, and a
documented way to put it all back on a new machine.

Three design rules make it safe to run unattended. Keep them intact:

1. **Allowlist, not denylist.** Only the paths listed in `references/scope.md`
   are copied. Whatever Claude Code adds to `~/.claude` next (caches,
   transcripts, token files) stays out by default.
2. **Exact mirror of four roots.** `dotclaude/`, `dotclaude-projects/`,
   `claude-mds/` and `dotfiles/` are rebuilt from scratch each run and are the
   only paths the script ever stages. Nothing else in the repo can be swept
   into a backup commit. Deletions propagate; git history keeps the old file.
3. **Snapshot, not incremental.** A missed week loses nothing; the next run
   captures the current state.

Bundled files, relative to this skill's directory (`${CLAUDE_SKILL_DIR}`; if
that is empty, use the base directory shown when this skill loaded):

| Path | Purpose |
|---|---|
| `scripts/claude-backup` | the backup script, bash, `set -euo pipefail` |
| `scripts/install-launchd`, `assets/local.claude-backup.plist` | macOS scheduler |
| `scripts/install-systemd`, `assets/claude-backup.service`, `assets/claude-backup.timer` | Linux scheduler |
| `scripts/smoke-test` | end-to-end test against a fake HOME and a scratch repo |
| `assets/REPO-README.md` | README template for the user's backup repo |
| `references/scope.md` | what is mirrored, what is not, and why |
| `references/restore.md` | restoring on a new machine, troubleshooting |

## Which task?

| The user wants to... | Section |
|---|---|
| set this up for the first time | Setup |
| run a backup right now | Run now |
| know whether backups are working | Check |
| get their config onto a new machine | Restore |
| change when it runs, or turn it off | Schedule |

## Setup

### 1. Confirm the inputs

Five facts decide everything. Ask only for the ones the conversation has not
already supplied, and propose the default for each.

| Input | Default | Why it matters |
|---|---|---|
| Backup repo: path, **private**, remote `origin`, default branch `main` | the current repo | The script refuses to run off `main` and pushes to `origin`. Private is non-negotiable: settings files can carry API keys in `env` blocks, and memories describe internal projects. |
| Projects directory | `~/projects` | Every `<projects>/<name>/.claude/` and `CLAUDE.md` is mirrored. |
| Shell rc file | `~/.zshrc` | Mirrored into `dotfiles/`. Often exports tokens; step 5 checks. |
| Schedule | Saturday 12:00 local | Weekly suits a snapshot. Pick a time the machine is usually awake. |
| OS | detect with `uname` | launchd on macOS, systemd user timer on Linux. |

Check visibility rather than assuming: when the remote is on GitHub and `gh`
is installed, `gh repo view --json visibility -q .visibility` must print
`PRIVATE`; otherwise ask the user to confirm. If the repo is public, stop and
say so. Do not continue with a public repo.

### 2. Copy the bundled files into the repo

Copy the scripts into the backup repo instead of running them from the skill
directory. The script finds its repo with `git rev-parse --show-toplevel` on
its own location, and a repo that carries its own tooling can be restored on
a machine that does not have this skill installed. From the repo root:

```sh
mkdir -p claude-backup/bin claude-backup/launchd claude-backup/systemd
cp "${CLAUDE_SKILL_DIR}"/scripts/claude-backup "${CLAUDE_SKILL_DIR}"/scripts/install-launchd \
   "${CLAUDE_SKILL_DIR}"/scripts/install-systemd "${CLAUDE_SKILL_DIR}"/scripts/smoke-test claude-backup/bin/
cp "${CLAUDE_SKILL_DIR}"/assets/local.claude-backup.plist claude-backup/launchd/
cp "${CLAUDE_SKILL_DIR}"/assets/claude-backup.service "${CLAUDE_SKILL_DIR}"/assets/claude-backup.timer claude-backup/systemd/
chmod +x claude-backup/bin/*
grep -qx '.DS_Store' .gitignore 2>/dev/null || echo '.DS_Store' >> .gitignore
```

### 3. Apply the inputs

- **Non-default projects dir, shell rc or Claude dir:** set `PROJECTS_DIR`,
  `SHELL_RC` (and, rarely, `CLAUDE_DIR`) as environment variables in the
  scheduler definition rather than editing the script, so the script stays
  byte-identical to the tested one. launchd:
  add keys to the plist's `EnvironmentVariables` dict. systemd: add
  `Environment=PROJECTS_DIR=/path` lines to the service unit. For manual
  runs, export them in the shell or note them in the README.
- **Schedule:** edit `StartCalendarInterval` in the plist (`Weekday` 0 is
  Sunday, 6 is Saturday) or `OnCalendar=` in the timer.

### 4. Prove it works before touching the real repo

```sh
claude-backup/bin/smoke-test
```

The test builds a fake HOME and a scratch repo with a bare origin, runs the
backup four times and checks 25 things: what is mirrored, what is excluded,
that deletions propagate, that an unrelated file is never committed, that a
no-op run makes no commit, and that the wrong branch is refused. Expect
`25 passed, 0 failed`. It works entirely inside a temporary directory under
`$TMPDIR` and never touches the real `~/.claude` or the real repo.
Fix any failure before continuing: an unattended job that is wrong on day one
is wrong every week.

### 5. Look for secrets before the first commit

The mirror is an allowlist, but a few allowed files commonly carry secrets:
`settings.json` and `settings.local.json` (`env` blocks, MCP server configs)
and the shell rc (`export SOME_TOKEN=`). Check the live sources:

```sh
grep -n -i -E 'sk-[a-z0-9]|ghp_|AKIA|token|secret|password|api[_-]?key' \
  ~/.claude/settings.json ~/.claude/settings.local.json ~/.zshrc \
  ~/projects/*/.claude/settings.local.json 2>/dev/null
```

Show every hit and let the user decide: move the secret into a separate,
unmirrored file that the rc sources, or accept it because the repo is
private. Do not continue past a hit without the user's answer.

### 6. Commit the tooling and write the README

Render `assets/REPO-README.md` into the repo's `README.md`, filling in
`{{SCHEDULE}}` (for example "Runs every Saturday at 12:00 local time via
launchd"), `{{PROJECTS_DIR}}`, `{{SHELL_RC}}` and `{{REPO_PATH}}` (where the
user keeps this clone). Then commit by hand, because the backup script only
ever stages the four mirror roots:

```sh
git add claude-backup README.md .gitignore
git commit -m "Add claude-backup: weekly mirror of Claude Code config"
git push
```

### 7. First run, by hand

```sh
claude-backup/bin/claude-backup
git log -1 --stat
git status -sb
```

The script prints timestamped log lines, including how many memory files
across how many projects were mirrored. Expect a commit titled
`backup: <date> — N file(s) changed` and `main...origin/main` in sync.

### 8. Install the scheduler

macOS:

```sh
claude-backup/bin/install-launchd
launchctl print "gui/$(id -u)/local.claude-backup" | grep -E 'state|last exit code'
```

Linux:

```sh
claude-backup/bin/install-systemd
systemctl --user list-timers claude-backup.timer
loginctl enable-linger "$USER"
```

Prefer these over cron. launchd's `StartCalendarInterval` and systemd's
`Persistent=true` both fire a missed run when the machine wakes; cron skips it
silently. The Linux installer was authored on macOS and syntax-checked only,
so confirm the timer is listed.

### 9. Report

Tell the user what was mirrored (the script's count), the commit on `origin`,
when the next run is, where the log lives, and the one-line command to back
up by hand.

## Run now

From anywhere inside the repo: `claude-backup/bin/claude-backup`. Safe at any
time; a run with nothing new logs "nothing to back up" and exits 0. Through
the scheduler: `launchctl kickstart "gui/$(id -u)/local.claude-backup"` or
`systemctl --user start claude-backup.service`.

## Check

- `git -C <repo> log --oneline -8`: expect `backup:` commits on schedule. A
  quiet week makes no commit; that is the no-op path, not a failure.
- Log: `tail -n 30 ~/Library/Logs/claude-backup.log` on macOS,
  `journalctl --user -u claude-backup -n 30` on Linux.
- Job state: `launchctl print "gui/$(id -u)/local.claude-backup" | grep -E 'state|last exit code'`
  or `systemctl --user status claude-backup.timer`.
- Failures raise a desktop notification and a log line starting `ERROR:`.
  Causes and fixes are in `references/restore.md`.

## Restore

Follow `references/restore.md`. In short: clone the backup repo, `rsync`
`dotclaude/` into `~/.claude/` (this overwrites same-named files, so say so
first), copy per-project `.claude/` and `CLAUDE.md` only for projects that
exist on the new machine, copy the shell rc, then re-run the installer so the
new machine keeps backing up.

## Schedule

Change the time in `claude-backup/launchd/local.claude-backup.plist` or
`claude-backup/systemd/claude-backup.timer`, commit, and re-run the installer;
it replaces the existing job. Turn it off with `install-launchd --uninstall`
or `install-systemd --uninstall`. To widen what is backed up, read the
"Extending" section of `references/scope.md` first.

## Common mistakes

| Mistake | Why it bites | Instead |
|---|---|---|
| Running the script from the skill directory | It resolves its repo from its own path, so it refuses to run or writes into the wrong repo | Copy it into the backup repo first (step 2) |
| "Just back up all of `~/.claude`" | Transcripts run to hundreds of MB and grow weekly; `~/.claude.json` holds OAuth tokens; caches churn every session | Keep the allowlist; extend it deliberately via `references/scope.md` |
| A public repo "because it's only config" | `settings.json` `env` blocks and shell rcs carry API keys; memories name internal systems | Private repo, plus the grep in step 5 |
| cron instead of launchd or systemd | cron skips the run while the lid is closed and never says so | The bundled installers; both fire on wake |
| Hand-editing files under the four mirror roots | The next run overwrites them | Edit the live file in `~/.claude`; the next run picks it up |
| Expecting the backup commit to include other files | Only the four roots are staged, by design | Commit your own files yourself |
| Skipping the smoke test | A wrong scope or a wrong branch check goes unnoticed for weeks | Run it; expect 25 passed |

## Quick reference

| Task | Command |
|---|---|
| Back up now | `claude-backup/bin/claude-backup` |
| Test | `claude-backup/bin/smoke-test` |
| Install or remove schedule, macOS | `claude-backup/bin/install-launchd [--uninstall]` |
| Install or remove schedule, Linux | `claude-backup/bin/install-systemd [--uninstall]` |
| Log | `~/Library/Logs/claude-backup.log` or `journalctl --user -u claude-backup` |
| Env overrides | `CLAUDE_BACKUP_REPO` `CLAUDE_DIR` `PROJECTS_DIR` `SHELL_RC` `CLAUDE_BACKUP_NO_NOTIFY` |
