# Scope: what claude-backup mirrors, what it leaves out, and why

The script builds an allowlist. Anything in `~/.claude` that is not named in
the first table is not copied, so new state Claude Code invents later stays
out until someone decides otherwise.

## Mirrored

| Live path | Repo path | Why it is worth keeping |
|---|---|---|
| `~/.claude/projects/<slug>/memory/` | `dotclaude/projects/<slug>/memory/` | Per-project memories: the accumulated context that is slowest to rebuild. The reason the script exists. |
| `~/.claude/skills/` | `dotclaude/skills/` | Hand-written skills. |
| `~/.claude/agents/`, `commands/`, `docs/` | `dotclaude/<same>/`, when present | Custom agents, legacy commands, personal notes. |
| `~/.claude/CLAUDE.md` | `dotclaude/CLAUDE.md` | Global instructions. |
| `~/.claude/settings.json`, `settings.local.json`, `keybindings.json` | `dotclaude/<same>`, when present | Model, permissions, hooks, keybindings. |
| `~/.claude/plugins/installed_plugins.json`, `known_marketplaces.json` | `dotclaude/plugins/<same>` | Enough to reinstall every plugin; the cache itself is not needed. |
| `<projects>/<name>/.claude/`, minus `worktrees/` | `dotclaude-projects/<name>/` | Per-project settings, project skills, launch configs. Often gitignored in the project itself. |
| `<projects>/<name>/CLAUDE.md` | `claude-mds/<name>.md` | Per-project instructions, frequently kept out of the project repo. |
| Shell rc (`~/.zshrc` by default) | `dotfiles/zshrc` (or `bashrc`) | Aliases and functions the user leans on. |

Rules that apply to every row:

- Empty memory directories, or ones holding only `.DS_Store`, are skipped.
  Git cannot store an empty directory anyway.
- `.DS_Store` is excluded everywhere and belongs in the repo's `.gitignore`.
- The four repo roots are exact mirrors: a file deleted locally disappears
  from the repo on the next run. Git history keeps it.
- Everything under the four roots is machine-written. Hand edits are
  overwritten on the next run.

## Excluded

| Path | Why it stays out |
|---|---|
| `~/.claude/projects/*/*.jsonl` (session transcripts) | Hundreds of MB and growing every week. Rarely worth reading back; grows the repo without bound. |
| `~/.claude.json` | Holds OAuth material and account state. Never mirror it. |
| `~/.claude/plugins/cache/`, `plugins/data/`, `plugins/marketplaces/`, `plugin-catalog-cache.json` | Rebuilt from the two manifest files that are kept. |
| `~/.claude/shell-snapshots/`, `file-history/`, `paste-cache/`, `session-env/`, `sessions/`, `tasks/`, `jobs/`, `telemetry/`, `cache/`, `backups/`, `downloads/`, `feedback/`, `daemon*`, `history.jsonl`, `*-cache.json`, `policy-limits.json`, `remote-settings.json` | Caches and per-session state. Regenerated on use; noisy in diffs. |
| `<projects>/<name>/.claude/worktrees/` | Full working copies created by worktree tooling. Can be enormous and are not config. |

## Secrets

The allowlist keeps the obvious token file out, but three allowed inputs
commonly carry secrets anyway:

- `settings.json` / `settings.local.json`: `env` blocks and MCP server
  definitions.
- Per-project `.claude/settings.local.json`: same.
- The shell rc: `export FOO_TOKEN=...` lines.

That is why the backup repo must be private, and why setup greps these files
before the first commit. A user who wants a cleaner rc can move exports into
a separate file the rc `source`s and leave that file unmirrored.

## Extending

- **Transcripts.** If the user insists, mirror `~/.claude/projects/` with
  `--include='*.jsonl'` into `dotclaude/projects/`, and say plainly that the
  repo will grow by the current transcript size every snapshot. Check
  `du -sh ~/.claude/projects` first and quote the number.
- **Another directory under `~/.claude`.** Add it to the `for d in skills
  agents commands docs` loop in the script. Ask whether it is config or cache
  first; only config belongs in the mirror.
- **A second dotfile.** Copy it into `dotfiles/` in the staging step, next to
  the shell rc line.
- **Cadence.** Change the scheduler definition and re-run the installer. The
  script itself is safe to run as often as wanted.
