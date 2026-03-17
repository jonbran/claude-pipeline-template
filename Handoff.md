# Claude Code Project Handoff — Multi-Agent Development Pipeline

## Who You Are Talking To

This project was designed by **Jon** in a Claude.ai desktop session, then continued
in Claude Code. This document gives you full context so you can continue the work
without losing anything.

---

## What We Are Building

A **multi-agent software development pipeline** inside Claude Code, modelled after
a working setup Jon built in OpenCode. The goal is to automate the full development
lifecycle — from a Jira ticket all the way through to a Git push — using a team of
specialist sub-agents orchestrated by a primary Orchestrator agent.

---

## Background — The OpenCode Reference Implementation

Jon's OpenCode setup (the thing we are replicating) had the following agents,
each defined as a `.md` file with its own system prompt and tool permissions:

| Agent             | Role                                                                                                                                                      |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `orchestrator.md` | Primary agent. Manages the pipeline. Runs `/jira {ticket-id}` command to start work. Creates a new Git branch for every task.                             |
| `ba.md`           | Business Analyst. First agent to receive the Jira ticket. Turns it into a requirements file. Asks the user clarifying questions if anything is ambiguous. |
| `architect.md`    | Software Architect. Runs after BA. Reads requirements, designs architecture, creates a sequenced checkbox tasklist.                                       |
| `dev-frontend.md` | Frontend Developer. Implements UI/client-side tasks from the architect's list.                                                                            |
| `dev-backend.md`  | Backend Developer. Implements API/server-side tasks from the architect's list.                                                                            |
| `tester-unit.md`  | Unit Test Engineer. Writes and runs unit tests for all new/modified code.                                                                                 |
| `tester-ui.md`    | UI Test Engineer. Runs end-to-end browser tests using Playwright MCP.                                                                                     |

**Key pipeline behaviours from OpenCode:**

- BA and Architect run **sequentially** (BA must complete before Architect starts)
- Dev agents run **in parallel** after Architect completes
- Test agents run after dev agents complete
- If tests fail → bug log is created → work returns to dev agents → re-test (loop until pass)
- When all tests pass, Orchestrator asks the user: _"Ready to push to GitLab?"_
- Push only happens after explicit user confirmation

---

## What We Researched

We read the official Claude Code documentation at `code.claude.com/docs` and confirmed
every capability needed is natively supported. Key findings:

### Agent Files — `.claude/agents/`

Each agent is a `.md` file with **YAML frontmatter**. This is the direct equivalent
of OpenCode's agent `.md` files. Example structure:

```yaml
---
name: agent-name
description: >
  What this agent does and when to use it. Claude uses this to auto-delegate.
  Include "Use proactively" to encourage automatic delegation.
tools: Read, Write, Edit, Bash, Glob, Grep
disallowedTools: WebSearch
model: sonnet # sonnet | opus | haiku | inherit
permissionMode: acceptEdits # default | acceptEdits | dontAsk | bypassPermissions | plan
maxTurns: 20
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
memory: project
---
Your agent's system prompt goes here in Markdown...
```

### Slash Commands — `.claude/commands/`

The `/jira` command lives here as `jira.md`. Uses `$ARGUMENTS` to accept the
ticket ID. Can call the Jira REST API via Bash or a Jira MCP server.

The `/configure-pipeline` command also lives here. It auto-detects the project
stack and tunes all agent files in-place for the specific project.

### Skills — `.claude/skills/` (also works via `.claude/commands/`)

Reusable prompt-based capabilities. Note: as of Claude Code v2.1.x, custom
commands and skills are unified — a file in `.claude/commands/` works the same
as a skill.

### Orchestrator Tool Restrictions

The orchestrator uses `Agent(ba, architect, dev-frontend, dev-backend, code-reviewer, security-reviewer, tester-unit, tester-ui)`
in its `tools` field — this is an **allowlist** that restricts which sub-agents
it can spawn. `Agent` without parentheses allows spawning any sub-agent.

### Playwright MCP Scoping

The `tester-ui` agent has Playwright defined **inline** in its `mcpServers` field.
This means Playwright is only available to that agent — it spins up when the agent
starts and disconnects when it finishes. No other agent has access.

