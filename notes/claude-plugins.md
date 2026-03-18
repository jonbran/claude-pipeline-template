# Claude Code Plugin System Notes

Source: https://code.claude.com/docs/en/plugins
Reference: https://code.claude.com/docs/en/plugins-reference
Date reviewed: 2026-03-18
Requires: Claude Code v1.0.33+

---

## What is a Plugin?

A **plugin** is a self-contained directory that extends Claude Code with custom functionality (skills, agents, hooks, MCP servers, LSP servers). Plugins can be shared across projects and teams via marketplaces.

**Plugins vs Standalone config (`.claude/` directory)**:
- Standalone: project-specific, short skill names like `/hello`, no sharing needed
- Plugin: shareable, namespaced skill names like `/my-plugin:hello`, versioned releases, marketplace distribution

---

## Directory Structure

```
my-plugin/
├── .claude-plugin/           # Metadata only — NO other dirs go here
│   └── plugin.json           # Manifest (optional if using defaults)
├── commands/                 # Legacy skill markdown files (use skills/ for new)
├── agents/                   # Custom subagent .md files
├── skills/                   # Agent skills — subfolders with SKILL.md
│   └── my-skill/
│       └── SKILL.md
├── hooks/
│   └── hooks.json
├── settings.json             # Default settings (only `agent` key supported)
├── .mcp.json                 # MCP server definitions
├── .lsp.json                 # LSP server configs
└── scripts/                  # Hook/utility scripts
```

**CRITICAL**: `commands/`, `agents/`, `skills/`, `hooks/` must be at the **plugin root**, NOT inside `.claude-plugin/`.

---

## Plugin Manifest (`plugin.json`)

Location: `.claude-plugin/plugin.json`

### Required fields (only `name` if manifest is present)
```json
{
  "name": "my-plugin"
}
```

### Full schema
```json
{
  "name": "plugin-name",           // required; used as skill namespace
  "version": "1.2.0",              // semantic versioning
  "description": "Brief description",
  "author": {
    "name": "Author Name",
    "email": "author@example.com",
    "url": "https://github.com/author"
  },
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/author/plugin",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "commands": ["./custom/commands/special.md"],
  "agents": "./custom/agents/",
  "skills": "./custom/skills/",
  "hooks": "./config/hooks.json",
  "mcpServers": "./mcp-config.json",
  "outputStyles": "./styles/",
  "lspServers": "./.lsp.json"
}
```

**Notes on component path fields**:
- Custom paths **supplement** default directories, not replace them
- All paths must be relative and start with `./`
- Can be string or array

---

## Skills

- Location: `skills/<skill-name>/SKILL.md`
- Skill name = folder name, prefixed with plugin name → `/my-plugin:skill-name`
- Use `$ARGUMENTS` placeholder to capture user input after skill name

### SKILL.md format
```markdown
---
description: What this skill does and when Claude should use it
disable-model-invocation: true   # optional — prevents Claude from invoking model
---

Instructions for Claude. Use "$ARGUMENTS" for user-provided input.
```

---

## Agents

- Location: `agents/<agent-name>.md`
- Appear in `/agents` interface
- Claude can auto-invoke based on context

### Agent file format
```markdown
---
name: agent-name
description: What this agent specializes in and when Claude should invoke it
---

Detailed system prompt for the agent.
```

---

## Hooks

Location: `hooks/hooks.json`

### Available events
- `PreToolUse` / `PostToolUse` / `PostToolUseFailure`
- `PermissionRequest`
- `UserPromptSubmit`
- `Notification`
- `Stop`
- `SubagentStart` / `SubagentStop`
- `SessionStart` / `SessionEnd`
- `TeammateIdle`
- `TaskCompleted`
- `PreCompact` / `PostCompact`

### Hook types
- `command` — shell command/script
- `prompt` — LLM prompt with `$ARGUMENTS`
- `agent` — agentic verifier with tools

