# CLAUDE.md

Guidance for Claude Code when working with this repository.

## Overview

This is a **Claude Code marketplace** for the Automatis team. It hosts a single plugin, `automatis`, which bundles the team's commands under one namespace. Install with `/plugin install automatis@automatis-tools`; invoke with `/automatis:<command>`.

The marketplace schema supports multiple plugins, but this one deliberately ships only `automatis` so team members get every command from a single install.

## Structure

```
.claude-plugin/
└── marketplace.json              # Plugin catalog (one entry: automatis)

automatis/
├── .claude-plugin/plugin.json
└── commands/
    ├── fix-pr.md                 # → /automatis:fix-pr
    └── ports-release.md          # → /automatis:ports-release
```

## Naming Convention

- **Plugin**: always `automatis`. New commands go inside this plugin — do not create sibling plugin directories unless there's a strong reason.
- **Commands**: kebab-case action names as the markdown filename (`fix-pr.md`, `ports-release.md`, `check-deps.md`). Noun-verb or verb-noun, whichever reads better.
- **Usage**: `/automatis:<command>` (e.g., `/automatis:fix-pr`). The command name is exactly the filename without `.md`.

## Adding a Command (common case)

Most work is adding a new command to the existing plugin:

1. Create `automatis/commands/<command-name>.md` following the house style in [Command File Structure](#command-file-structure) below.
2. No manifest changes needed — Claude Code auto-discovers files in `commands/`.
3. Reload the plugin (`/plugin reload automatis@automatis-tools`) or restart Claude Code.

## Adding a Plugin (rare)

Only needed if a new tool genuinely belongs in its own namespace (distinct audience, separate install lifecycle, licensing boundary, etc.). Otherwise use "Adding a Command" above.

1. Create directory at repo root (**not** under `plugins/` — that path was removed in commit `24e6c10` to work around plugin discovery):
   ```
   <plugin-name>/
   ├── .claude-plugin/plugin.json
   └── commands/<action>.md
   ```

2. Plugin manifest (`.claude-plugin/plugin.json`):
   ```json
   {
     "name": "<plugin-name>",
     "version": "1.0.0",
     "description": "What it does",
     "author": { "name": "Automatis Tools" },
     "keywords": ["relevant", "tags"]
   }
   ```

3. Register in `.claude-plugin/marketplace.json`:
   ```json
   {
     "name": "<plugin-name>",
     "source": "./<plugin-name>",
     "description": "What it does",
     "category": "category",
     "tags": ["tags"]
   }
   ```
   Source paths are relative to the repo root and must use the `./<dir>` form — other formats have broken plugin discovery before (commits `978043a`, `9dd783e`, `7adaae1`, `ae84edf`, `24e6c10`).

## Plugin Capabilities

Each plugin can include:
- `commands/` - Slash commands (markdown)
- `agents/` - Custom subagents (markdown)
- `skills/` - Agent skills with `SKILL.md`
- `hooks/` - Event hooks in `hooks.json`
- `.mcp.json` - MCP server configs
- `.lsp.json` - LSP server configs

### Command File Structure

Every `commands/<action>.md` in this repo follows the same shape — keep new commands consistent so the plugins read as one family:

- **YAML frontmatter** (between `---` fences) with three fields: `description` (one-line blurb for the `/` menu), `argument-hint` (arg shape shown after the command name in autocomplete), `allowed-tools` (comma-separated whitelist — tighter is safer, e.g. `Bash` alone for commands that never edit files). Then a `# Title` line.
- `## When to Use` — 2–4 bullets describing trigger scenarios.
- `## Arguments` — every accepted input form shown as a concrete example line (`/<plugin>:<cmd> <positional>`, `/<plugin>:<cmd>` for interactive mode). Document optional `--flag=value` here.
- `## Procedure` — numbered steps (`### Step 1: …`), each containing the exact bash block Claude should run. Mark irreversible steps with `**CRITICAL**` so Claude treats them as blocking.
- `## Safety Rules` — numbered list of guardrails (what to refuse, what to confirm with the user).
- `## Example Session` (optional) — fenced block showing a real interaction.

Reference implementation: `automatis/commands/fix-pr.md`.

## Shell Safety

Plugin command files generate bash snippets that Claude executes. A few traps have bitten this repo before (see commits `bdad29e`, `aa4e7be`); codify them here so new plugins don't repeat them.

- **`!` in jq inside double-quoted bash breaks.** Bash history expansion corrupts `!` even inside `"..."`. Never write `jq '... != null ...'` in a bash block — use Python (`python3 -c`) for any filter that needs `!=`.
- **Don't chain `--argjson` with shell variables.** If the variable is empty or not valid JSON, `gh`/`jq` fail silently and the pipeline "succeeds" with wrong data. Prefer Python when you need to pass structured data.
- **Use Python for multi-step JSON filtering.** Single-field extraction with `--jq '.user.login'` is safe. Anything involving sets, joins, or author comparisons: switch to `python3 -c`.
- **Command files are executed as written.** Copy exact code blocks; do not let Claude "rewrite jq from memory" — that's how the above bugs entered.

These rules live here (not inside individual command files) because any new plugin calling `gh`, `jq`, or `curl | jq` will hit the same traps.

## Manual Verification

This repo has no build, lint, or CI pipeline — plugins are declarative markdown + JSON. To verify a change:

1. Install the marketplace locally and run the slash command against a real target (a PR number for `/automatis:fix-pr`, a port for `/automatis:ports-release`).
2. Confirm the plugin manifest parses:
   ```bash
   python3 -c "import json; json.load(open('automatis/.claude-plugin/plugin.json'))"
   ```
3. Confirm `marketplace.json` still parses after any edit:
   ```bash
   python3 -c "import json; json.load(open('.claude-plugin/marketplace.json'))"
   ```

If you later add tooling (lint, schema validation, CI), update this section.