### Sequential vs Parallel Execution

- **Sequential** (BA → Architect, then Test after Dev): Orchestrator spawns each
  agent and waits for completion before moving to the next phase
- **Parallel** (Dev agents, Review agents): Orchestrator spawns agents as background
  agents simultaneously
- **Bug loop**: If testers return failures, orchestrator re-delegates to the relevant
  dev agent with the bug log, then re-runs the tester — loops until clean

### Important Version Note

As of **Claude Code v2.1.63**, the `Task` tool was renamed to `Agent`.
If you see older docs or examples using `Task(...)`, it is the same thing.
`Task(...)` still works as an alias.

---

## What Has Already Been Built

A **setup shell script** (`setup-claude-pipeline.sh`) was generated and tested.
It creates the entire project structure in one command:

```bash
chmod +x setup-claude-pipeline.sh
./setup-claude-pipeline.sh /path/to/your/project
```

### Files the Script Creates

```
your-project/
├── CLAUDE.md                              ← project-level pipeline rules
├── docs/                                  ← requirements & architecture files land here
└── .claude/
    ├── agents/
    │   ├── orchestrator.md                ← pipeline manager (primary agent)
    │   ├── ba.md                          ← business analyst
    │   ├── architect.md                   ← software architect
    │   ├── dev-frontend.md                ← frontend developer
    │   ├── dev-backend.md                 ← backend developer
    │   ├── code-reviewer.md               ← code quality reviewer (Phase 4.5)
    │   ├── security-reviewer.md           ← security vulnerability auditor (Phase 4.5)
    │   ├── tester-unit.md                 ← unit test engineer
    │   └── tester-ui.md                   ← UI test engineer (Playwright MCP)
    └── commands/
        ├── jira.md                        ← /jira <ticket-id> command
        └── configure-pipeline.md          ← /configure-pipeline command
```

---

## The Full Pipeline Flow (Step by Step)

```
User runs: /jira PROJ-123
           │
           ▼
    ┌─────────────┐
    │ orchestrator│  ← creates git branch: feature/PROJ-123
    └──────┬──────┘
           │ delegates
           ▼
    ┌─────────────┐
    │     ba      │  ← reads ticket, asks clarifying questions, writes docs/requirements.md
    └──────┬──────┘
           │ returns ✅
           ▼
    ┌─────────────┐
    │  architect  │  ← reads requirements, writes docs/architecture.md + checkbox tasklist
    └──────┬──────┘
           │ returns ✅
           ▼
    ┌──────┴──────────────────┐
    │                         │  ← parallel background agents
    ▼                         ▼
┌──────────────┐    ┌──────────────┐
│ dev-frontend │    │ dev-backend  │  ← both check off tasks in architecture.md as they go
└──────┬───────┘    └──────┬───────┘
       └────────┬───────────┘
                │ both complete ✅
                ▼
    ┌──────┴──────────────────┐
    │                         │  ← parallel background agents (Phase 4.5)
    ▼                         ▼
┌──────────────┐    ┌──────────────────┐
│ code-reviewer│    │security-reviewer │  ← report only, no file modifications
└──────┬───────┘    └──────┬───────────┘
       └────────┬───────────┘
                │ CRITICAL/HIGH issues → loop back to dev
                │ clean ✅
                ▼
    ┌─────────────────┐
    │  tester-unit    │  ← writes & runs unit tests, creates docs/bug-log.md if failures
    └──────┬──────────┘
           │ pass ✅ (or loop back to dev if fail)
           ▼
    ┌─────────────────┐
    │  tester-ui      │  ← runs Playwright e2e tests, appends to bug-log.md if failures
    └──────┬──────────┘
           │ pass ✅ (or loop back to dev if fail)
           ▼
    ┌─────────────┐
    │ orchestrator│  ← reports summary, asks user to confirm push
    └──────┬──────┘
           │ user confirms
           ▼
    git push origin feature/PROJ-123
```

### Review Agent Behaviour (Phase 4.5)