### hooks.json format
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format-code.sh"
          }
        ]
      }
    ]
  }
}
```

---

## MCP Servers

Location: `.mcp.json` or inline in `plugin.json`

```json
{
  "mcpServers": {
    "my-server": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/my-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": {
        "DATA_PATH": "${CLAUDE_PLUGIN_DATA}/data"
      }
    }
  }
}
```

- Start automatically when plugin is enabled
- Appear as standard MCP tools in Claude's toolkit

---

## LSP Servers

Location: `.lsp.json` or inline in `plugin.json`

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

**Required fields**: `command`, `extensionToLanguage`
**Optional**: `args`, `transport`, `env`, `initializationOptions`, `settings`, `workspaceFolder`, `startupTimeout`, `shutdownTimeout`, `restartOnCrash`, `maxRestarts`

Note: The binary must be installed separately by the user.

---

## Environment Variables

Two special variables available in hook commands, skill/agent content, and MCP/LSP configs:

- **`${CLAUDE_PLUGIN_ROOT}`** — absolute path to plugin installation dir (changes on update)
- **`${CLAUDE_PLUGIN_DATA}`** — persistent dir that survives updates (`~/.claude/plugins/data/{id}/`)

Use `${CLAUDE_PLUGIN_DATA}` for: `node_modules`, venvs, caches, generated files.

### Pattern for installing deps on first run / on update
```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "diff -q \"${CLAUDE_PLUGIN_ROOT}/package.json\" \"${CLAUDE_PLUGIN_DATA}/package.json\" >/dev/null 2>&1 || (cd \"${CLAUDE_PLUGIN_DATA}\" && cp \"${CLAUDE_PLUGIN_ROOT}/package.json\" . && npm install) || rm -f \"${CLAUDE_PLUGIN_DATA}/package.json\""
      }]
    }]
  }
}
```

---

## Default Settings (`settings.json`)

```json
{
  "agent": "security-reviewer"
}
```

Only `agent` key is currently supported — activates a plugin agent as the main thread.

---

## Installation Scopes

| Scope     | Settings file                   | Use case                          |
|:----------|:--------------------------------|:----------------------------------|
| `user`    | `~/.claude/settings.json`       | Personal, all projects (default)  |
| `project` | `.claude/settings.json`         | Team-shared via version control   |
| `local`   | `.claude/settings.local.json`   | Project-specific, gitignored      |
| `managed` | Managed settings                | Read-only, managed by org         |

---

## CLI Commands

```bash
# Test locally during development
claude --plugin-dir ./my-plugin
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two  # multiple

# Install/manage
claude plugin install <plugin>[@marketplace] [--scope user|project|local]
claude plugin uninstall <plugin> [--scope user] [--keep-data]
claude plugin enable <plugin> [--scope user]
claude plugin disable <plugin> [--scope user]
claude plugin update <plugin> [--scope user]
claude plugin validate  # check plugin.json and frontmatter for errors
```

### In TUI
- `/reload-plugins` — reload without restarting
- `/plugin validate` — validate plugin structure
- `/debug` — show loading details

---

## Versioning

Semantic versioning: `MAJOR.MINOR.PATCH`
- Bump version in `plugin.json` before distributing — users won't get updates otherwise (caching)
- Start at `1.0.0` for first stable release
- Pre-release: `2.0.0-beta.1`

---

## Caching & Path Rules

- Marketplace plugins are copied to `~/.claude/plugins/cache/` — not used in-place
- Cannot reference files outside plugin dir (no `../` path traversal)
- Use symlinks if you need external files (they are copied during cache)

---

## Common Mistakes & Fixes

| Issue | Cause | Fix |
|:------|:------|:----|
| Plugin not loading | Invalid `plugin.json` | Run `claude plugin validate` |
| Commands not appearing | Wrong dir structure | Move dirs to plugin root, not inside `.claude-plugin/` |
| Hooks not firing | Script not executable | `chmod +x script.sh` |
| MCP server fails | Missing `${CLAUDE_PLUGIN_ROOT}` | Use variable for all plugin paths |
| Path errors | Absolute paths used | All paths relative, start with `./` |
| LSP `Executable not found` | Binary not installed | Install language server separately |

---

## Distribution

1. Add `README.md` with install/usage instructions
2. Use semantic versioning in `plugin.json`
3. Create or use a marketplace (see plugin-marketplaces docs)
4. Submit to official Anthropic marketplace:
   - Claude.ai: `claude.ai/settings/plugins/submit`
   - Console: `platform.claude.com/plugins/submit`

---

## Converting Standalone Config to Plugin

1. `mkdir -p my-plugin/.claude-plugin`
2. Create `plugin.json` with name/description/version
3. Copy: `cp -r .claude/commands my-plugin/` (and agents/, skills/)
4. Migrate hooks from `settings.json` → `hooks/hooks.json`
5. Test: `claude --plugin-dir ./my-plugin`
6. Remove originals from `.claude/` to avoid duplicates
