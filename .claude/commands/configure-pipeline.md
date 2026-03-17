---
name: configure-pipeline
description: Configure the Claude Code pipeline agents for this specific project. Auto-detects the tech stack and asks targeted questions to tune all agent files in-place.
allowed-tools: Read, Write, Edit, Bash, Glob
---

# Configure Pipeline for This Project

You are configuring the Claude Code multi-agent pipeline for this specific project.
Your goal is to update the agent files so they understand the project's tech stack,
conventions, and tooling — without the developer having to edit agent prompts manually.

Work through the steps below in order.

---

## Step 1 — Auto-detect the Stack

Run the following detection checks using Bash and Read. Collect all findings before
asking the user anything.

### Detect project type
```bash
ls -1 package.json requirements.txt pyproject.toml setup.py pom.xml build.gradle Cargo.toml go.mod composer.json Gemfile *.csproj 2>/dev/null
```

### If package.json exists, read it
Read `package.json` and extract:
- `dependencies` and `devDependencies` keys
- `scripts` (especially `dev`, `start`, `test`, `build`)

### If requirements.txt / pyproject.toml exists, read it
Read the file to identify Python frameworks (Django, Flask, FastAPI, etc.)

### Detect existing test config files
```bash
ls -1 jest.config.* vitest.config.* pytest.ini setup.cfg .mocharc.* karma.conf.* 2>/dev/null
```

### Detect existing Playwright / Cypress config
```bash
ls -1 playwright.config.* cypress.config.* 2>/dev/null
```

### Detect git remote
```bash
git remote -v 2>/dev/null | head -4
```

### Detect existing branch naming from recent git log
```bash
git branch --all 2>/dev/null | head -10
```

---

## Step 2 — Synthesise Your Findings

From what you detected, build a draft profile:

| Property | Detected Value |
|---|---|
| Frontend framework | e.g. React 18 / Vue 3 / Angular / None detected |
| Styling | e.g. Tailwind CSS / CSS Modules / styled-components / None detected |
| State management | e.g. Redux Toolkit / Zustand / Pinia / None detected |
| Backend language | e.g. Node.js / Python / Java / .NET / Go / None detected |
| Backend framework | e.g. Express / NestJS / FastAPI / Django / Spring Boot / None detected |
| ORM / DB layer | e.g. Prisma / TypeORM / SQLAlchemy / None detected |
| Unit test framework | e.g. Jest / Vitest / pytest / JUnit / None detected |
| Test run command | e.g. `npm test` / `npx vitest` / `pytest` / None detected |
| App start command | e.g. `npm run dev` / `python manage.py runserver` / None detected |
| Git remote | e.g. GitHub / GitLab / None detected |
| Branch format | e.g. `feature/PROJ-123` / `feat/slug` / Not clear |

Present this table to the user so they can see what was detected.

---

## Step 3 — Ask Targeted Questions

Ask only about things that are missing, ambiguous, or need a preference decision.
Ask all questions at once in a single message. Do not ask about things you already
detected with confidence.

Use this list to determine what to ask:

- **Frontend framework**: Ask if not detected, or if multiple candidates were found
- **Styling approach**: Ask if not detected
- **State management**: Ask if not detected (acceptable answer: "none / component state only")
- **Backend language + framework**: Ask if not detected
- **ORM / DB layer**: Ask if not detected (acceptable answer: "none / raw SQL / no database")
- **Unit test framework + run command**: Ask if not detected
- **App dev server start command**: Ask if not detected (needed for UI tests)
- **App base URL**: Ask what port/URL the dev server runs on (e.g. `http://localhost:3000`)
- **Git remote type**: Ask if not detected (GitHub / GitLab / Bitbucket / other)
- **Branch naming convention**: Ask for their preferred format, e.g.:
  - `feature/TICKET-123-short-description`
  - `feat/short-description`
  - `TICKET-123/description`
- **Any project-specific conventions**: Ask if there are coding standards, linting rules,
  or team conventions the agents should be aware of (acceptable answer: "none")

Wait for the user to respond before proceeding to Step 4.

---

## Step 4 — MCP Server Selection

Present the user with a curated menu of optional MCP servers that enhance the pipeline agents.
Before showing the menu, use the detection results from Step 1 to pre-mark servers as
**recommended** where the stack makes them an obvious fit (e.g. GitHub remote detected →
mark GitHub MCP as recommended; PostgreSQL dependency detected → mark DBHub as recommended).

