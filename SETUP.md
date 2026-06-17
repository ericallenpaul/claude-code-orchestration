# Claude Code Orchestrator Setup

This is a cost-aware orchestrator workflow on Claude Code where the main (expensive) model handles judgment, routing, and integration while cheap or specialized sub-agents do the actual work. Four plugins reinforce this: **context-mode** keeps verbose tool output out of the conversation context window, **claude-mem** preserves observations across sessions, **superpowers** enforces process discipline through named skill checklists, and **context7** pulls live library documentation instead of relying on training-cutoff knowledge. The result is a setup that scales to multi-hour sessions without burning through token budgets on mechanical work.

---

## Why the Orchestrator Agent Exists

The core insight is that Opus/Sonnet token cost is dominated by *context volume*, not just output length. When the main model repeatedly reads files, runs greps, parses test output, and edits code, every one of those tool results lands in the context window and gets re-encoded on every subsequent call. A 5-file refactor can burn 50k tokens just on reading, then another 30k on editing — all at the highest per-token rate.

The orchestrator pattern offloads that volume to cheaper models. File reads, searches, log parsing, and test execution go to Haiku agents (roughly 1/10 the cost). Bounded multi-file edits go to Sonnet agents. The main model sees only structured summaries, diffs, and failure reports — the output of the workers — and spends its budget on the judgment calls that actually require it: deciding what to do, sequencing dependencies, reviewing integrated results, and handling ambiguous situations. The same 5-file refactor becomes ~80k Haiku tokens for reading + ~30k Sonnet tokens for editing + a few thousand main-model tokens for integration. The saving compounds across a long session.

Hub-and-spoke topology prevents agent sprawl: sub-agents report only to the orchestrator, never to each other. This keeps the communication graph simple (no emergent coordination bugs), enforces that only one model holds the task ledger, and makes it easy to audit what changed and why. The CLAUDE.md doctrine defines the dispatch tiers explicitly: Haiku for reads/searches/test runs, Sonnet for code generation/refactoring/multi-file edits, and Opus or the main model for architecture decisions, high-risk refactors, security review, and final integration.

---

## The Four Pillars

### Orchestrator Agents

Seven agent definition files live at `~/.claude/agents/`. Each is a Markdown file with a system prompt that scopes the agent's role, model tier, allowed tools, and write permissions.

| Agent | Tier | Role |
|---|---|---|
| `orchestrator` | main | Hub. Coordinates all others. No direct file edits. |
| `haiku-explorer` | Haiku | Read-only discovery, file inventory, log parsing |
| `haiku-test-runner` | Haiku | Bounded test execution, quick verification |
| `sonnet-implementer` | Sonnet | Multi-file implementation |
| `sonnet-code-reviewer` | Sonnet | Review and risk analysis |
| `sonnet-fix-agent` | Sonnet | Scoped follow-up fixes from review feedback |
| `tech-writer` | Sonnet | Documentation generation |

The orchestrator is launched via `claude --agent orchestrator`. The worker agents are dispatched by the orchestrator; you don't invoke them directly. All seven agent files ship in this repo's `agents/` directory — copy them straight into `~/.claude/agents/` rather than recreating them from this document.

### Superpowers

The `superpowers` plugin is a process skills library. It provides named skills — brainstorming, TDD, systematic-debugging, executing-plans, subagent-driven-development, writing-skills, writing-plans, verification-before-completion, requesting-code-review, receiving-code-review, using-git-worktrees, finishing-a-development-branch, dispatching-parallel-agents — activated via the `Skill` tool inside any agent.

The value is making implicit engineering habits explicit and repeatable. TDD as a checklist means you don't skip the failing-test step under pressure. `systematic-debugging` means you validate data assumptions before touching code. `subagent-driven-development` maps directly onto the named worker agents in this setup: the skill's workflow expects an explorer, test runner, implementer, reviewer, and fix agent — those are exactly the roles the agent files here fill. Superpowers is the process layer; the agent files are the execution layer.

### context-mode (ctx)

Context-mode's job is keeping raw tool output *out of the conversation context*. Every verbose shell command, every file listing, every test run that would otherwise dump hundreds of lines into the conversation thread is instead indexed into an FTS5 SQLite database running in a sandbox. You query it with `ctx_search` instead of re-reading the raw output.

