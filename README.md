# Automatis Tools - Claude Code Marketplace

A Claude Code plugin marketplace for the Automatis team. Contains curated plugins to enhance your development workflow.

## Quick Start

```bash
# Add the marketplace
/plugin marketplace add automatis-tools/claude-code-plugins

# Install plugins you need
/plugin install automatis-fix-pr@automatis-tools
/plugin install automatis-free-ports@automatis-tools
```

## Available Plugins

| Plugin | Command | Description |
|--------|---------|-------------|
| `automatis-fix-pr` | `/automatis-fix-pr:fix` | Fix GitHub PR review comments |
| `automatis-free-ports` | `/automatis-free-ports:free` | Free port conflicts on macOS |

---

### automatis-fix-pr

Fetch PR review comments from GitHub and fix identified issues in the codebase.

```bash
/automatis-fix-pr:fix https://github.com/owner/repo/pull/123
/automatis-fix-pr:fix 123                    # uses current repo
/automatis-fix-pr:fix                        # interactive mode
```

**Features:**
- Fetches all review comments with pagination
- Filters to OPEN comments only (no author reply)
- Groups issues by file and severity
- Fixes issues one by one

---

### automatis-free-ports

Detect, diagnose, and free port conflicts on macOS.

```bash
/automatis-free-ports:free 8000           # free single port
/automatis-free-ports:free 8000 8001      # free multiple ports
/automatis-free-ports:free                # interactive mode
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

1. Create `plugins/automatis-<name>/`
2. Add `.claude-plugin/plugin.json`
3. Add commands in `commands/`
4. Register in `.claude-plugin/marketplace.json`

See [CLAUDE.md](CLAUDE.md) for detailed guidelines.

## License

MIT