Show the menu in a single message, grouped by agent. Use this exact format:

---

**Optional MCP Servers — reply with the numbers you want to enable (e.g. `1 3 7`) or `none`**

> Servers marked ★ are recommended based on your detected stack.

**Orchestrator**
1. GitHub MCP ★ (if GitHub remote detected) — create branches, open PRs, monitor CI runs
2. Slack MCP — post pipeline status notifications to a team channel
3. Memory MCP — persist pipeline state across session boundaries

**BA**
4. Atlassian/Jira MCP (Cloud) — read Jira tickets and Confluence pages natively
5. mcp-atlassian (Server/Data Center) — same as above but for self-hosted Jira/Confluence

**Architect & Dev Agents**
6. Context7 MCP — pulls live, version-specific library docs into context (prevents hallucinated APIs)
7. Sourcegraph MCP — semantic search across the codebase to find existing patterns

**Dev — Frontend**
8. Figma MCP — read Figma frames and design tokens to generate pixel-accurate code
9. ESLint MCP — run ESLint directly from the agent; surface and auto-fix linting errors
10. Browser MCP — preview the running app and click through flows during development

**Dev — Backend**
11. DBHub MCP ★ (if a database dependency is detected) — inspect live DB schemas, run queries
12. PostgreSQL MCP ★ (if PostgreSQL detected) — official read-only Postgres schema inspection

**Code Reviewer**
13. ESLint MCP (if not already selected above) — run linting on changed files during review

**Security Reviewer**
14. Semgrep MCP — SAST scanning with 5,000+ rules covering OWASP Top 10, injection, secrets

**Tester — Unit**
15. Context7 MCP (if not already selected above) — current docs for Jest, Vitest, pytest, etc.

**Tester — UI**
16. ExecuteAutomation Playwright MCP — extended Playwright with API testing tools (alternative
    to the built-in Playwright MCP already configured on tester-ui)

---

Wait for the user's selection before continuing.

Once the user replies, ask — in a **single follow-up message** — only for credentials or config
values required by the servers they selected. Use this reference:

| MCP Server | Required config |
|---|---|
| GitHub MCP | GitHub Personal Access Token (PAT) with `repo` scope |
| Slack MCP | Slack Bot OAuth Token (`xoxb-...`) and target channel name |
| Atlassian/Jira MCP | Jira domain (e.g. `yourco.atlassian.net`), email, API token |
| mcp-atlassian | Same as above, plus whether it's Cloud or Server/DC |
| Figma MCP | Figma Personal Access Token |
| Sourcegraph MCP | Sourcegraph instance URL and access token (or `sourcegraph.com` if using cloud) |
| DBHub MCP | DB connection string (e.g. `postgresql://user:pass@localhost:5432/mydb`) |
| PostgreSQL MCP | Postgres connection string |
| Semgrep MCP | Semgrep API token (optional — works without one but unlocks Pro rules) |
| Context7 MCP | None required |
| Browser MCP | None required |
| ESLint MCP | None required |
| Memory MCP | None required |
| ExecuteAutomation Playwright MCP | None required |

Wait for the user's credential responses before proceeding to Step 5.

---

## Step 5 — Update Agent Files

Using the detected and confirmed information, update each agent file listed below.
For each file, locate the `## Rules` section and insert a `## Project Context` section
**directly above it**. If a `## Project Context` section already exists in the file,
replace it entirely.

The `## Project Context` section should be written as clear, direct instructions
that the agent will act on every time it runs.

### Update `.claude/agents/dev-frontend.md`

Add a `## Project Context` section covering:
- Framework and version (if applicable)
- Styling approach and any utility classes to use
- State management library (or note that component state only is used)
- Any component conventions (e.g. always use functional components, named exports only)
- File and folder naming conventions if known

Example shape:
```
## Project Context
- Framework: React 18 with TypeScript
- Styling: Tailwind CSS — use utility classes, avoid inline styles
- State: Zustand for global state, React useState for local state
- Components: functional only, named exports, one component per file
- File naming: PascalCase for components, camelCase for hooks and utils
```

### Update `.claude/agents/dev-backend.md`

Add a `## Project Context` section covering:
- Language and version
- Framework
- ORM / DB layer and how to define models/queries
- API style (REST / GraphQL / tRPC)
- Auth approach if known (JWT, session, OAuth)
- Any backend conventions