The core tool is `ctx_batch_execute`: pass it a list of labeled commands, it runs them all, indexes the output, and returns a short summary. One call replaces many sequential Bash + Read cycles. `ctx_fetch_and_index` replaces `WebFetch` for the same reason — fetched content goes into the index, not the thread. The `context-mode-cache-heal.mjs` hook in `~/.claude/hooks/` runs at session start to heal the FTS5 index if it was left in a bad state. Slash commands for maintenance: `/ctx-stats`, `/ctx-doctor`, `/ctx-purge`, `/ctx-upgrade`, `/ctx-insight`.

### claude-mem

claude-mem provides cross-session memory that survives `/clear` and `/compact`. As you work, you capture observations; the plugin indexes them and auto-injects relevant prior context into future sessions starting on session #2. This is distinct from the curated `.claude/memory-bank/` markdown files that Claude writes deliberately — claude-mem is observation-indexed and automatic, memory-bank is curated and branch-aware.

Useful slash commands: `/learn-codebase` to do an initial scan and build the memory index for a new repo, `/make-plan` and `/do` for structured task execution, `/smart-explore` for targeted repo discovery, `/mem-search` for querying what's been remembered. The main gotcha: observations are persistent. If you teach it something wrong, you need to correct it explicitly — it won't forget on its own.

---

## Replication Steps

### 1. Install Claude Code

Follow the official docs: [https://docs.claude.com/en/docs/claude-code/overview](https://docs.claude.com/en/docs/claude-code/overview)

### 2. Create `~/.claude/settings.json`

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          { "type": "command", "command": "node \"<PATH_TO_HOME>/.claude/hooks/context-mode-cache-heal.mjs\"" }
        ]
      }
    ]
  },
  "enabledPlugins": {
    "skill-creator@claude-plugins-official": true,
    "superpowers@claude-plugins-official": true,
    "claude-mem@thedotmack": true,
    "frontend-design@claude-plugins-official": true,
    "context-mode@context-mode": true,
    "context7@claude-plugins-official": true,
    "typescript-lsp@claude-plugins-official": true,
    "csharp-lsp@claude-plugins-official": true
  },
  "extraKnownMarketplaces": {
    "context-mode": { "source": { "source": "github", "repo": "mksglu/context-mode" } },
    "thedotmack":   { "source": { "source": "github", "repo": "thedotmack/claude-mem" } }
  },
  "autoUpdatesChannel": "latest"
}
```

**Before copying verbatim:** the Node.exe path and hook path are platform-specific and user-specific. Edit them for your machine. On macOS/Linux the hook command would use the system `node` binary and a Unix path.

### 3. Add marketplaces and install plugins

The `claude-plugins-official` marketplace is built-in. Add the two external marketplaces first:

```bash
/plugin marketplace add mksglu/context-mode
/plugin marketplace add thedotmack/claude-mem
```

Now install the plugins. Run these in order so each source is grouped:

**context-mode** (https://context-mode.com/) — requires Node.js 18+:

```bash
/plugin install context-mode@context-mode
```

**claude-mem** (https://github.com/thedotmack/claude-mem):

```bash
/plugin install claude-mem@thedotmack
```

> **Warning:** `npm install -g claude-mem` installs the SDK/library only. It does NOT register the plugin hooks or set up the worker service. Use `/plugin install claude-mem@thedotmack` (above) or `npx claude-mem install` instead. Restart Claude Code after install. Context from previous sessions appears automatically starting on session #2.

**superpowers and the five `claude-plugins-official` plugins** (https://claude.com/plugins/superpowers — no `marketplace add` step needed for any of these):

```bash
/plugin install superpowers@claude-plugins-official
/plugin install context7@claude-plugins-official
/plugin install skill-creator@claude-plugins-official
/plugin install frontend-design@claude-plugins-official
/plugin install typescript-lsp@claude-plugins-official
/plugin install csharp-lsp@claude-plugins-official
```

### 4. Copy the agent files

Create the directory if it doesn't exist:

```bash
mkdir -p ~/.claude/agents
```

The seven agent `.md` files ship in this repo's `agents/` directory. Copy them directly:

```
orchestrator.md
haiku-explorer.md
haiku-test-runner.md
sonnet-implementer.md
sonnet-code-reviewer.md
sonnet-fix-agent.md
tech-writer.md
```

Their contents are not reproduced in this document. Open them directly from the repo's `agents/` directory if you want to inspect or tweak before copying.

### 5. Set up the context-mode hook

The `context-mode-cache-heal.mjs` hook ships with the context-mode plugin installation. Verify it was placed at `~/.claude/hooks/context-mode-cache-heal.mjs` after install; if not, locate it in the plugin's installed files and copy it there. Then confirm the path in your `settings.json` SessionStart hook matches where it actually landed.

### 6. Write your own CLAUDE.md

Copy `CLAUDE.md.template` to `~/.claude/CLAUDE.md`. This file is the behavioral contract for every Claude Code session. The template has two halves:

1. **Orchestrator doctrine** (top half) — the Cost-Aware Orchestrator Mode section is generic and ships verbatim. It defines dispatch rules by model tier, hub-and-spoke topology, pattern selection, and cost/safety guidelines. Keep it as-is unless you want to change the doctrine itself.
2. **Your customizations** (bottom half) — three TODO placeholder sections to fill in:
   - `## About You` — who you are, how you like to work, what you don't want
   - `## Tech Stack Preferences` — languages, frameworks, libs, infra you favor
   - `## Working Conventions` — branching, testing, deployment, schema-checking habits

