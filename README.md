# claude-code-orchestration

A turnkey replication kit for a cost-aware multi-agent orchestrator setup on Claude Code. This repo contains the agent definitions, plugin configuration, hook script, and CLAUDE.md template needed to run a hub-and-spoke orchestration workflow where an expensive main model handles planning and judgment while cheaper sub-agents do the actual file reads, code edits, and test runs. The result is significantly lower token costs on long sessions without sacrificing quality on decisions that actually need it.

## What's in here

- `SETUP.md` — full replication guide (read this first)
- `agents/` — the 7 agent definitions used by the orchestrator
- `hooks/` — session-start hook that heals the context-mode FTS5 index
- `settings.json.template` — drop in at `~/.claude/settings.json` and edit paths
- `CLAUDE.md.template` — drop in at `~/.claude/CLAUDE.md` and customize the lower half

## Quick start

1. Read `SETUP.md`.
2. Copy `settings.json.template` → `~/.claude/settings.json` and edit the hook path.
3. Copy `CLAUDE.md.template` → `~/.claude/CLAUDE.md` and fill in the TODOs.
4. Copy `agents/*.md` → `~/.claude/agents/`.
5. Copy `hooks/context-mode-cache-heal.mjs` → `~/.claude/hooks/` (or skip — context-mode plugin ships this; verify before copying).
6. Add the external marketplaces and install all plugins:

   ```bash
   # Add external marketplaces
   /plugin marketplace add mksglu/context-mode
   /plugin marketplace add thedotmack/claude-mem

   # Install plugins
   /plugin install context-mode@context-mode
   /plugin install claude-mem@thedotmack
   /plugin install superpowers@claude-plugins-official
   /plugin install context7@claude-plugins-official
   /plugin install skill-creator@claude-plugins-official
   /plugin install frontend-design@claude-plugins-official
   /plugin install typescript-lsp@claude-plugins-official
   /plugin install csharp-lsp@claude-plugins-official
   ```
7. Launch with `claude --agent orchestrator`.
   - Optional: see the `co` PowerShell shortcut in `SETUP.md` to run this from any directory by typing `co`.

## License

MIT — see [LICENSE](LICENSE)

Author: Eric Paul <ericallenpaul@hotmail.com>