Example shape:
```
## Project Context
- Language: Python 3.11
- Framework: FastAPI
- ORM: SQLAlchemy with Alembic migrations
- API style: REST — JSON responses, snake_case field names
- Auth: JWT via python-jose, tokens passed as Bearer headers
```

### Update `.claude/agents/tester-unit.md`

Add a `## Project Context` section covering:
- Test framework name and config file location
- Exact command to run tests (e.g. `npm test`, `npx vitest run`, `pytest -v`)
- Test file naming convention (e.g. `*.test.ts`, `*_test.py`, `*.spec.ts`)
- Where test files live relative to source files (co-located vs `__tests__` folder vs `tests/` dir)

Example shape:
```
## Project Context
- Test framework: Vitest
- Run command: `npx vitest run`
- Test file naming: `*.test.ts` co-located next to the source file
- Config: vitest.config.ts at project root
```

### Update `.claude/agents/tester-ui.md`

Add a `## Project Context` section covering:
- Command to start the dev server (e.g. `npm run dev`)
- Base URL for tests (e.g. `http://localhost:3000`)
- Any known auth flow the tester may need to bypass or set up (e.g. seed a test user)

Example shape:
```
## Project Context
- Start command: `npm run dev`
- Base URL: http://localhost:5173
- Auth: tests should use test@example.com / password123 (seeded in dev DB)
```

### Update `.claude/agents/orchestrator.md`

Add a `## Project Context` section covering:
- Git remote type (GitHub / GitLab / etc.)
- Branch naming format with a concrete example
- Remote name to push to (usually `origin`)

Example shape:
```
## Project Context
- Git remote: GitLab
- Branch format: feature/PROJ-123-short-description
- Push remote: origin
```

### Add MCP Servers to Agent Frontmatter

For each MCP server the user selected in Step 4, add it to the `mcpServers:` block in the
YAML frontmatter of the relevant agent file(s). If a `mcpServers:` key already exists,
merge the new entries in; do not overwrite existing entries.

The target agent for each server is:

| MCP Server | Agent file(s) to update |
|---|---|
| GitHub MCP | `orchestrator.md`, `code-reviewer.md` |
| Slack MCP | `orchestrator.md` |
| Memory MCP | `orchestrator.md` |
| Atlassian/Jira MCP or mcp-atlassian | `ba.md` |
| Context7 MCP | `architect.md`, `dev-frontend.md`, `dev-backend.md`, `tester-unit.md` (whichever apply) |
| Sourcegraph MCP | `architect.md`, `code-reviewer.md` |
| Figma MCP | `dev-frontend.md` |
| ESLint MCP | `dev-frontend.md`, `code-reviewer.md` (whichever apply) |
| Browser MCP | `dev-frontend.md` |
| DBHub MCP | `dev-backend.md`, `tester-unit.md` |
| PostgreSQL MCP | `dev-backend.md` |
| Semgrep MCP | `security-reviewer.md` |
| ExecuteAutomation Playwright MCP | `tester-ui.md` |

Use these YAML snippets for each server. Substitute any credential values the user provided.
Place each under a named key inside `mcpServers:`.

**GitHub MCP**
```yaml
mcpServers:
  github:
    type: stdio
    command: docker
    args: ["run", "-i", "--rm", "-e", "GITHUB_PERSONAL_ACCESS_TOKEN", "ghcr.io/github/github-mcp-server"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "<user-provided-token>"
```

**Slack MCP**
```yaml
mcpServers:
  slack:
    type: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-slack"]
    env:
      SLACK_BOT_TOKEN: "<user-provided-token>"
      SLACK_CHANNEL: "<user-provided-channel>"
```

**Memory MCP**
```yaml
mcpServers:
  memory:
    type: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-memory"]
```

**Atlassian/Jira MCP (Cloud)**
```yaml
mcpServers:
  atlassian:
    type: stdio
    command: npx
    args: ["-y", "@atlassian/mcp-server"]
    env:
      ATLASSIAN_DOMAIN: "<user-provided-domain>"
      ATLASSIAN_EMAIL: "<user-provided-email>"
      ATLASSIAN_API_TOKEN: "<user-provided-token>"
```

**mcp-atlassian (Server/Data Center)**
```yaml
mcpServers:
  atlassian:
    type: stdio
    command: uvx
    args: ["mcp-atlassian"]
    env:
      JIRA_URL: "<user-provided-url>"
      JIRA_USERNAME: "<user-provided-email>"
      JIRA_API_TOKEN: "<user-provided-token>"
```