Optional additions not in the template:
- A security policy section if you have a security MCP (Snyk, Semgrep, etc.) configured — instruct Claude to run scans on new code.
- A memory-bank or project-state convention if you want Claude to persist plans/reviews under a known path.

### 7. (Optional) Default to orchestrator mode

To launch Claude Code in orchestrator mode by default rather than having to specify `--agent orchestrator` each time, add a shell alias:

```bash
alias claude='claude --agent orchestrator'
```

Or set it in your shell profile. Note this makes sense only if you want the orchestrator behavior globally — for quick one-off questions it adds unnecessary overhead.

---

## Customization Notes

- The seven agents in this repo cover the core roles: exploration, testing, implementation, review, fixes, and documentation. If you have environment-specific deployment pipelines (e.g., a CI system or a release orchestrator), the existing agents are good patterns to follow when writing your own deployment agents.
- The Snyk section in CLAUDE.md assumes Snyk MCP is configured and the `snyk_code_scan` tool is available. Remove it if you're not using Snyk.
- The memory-bank path convention (`.claude/memory-bank/[branch-name]/{plans,reviews,sessions}`) is opinionated. Change it to whatever structure you prefer — it's just a CLAUDE.md instruction, not a plugin convention.
- The `ecc` marketplace is optional; add it or leave it out as you prefer.

---

## Optional: Add a `co` PowerShell Shortcut

Goal: type `co` from any directory instead of `claude --agent orchestrator`.

**1. Create the profile file if it doesn't exist yet:**

```powershell
New-Item -ItemType File -Path $PROFILE -Force
```

**2. Open it:**

```powershell
notepad $PROFILE
```

**3. Add the function** (the `@args` passthrough keeps flags like `--resume` working):

```powershell
function co {
    claude --agent orchestrator --model claude-opus-4-7 @args
}
```

> **Why pin the model?** The bare `opus` alias resolves to the latest Opus, which is currently Opus 4.8 — a quality/cost regression. The agent frontmatter `model:` does not reliably override the main session model, so pin `claude-opus-4-7` explicitly on the launch command.

Save and close.

**4. Reload the profile** (note the space after the leading dot):

```powershell
. $PROFILE
```

**5. Verify** by running `co` — it should launch `claude --agent orchestrator` in the current directory.

Optional check that the function loaded:

```powershell
Get-Command co
```

Expected output:

```text
CommandType     Name
-----------     ----
Function        co
```

**Troubleshooting — execution policy error on profile load:**

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Then reload with `. $PROFILE`.

---

## Caveats and Things to Watch Out For

- **`autoUpdatesChannel: "latest"`** opts you into bleeding-edge Claude Code releases. Breakage is possible. Switch to `"stable"` if you need predictability.
- **claude-mem observations persist across `/clear` and `/compact`**. If the plugin captures a wrong assumption about your codebase, it will keep injecting it into future sessions until you explicitly correct or purge it.
- **The orchestrator pattern adds latency**. Hub-and-spoke means more round-trips: main dispatches, worker executes, main reviews, main decides. On a quick one-liner question this feels slow compared to just asking directly. The cost saving is real and meaningful for sessions over ~30 minutes; for short questions, skip `--agent orchestrator`.
- **context-mode's FTS5 index can drift**. That's why the SessionStart hook heals it. If you see `ctx_search` returning stale or missing results, run `/ctx-doctor` to diagnose and `/ctx-purge` to reset if needed.
- **Agent files are behavioral contracts, not code**. If a worker agent ignores a constraint in its system prompt, the fix is to tighten the prompt — not to add more orchestrator logic. Keep agent prompts narrow and explicit.
