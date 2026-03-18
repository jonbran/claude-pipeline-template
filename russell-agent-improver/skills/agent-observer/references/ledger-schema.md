# Agent Observer — Ledger & Log Schema Reference

Use this reference during Steps 4 and 5 of the observation skill for exact file formats.

---

## Observation Log Format

**Path:** `.claude/agent-logs/<agent>/<task_id>.md`
**One file per task per agent.** If a file for this task_id already exists (e.g. agent was
re-delegated multiple times for the same task), append a new dated section rather than
overwriting.

```markdown
# Agent Observation: <agent> / <task_id>
Date: <YYYY-MM-DD>
Agent: <agent>
Task ID: <task_id>
Retries: <integer>

## What Was Requested
<One to three sentences describing what the orchestrator asked the agent to do.>

## What Was Returned
<One to three sentences summarizing what the agent produced or reported back.>

## Bug Log Cross-Reference
<List matching entries from docs/bug-log.md as bullet points. Include the entry summary
and which phase it occurred in. If no matches, write: None>

## Performance Signals
<List each signal on its own line, prefixed with its type:>
- [NEGATIVE] <brief description of the signal and what criterion it maps to>
- [POSITIVE] <brief description of the signal and what criterion it maps to>
<If no signals were identified, write: None detected — agent performed within expected parameters>

## Suggestions for agent .md
<Numbered list of concrete, actionable suggestions. Each must:
  - Reference the specific signal that prompted it
  - Describe exactly what text or instruction to add/change in the agent's .md
  - Be phrased as an improvement to the agent definition, not a criticism of the agent>
1. <suggestion>
2. <suggestion>
<If none, write: None>

## Positive Reinforcements
<Numbered list of behaviors to explicitly reinforce in the agent's .md.>
1. <reinforcement>
<If none, write: None>
```

---

## Suggestions Ledger Format

**Path:** `.claude/agent-logs/<agent>/suggestions-ledger.json`
**One ledger per agent.** This file accumulates across all tasks.

```json
{
  "agent": "<agent-name>",
  "last_updated": "<YYYY-MM-DD>",
  "suggestions": [
    {
      "id": "<agent>-<NNN>",
      "summary": "<One sentence describing the suggestion or reinforcement>",
      "category": "<one of: verification | scoping | completion | self-review | context-reading | behavior | reinforcement>",
      "positive": false,
      "count": 1,
      "first_seen": "<YYYY-MM-DD>",
      "last_seen": "<YYYY-MM-DD>",
      "task_ids": ["<task_id>"],
      "status": "pending"
    }
  ]
}
```

### Field definitions

| Field | Type | Description |
|:------|:-----|:------------|
| `id` | string | Unique ID — agent name + zero-padded sequence number (e.g. `dev-backend-007`) |
| `summary` | string | One sentence capturing the core of the suggestion, stable enough to match future occurrences |
| `category` | string | Classifies the type of improvement (see categories below) |
| `positive` | boolean | `true` for reinforcements, `false` for improvements |
| `count` | integer | Number of tasks where this pattern was observed |
| `first_seen` | string | Date the pattern was first logged |
| `last_seen` | string | Date the pattern was most recently logged |
| `task_ids` | array | All task IDs where this pattern was observed |
| `status` | string | `"pending"` / `"applied"` / `"dismissed"` |

### Categories

| Category | Meaning |
|:---------|:--------|
| `verification` | Agent should verify its own output (build, tests, imports) before finishing |
| `scoping` | Agent's scope boundaries need to be clearer |
| `completion` | Agent's definition of "done" needs to be more explicit |
| `self-review` | Agent should check its own output quality before returning |
| `context-reading` | Agent should read available context files before asking questions |
| `behavior` | Agent's actual behavior diverged from its .md description |
| `reinforcement` | A positive behavior to preserve explicitly |

---

## Semantic Matching Rules

When comparing a new suggestion against existing ledger entries to decide whether to
increment or add:

1. **Match** if the core instruction being suggested is the same, even if phrased differently.
   Example: "Run `npm test` before finishing" matches "Execute the test suite prior to
   marking the task complete."

2. **Do not match** if the suggestions target different root causes, even if they sound
   similar. Example: "Run tests to catch import errors" is different from "Run tests to
   verify business logic."

3. **When in doubt, add a new entry** rather than incorrectly merging. False merges hide
   distinct patterns; false splits only result in a lower count.

4. **Never update entries with `status: "applied"` or `status: "dismissed"`** — these
   have been acted on. If the same pattern resurfaces after being dismissed, add a new
   entry with `count: 1`.

---

## ID Generation

IDs follow the pattern `<agent>-<NNN>` where NNN is a zero-padded sequential integer
starting from `001`. To assign the next ID, find the highest existing numeric suffix in
the ledger and increment by 1.

Examples: `dev-backend-001`, `dev-backend-002`, `tester-unit-001`