**Context7 MCP**
```yaml
mcpServers:
  context7:
    type: stdio
    command: npx
    args: ["-y", "@upstash/context7-mcp"]
```

**Sourcegraph MCP**
```yaml
mcpServers:
  sourcegraph:
    type: stdio
    command: npx
    args: ["-y", "@sourcegraph/mcp-server"]
    env:
      SRC_ENDPOINT: "<user-provided-url>"
      SRC_ACCESS_TOKEN: "<user-provided-token>"
```

**Figma MCP**
```yaml
mcpServers:
  figma:
    type: stdio
    command: npx
    args: ["-y", "@figma/mcp-server"]
    env:
      FIGMA_ACCESS_TOKEN: "<user-provided-token>"
```

**ESLint MCP**
```yaml
mcpServers:
  eslint:
    type: stdio
    command: npx
    args: ["-y", "@eslint/mcp"]
```

**Browser MCP**
```yaml
mcpServers:
  browser:
    type: stdio
    command: npx
    args: ["-y", "@browsermcp/mcp"]
```

**DBHub MCP**
```yaml
mcpServers:
  dbhub:
    type: stdio
    command: npx
    args: ["-y", "@bytebase/dbhub"]
    env:
      DATABASE_URL: "<user-provided-connection-string>"
```

**PostgreSQL MCP**
```yaml
mcpServers:
  postgres:
    type: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-postgres", "<user-provided-connection-string>"]
```

**Semgrep MCP**
```yaml
mcpServers:
  semgrep:
    type: stdio
    command: uvx
    args: ["semgrep-mcp"]
    env:
      SEMGREP_APP_TOKEN: "<user-provided-token-or-omit-if-not-provided>"
```

**ExecuteAutomation Playwright MCP**
```yaml
mcpServers:
  playwright-extended:
    type: stdio
    command: npx
    args: ["-y", "@executeautomation/playwright-mcp-server"]
```

---

### Update `CLAUDE.md`

Prepend a `## Project` section at the top of the file (before the existing content)
covering:
- Project name
- One-line description of what it does
- Tech stack summary (frontend + backend + DB)
- Any team conventions mentioned by the user

Example shape:
```
## Project
**Name**: Acme Portal
**Description**: Internal HR portal for employee self-service
**Stack**: React 18 + TypeScript (frontend), FastAPI + PostgreSQL (backend)
**Conventions**: All PRs require passing CI; branch names must reference a Jira ticket ID
```

---

## Step 6 — Write PIPELINE.config.md

Create (or overwrite) a file called `PIPELINE.config.md` in the project root.
This is a human-readable reference that summarises everything configured.
It is not read by agents — it is for the developer to review and edit later.

```markdown
# Pipeline Configuration

_Generated by `/configure-pipeline`. Edit this file and re-run the command to update agent files._

## Tech Stack
- **Frontend**: ...
- **Styling**: ...
- **State management**: ...
- **Backend**: ...
- **ORM / DB**: ...

## Tooling
- **Unit test framework**: ...
- **Unit test run command**: ...
- **Test file convention**: ...
- **App start command**: ...
- **App base URL**: ...

## Git
- **Remote**: ... (GitHub / GitLab / etc.)
- **Branch naming**: ...
- **Push remote**: origin

## MCP Servers Enabled
<!-- List each enabled server, the agent(s) it was added to, and any non-secret config values -->
...

## Conventions
...
```

---

## Step 7 — Confirm to the User

Report what was updated:

```
✅ Pipeline configured for this project.

Updated agent files:
  • .claude/agents/dev-frontend.md
  • .claude/agents/dev-backend.md
  • .claude/agents/tester-unit.md
  • .claude/agents/tester-ui.md
  • .claude/agents/orchestrator.md
  • CLAUDE.md
  [plus any agent files updated with MCP server config]

MCP servers enabled:
  [list each server and the agent(s) it was added to, or "none selected"]

Created:
  • PIPELINE.config.md  ← edit this and re-run /configure-pipeline to update agents

The pipeline is ready. Run /jira <ticket-id> or describe a task to begin.
```

---

## Rules
- Never guess — only write confirmed information into agent files
- If the user answers "I don't know" or "not sure" for something, leave that property
  out of the agent's Project Context rather than writing a placeholder
- Always preserve the existing content of each agent file — only add or replace
  the `## Project Context` section
- Do not modify the YAML frontmatter of any agent file **except** to add or update the
  `mcpServers:` block as directed in Step 5
