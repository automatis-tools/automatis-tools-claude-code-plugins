# CLAUDE.md

Guidance for Claude Code when working with this repository.

## Overview

This is a **Claude Code marketplace** for the Automatis team. Hosts multiple plugins installable via `/plugin install <name>@automatis-tools`.

## Structure

```
.claude-plugin/
└── marketplace.json              # Plugin catalog

plugins/
├── automatis-fix-pr/
│   ├── .claude-plugin/plugin.json
│   └── commands/fix.md           # → /automatis-fix-pr:fix
└── automatis-free-ports/
    ├── .claude-plugin/plugin.json
    └── commands/free.md          # → /automatis-free-ports:free
```

## Naming Convention

- **Plugins**: `automatis-<action>` (e.g., `automatis-fix-pr`, `automatis-free-ports`)
- **Commands**: Short action verb (e.g., `fix.md`, `free.md`, `check.md`)
- **Usage**: `/automatis-<plugin>:<command>` (e.g., `/automatis-fix-pr:fix`)

## Adding a Plugin

1. Create directory:
   ```
   plugins/automatis-<name>/
   ├── .claude-plugin/plugin.json
   └── commands/<action>.md
   ```

2. Plugin manifest (`.claude-plugin/plugin.json`):
   ```json
   {
     "name": "automatis-<name>",
     "version": "1.0.0",
     "description": "What it does",
     "author": { "name": "Automatis Tools" },
     "keywords": ["relevant", "tags"]
   }
   ```

3. Register in `.claude-plugin/marketplace.json`:
   ```json
   {
     "name": "automatis-<name>",
     "source": "./plugins/automatis-<name>",
     "description": "What it does",
     "category": "category",
     "tags": ["tags"]
   }
   ```

## Plugin Capabilities

Each plugin can include:
- `commands/` - Slash commands (markdown)
- `agents/` - Custom subagents (markdown)
- `skills/` - Agent skills with `SKILL.md`
- `hooks/` - Event hooks in `hooks.json`
- `.mcp.json` - MCP server configs
- `.lsp.json` - LSP server configs

## Testing

```bash
# Test single plugin
claude --plugin-dir ./plugins/automatis-<name>

# Validate marketplace
claude plugin validate .
```