| Severity | Code Reviewer | Security Reviewer | Blocks testing? |
|---|---|---|---|
| CRITICAL | Bug / logic error / test will fail | — | Yes — dev agent must fix first |
| HIGH | — | OWASP vulnerability | Yes — dev agent must fix first |
| MEDIUM | — | Security concern | No — noted in summary |
| MINOR | Style / readability | — | No — noted in summary |
| LOW | — | Minor risk | No — noted in summary |

---

## The `/configure-pipeline` Command

After running the setup script on a new project, run `/configure-pipeline` once to
tune the agents for that project's specific stack.

**What it does:**
1. **Auto-detects** the stack by reading `package.json`, `requirements.txt`, `pom.xml`, etc.
2. Shows a detection table to the user
3. **Asks targeted questions** only for things it couldn't detect (all in one message)
4. **MCP server selection** — presents a numbered menu of 16 optional MCP servers grouped by
   agent. Servers are pre-marked ★ recommended based on the detected stack (e.g. GitHub remote
   detected → GitHub MCP recommended). User replies with numbers to enable. Then asks for
   credentials/config only for the selected servers (in a single follow-up message).
5. **Updates agent files in-place** — for each agent file:
   - Inserts a `## Project Context` section above `## Rules` with stack-specific instructions
   - Adds a `mcpServers:` block to the YAML frontmatter for any MCP servers the user selected
   - Agent files updated: `dev-frontend.md`, `dev-backend.md`, `tester-unit.md`,
     `tester-ui.md`, `orchestrator.md`, `ba.md`, `architect.md`, `code-reviewer.md`,
     `security-reviewer.md` (only files relevant to selected MCPs are touched)
6. **Updates `CLAUDE.md`** — prepends a `## Project` block with stack summary
7. **Creates `PIPELINE.config.md`** — human-readable config file the developer can edit
   and re-run the command against to push updates back into agents. Includes an
   `## MCP Servers Enabled` section listing what was configured.

**The command is idempotent** — running it multiple times is safe; it replaces the
`## Project Context` section rather than appending.

---

## Workflow for a New Project

```bash
# 1. Run the setup script
chmod +x setup-claude-pipeline.sh
./setup-claude-pipeline.sh /path/to/new-project

# 2. Open Claude Code
cd /path/to/new-project
claude

# 3. Tune the pipeline for this project (once)
/configure-pipeline

# 4. Start working
/jira PROJ-123
# or just describe a task and the orchestrator picks it up
```

---

## Jira Integration (Not Yet Configured)

The `/jira` command is built and ready. It uses the Jira REST API via `curl`.
Three environment variables need to be set before using it:

```bash
export JIRA_EMAIL="you@yourcompany.com"
export JIRA_API_TOKEN="your_token_here"   # generate at id.atlassian.com
export JIRA_DOMAIN="yourcompany.atlassian.net"
```

Add these to `~/.zshrc` or `~/.bash_profile` to make them permanent.

Alternatively, if a **Jira MCP server** is configured in the future, the
`/jira` command can be updated to use it instead of the REST API.

---

## Potential Next Steps / Things to Explore

- **Wire up Jira**: Set env vars (`JIRA_EMAIL`, `JIRA_API_TOKEN`, `JIRA_DOMAIN`) or use
  the Atlassian MCP server — now selectable via `/configure-pipeline`
- **Add a GitLab/GitHub MCP**: So the orchestrator can create PRs automatically after
  pushing — now selectable via `/configure-pipeline` (GitHub MCP option)
- **Add memory**: Set `memory: project` on key agents so they learn project conventions
  across sessions
- **Tune agent prompts further**: After first real runs, refine individual agent system
  prompts based on actual output quality — the `/configure-pipeline` command handles
  stack-specific tuning and MCP wiring; this is about prompt quality improvements

---

## Reference Links

- Official Claude Code sub-agents docs: https://code.claude.com/docs/en/sub-agents
- Official Claude Code skills/commands docs: https://code.claude.com/docs/en/slash-commands
- claudefa.st agent fundamentals guide: https://claudefa.st/blog/guide/agents/agent-fundamentals
- Jira API token management: https://id.atlassian.com/manage-profile/security/api-tokens
