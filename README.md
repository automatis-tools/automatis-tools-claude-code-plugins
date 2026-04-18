# Automatis Tools - Claude Code Marketplace

A Claude Code plugin marketplace for the Automatis team. Ships a single plugin, `automatis`, which bundles the team's commands under one namespace.

## Quick Start

```bash
# Add the marketplace
/plugin marketplace add automatis-tools/claude-code-plugins

# Install the plugin (one install covers every command)
/plugin install automatis@automatis-tools
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/automatis:fix-pr` | Fix GitHub PR review comments |
| `/automatis:ports-release` | Release port conflicts on macOS |

---

### /automatis:fix-pr

Fetch PR review comments from GitHub and fix identified issues in the codebase.

```bash
/automatis:fix-pr https://github.com/owner/repo/pull/123
/automatis:fix-pr 123                    # uses current repo
/automatis:fix-pr                        # interactive mode
```

**Features:**
- Fetches all review comments with pagination
- Filters to OPEN comments only (no author reply)
- Groups issues by file and severity
- Fixes issues one by one, replies on the PR before pushing

---

### /automatis:ports-release

Detect, diagnose, and release port conflicts on macOS.

```bash
/automatis:ports-release 8000           # release single port
/automatis:ports-release 8000 8001      # release multiple ports
/automatis:ports-release                # interactive mode
```

**Features:**
- Identifies process using a port
- Determines if process belongs to current project
- Auto-kills project processes, asks before killing others

---

## Team Setup

Add to your project's `.claude/settings.json` to auto-prompt teammates:

```json
{
  "extraKnownMarketplaces": {
    "automatis-tools": {
      "source": {
        "source": "github",
        "repo": "automatis-tools/claude-code-plugins"
      }
    }
  }
}
```

## Contributing

Most contributions add a new command to the `automatis` plugin:

1. Create `automatis/commands/<command-name>.md` following the house style documented in [CLAUDE.md](CLAUDE.md#command-file-structure).
2. No manifest changes needed — commands are auto-discovered.

See [CLAUDE.md](CLAUDE.md) for conventions, shell-safety rules, and verification steps.

## License

MIT
