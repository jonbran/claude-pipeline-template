# Plugin Review: claude-md-management

Source: https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-md-management
Reviewed: 2026-03-18
Author: Isabella He (Anthropic)

---

## Purpose

Two complementary tools for keeping CLAUDE.md files accurate and useful:
1. **claude-md-improver** (Skill) — audits CLAUDE.md quality against actual codebase state
2. **/revise-claude-md** (Command) — captures learnings from the current session into CLAUDE.md

---

## Plugin Structure

```
claude-md-management/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── revise-claude-md.md
└── skills/
    └── claude-md-improver/
        ├── SKILL.md
        └── references/
            ├── quality-criteria.md
            ├── templates.md
            └── update-guidelines.md
```

**Key pattern**: The skill has a `references/` subfolder with supporting documents (criteria, templates, guidelines). The SKILL.md instructs Claude to consult these reference files during execution. This is a clean way to keep SKILL.md focused while providing rich supporting content.

---

## plugin.json

```json
{
  "name": "claude-md-management",
  "description": "Tools to maintain and improve CLAUDE.md files - audit quality, capture session learnings, and keep project memory current.",
  "version": "1.0.0",
  "author": {
    "name": "Anthropic",
    "email": "support@anthropic.com"
  }
}
```

---

## Skill: claude-md-improver

Invoked by Claude automatically based on context (e.g., after codebase changes, for periodic maintenance).

### Five operational phases:
1. **Discovery** — finds all CLAUDE.md variants: `./CLAUDE.md`, `./.claude.local.md`, `~/.claude/CLAUDE.md`, `./packages/*/CLAUDE.md`, nested dirs
2. **Assessment** — scores against 6 quality criteria (see below)
3. **Report generation** — grades A–F, shows issues per criterion
4. **Improvement proposals** — shows diffs in format: file, why, diff block
5. **Apply with approval** — only edits files the user approves

### Quality criteria (100 points total):
| Criterion | Points | Notes |
|:----------|:-------|:------|
| Commands/Workflows | 20 | Essential commands with context |
| Architecture Clarity | 20 | Key dirs, module relationships, entry points |
| Non-Obvious Patterns | 15 | Gotchas, workarounds, edge cases |
| Conciseness | 15 | No filler, every line adds value |
| Currency/Freshness | 15 | Commands work, file refs accurate |
| Actionability | 15 | Copy-paste ready, concrete steps |

### Key principles from SKILL.md:
- Minimal, project-specific updates only
- Copy-paste-ready commands
- Flag stale or missing content
- Pressing `#` during a session auto-incorporates learnings

### Red flags to detect:
- Commands that would fail
- References to deleted files
- Outdated tech versions
- Copy-paste templates not customized
- Generic advice
- Incomplete TODOs
- Duplicated info across CLAUDE.md files

---

## Command: /revise-claude-md

```markdown
---
description: Update CLAUDE.md with learnings from this session
allowed-tools: Read, Edit, Glob
---
```

Restricts to just `Read`, `Edit`, `Glob` — scoped permissions.

### 5-step workflow:
1. **Reflect** — what context was missing? (commands, style patterns, testing, env quirks, gotchas)
2. **Find CLAUDE.md files** — `find . -name "CLAUDE.md" -o -name ".claude.local.md"`
3. **Draft additions** — one line per concept; format: `<command>` - `<brief description>`
4. **Show proposed changes** — shows reason + diff for each addition
5. **Apply with approval** — ask user before editing

### What to add vs avoid:
| Add | Avoid |
|:----|:------|
| Discovered commands/workflows | Obvious code info ("UserService handles users") |
| Gotchas and non-obvious patterns | Generic best practices |
| Package relationships | One-off bug fixes |
| Testing approaches that worked | Verbose explanations |
| Configuration quirks | |

---

## Design Patterns to Reuse

1. **references/ subfolder** — skill has supporting .md files (criteria, templates, guidelines) that Claude reads during execution
2. **Diff proposal format** — show file, reason, diff — then ask approval before applying
3. **Scoped tool permissions** — command uses `allowed-tools` to restrict to only what's needed
4. **Dual approach** — one auto-invoked skill (broad, proactive) + one manual command (targeted, on-demand)
5. **Phased execution** — discovery → assessment → report → proposals → apply
6. **Graded output** — A–F scoring gives user a clear quality signal

---

## Relevance to Our Agent-Improvement Plugin

Our plugin should improve agents when they run. Patterns to borrow:
- Reference files in the skill folder to keep SKILL.md lean
- Phased execution: assess → report → propose → apply with approval
- Scoped `allowed-tools` in commands
- Quality criteria / rubric approach for scoring agent files
- Diff-based proposals with explanations before applying
