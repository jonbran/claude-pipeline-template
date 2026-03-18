---
description: Capture learnings from the last pipeline run and propose agent definition improvements
allowed-tools: Read, Edit, Glob, Grep
---

Review the current conversation for learnings about how each agent performed during this pipeline run. Propose targeted improvements to agent definitions in `.claude/agents/`.

## Step 1: Reflect on Each Agent's Execution

For every agent that ran during this session, assess:

- Did it follow its defined process?
- Did it produce output in the expected format?
- Did it stay within its scope, or did it do work belonging to another agent?
- Did it hit its maxTurns limit?
- Did it handle ambiguity correctly (stop and ask vs. guess)?
- Were there instructions it consistently ignored?
- Did the orchestrator have trouble parsing its output?

## Step 2: Read Current Agent Definitions

```bash
find .claude/agents -name "*.md" 2>/dev/null
```

Read each agent definition file that was involved in this pipeline run.

## Step 3: Draft Improvements

For each issue found, draft a specific, minimal change. Keep changes targeted — one issue per diff.

Format: action verb + specific instruction

Avoid:
- Rewriting agents that performed well
- Adding generic advice the model already knows
- Project-specific knowledge that belongs in CLAUDE.md
- Over-specifying behavior the model handles naturally

## Step 4: Show Proposed Changes

For each change:

```
### Update: .claude/agents/<agent-name>.md

**Observation:** <what happened during the run>
**Why this change helps:** <how it prevents the issue>

```diff
 <context line>
+<added line>
-<removed line>
```
```

## Step 5: Apply with Approval

Ask if the user wants to apply the changes. Only edit files they approve.

Present changes grouped by agent, so the user can approve/reject per agent.
