# Project Development Pipeline

This project uses a multi-agent development pipeline managed by the orchestrator agent.

## How to Start Work

Run `/jira <ticket-id>` to pull a Jira ticket and begin the pipeline automatically,
or describe a task directly and ask the orchestrator to manage the process.

## Pipeline Overview

1. **Orchestrator** — Manages the whole flow, creates a git branch, offers to push when done
2. **BA Agent** — Converts requirements into a structured spec file; asks questions if ambiguous
3. **Architect Agent** — Designs architecture and creates a sequenced tasklist with checkboxes
4. **Dev Agents** — Frontend and backend developers work in parallel from the tasklist
5. **Review Agents** — Code reviewer and security reviewer run in parallel after dev agents
6. **Test Agents** — Unit tester and UI tester (Playwright) validate all work
7. **Bug Loop** — If tests fail, work returns to dev agents until all tests pass (max 3 retries)

## Agent Delegation Rules

- Always delegate to the appropriate specialist — do not do their work in the main thread
- BA and Architect run **sequentially** (BA must complete before Architect starts)
- Dev agents run **in parallel** after Architect completes
- Review agents run **in parallel** after dev agents complete
- Test agents run **after** review agents complete (not directly after dev agents)
- If tests fail, re-delegate to the relevant dev agent with the bug log, then re-test
- Always ask the user before pushing to remote git
