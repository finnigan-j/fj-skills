# Claude Code config backup

Weekly, unattended mirror of the durable parts of my Claude Code setup:
per-project memories, skills, `CLAUDE.md` files, settings and plugin
manifests, plus my shell rc. Kept so a dead laptop does not take months of
accumulated context with it. **This repo is private and must stay private:**
settings and shell files can carry API keys, and memories describe internal
work.

Set up with the `claude-backup` skill from https://github.com/finnigan-j/skills.

## Layout

```
dotclaude/              mirror of the durable parts of ~/.claude
├── CLAUDE.md, settings.json, keybindings.json
├── skills/  agents/  commands/  docs/
├── plugins/            installed_plugins.json + known_marketplaces.json only
└── projects/<slug>/memory/      per-project memory stores
dotclaude-projects/     each {{PROJECTS_DIR}}/<name>/.claude/  (minus worktrees/)
claude-mds/             each {{PROJECTS_DIR}}/<name>/CLAUDE.md, as <name>.md
dotfiles/               {{SHELL_RC}}
claude-backup/
├── bin/                claude-backup, install-launchd, install-systemd, smoke-test
├── launchd/            macOS scheduler definition
└── systemd/            Linux scheduler definition
```

Everything under `dotclaude/`, `dotclaude-projects/`, `claude-mds/` and
`dotfiles/` is machine-written. Do not hand-edit it; edit the live file and
let the next run pick it up. Session transcripts, caches and `~/.claude.json`
(OAuth) are deliberately excluded.

## How it runs

{{SCHEDULE}}. A missed run fires when the machine wakes; a machine that is off
skips that week and the next run captures everything, because each backup is
a full snapshot. Failures raise a desktop notification and are logged.

```sh
claude-backup/bin/claude-backup            # back up now (safe any time)
claude-backup/bin/smoke-test               # exercise the script against a fake HOME
claude-backup/bin/install-launchd          # (re)install the schedule on macOS
claude-backup/bin/install-systemd          # (re)install the schedule on Linux
claude-backup/bin/install-launchd --uninstall
tail -f ~/Library/Logs/claude-backup.log   # macOS log
journalctl --user -u claude-backup -f      # Linux log
```

## Restore on a new machine

```sh
git clone <this repo> {{REPO_PATH}} && cd {{REPO_PATH}}
rsync -a dotclaude/ ~/.claude/              # CAREFUL: overwrites same-named files
for d in dotclaude-projects/*/; do
  n=$(basename "$d"); [ -d ~/projects/"$n" ] || continue
  mkdir -p ~/projects/"$n"/.claude && rsync -a "$d" ~/projects/"$n"/.claude/
done
for f in claude-mds/*.md; do
  n=$(basename "$f" .md); [ -d ~/projects/"$n" ] && cp "$f" ~/projects/"$n"/CLAUDE.md
done
cp dotfiles/zshrc {{SHELL_RC}}
claude-backup/bin/install-launchd           # or install-systemd
```
