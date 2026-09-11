# skills

Claude Code skills by [@finnigan-j](https://github.com/finnigan-j). Install the set
as a plugin, or copy a single skill into `~/.claude/skills/`.

| Skill | What it does |
|---|---|
| [`claude-backup`](skills/claude-backup/SKILL.md) | Unattended weekly backup of Claude Code memories, skills, `CLAUDE.md` files and settings into a private git repo, with restore on a new machine. macOS (launchd) and Linux (systemd). |

## Install

### Claude Code

This repo is a plugin marketplace. In Claude Code:

```
/plugin marketplace add finnigan-j/fj-skills
/plugin install fj-skills@finnigan-j-skills
```

Skills load as `/fj-skills:<skill>` (for example `/fj-skills:claude-backup`) and
Claude also picks them up on its own when a request matches. Update later
with `/plugin marketplace update finnigan-j-skills`.

### Any agent that speaks the Agent Skills standard

```
npx skills add https://github.com/finnigan-j/fj-skills
```

### By hand

```sh
git clone https://github.com/finnigan-j/fj-skills.git
cp -r fj-skills/skills/claude-backup ~/.claude/skills/
```

## claude-backup

Claude Code keeps its most valuable state in `~/.claude`: per-project
memories, skills, `CLAUDE.md` files, settings. Nothing backs it up. This skill
walks Claude through setting up a small bash script and a scheduler job that
mirror the durable parts into a private git repo every week, proves it with a
25-check smoke test before the first real run, greps for secrets, installs the
schedule, and writes a README with restore instructions into the backup repo.

What gets mirrored is an allowlist: memories, skills, agents, commands, docs,
`CLAUDE.md`, settings, keybindings, plugin manifests, each project's `.claude/`
and `CLAUDE.md`, and the shell rc. Transcripts, caches, worktrees and
`~/.claude.json` (OAuth) stay out. The full tables with reasons are in
[`skills/claude-backup/references/scope.md`](skills/claude-backup/references/scope.md).

Try it from inside an empty **private** repo:

> Back up my Claude Code memories and settings to this private repo every week so I don't lose them if my laptop dies.

The scripts inside the skill are the same ones the author runs; they have
been backing up a real machine weekly since August 2026. The Linux
(`systemd`) installer was authored on macOS and is syntax-checked only;
reports and fixes welcome.

## Contributing

Issues and pull requests are welcome. Each skill is a directory under
`skills/` with a `SKILL.md` and, where useful, `scripts/`, `references/` and
`assets/`. Run `claude plugin validate .` before opening a PR.

## License

MIT. See [LICENSE](LICENSE).
