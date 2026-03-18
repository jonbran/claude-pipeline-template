---
description: Review agent observation logs and propose targeted improvements to agent .md files. Run after a task completes to surface patterns and apply approved changes.
allowed-tools: Read, Edit, Glob
---

# Agent MD Improver

Read agent observation logs and ledgers, surface suggestions that have been seen 2 or more
times, and propose diffs to the relevant agent `.md` files. Always show diffs with reasoning
before applying. Never modify a file without explicit user approval.

## Step 1: Determine Scope

Parse $ARGUMENTS to determine which ledgers to load:

- `--task <task-id>` → load ledgers for all agents that have a log file for that task ID
- `--agent <agent-name>` → load only that agent's ledger
- `--all` → load all ledgers found under `.claude/agent-logs/`
- *(no argument)* → find the most recently modified log file under `.claude/agent-logs/`
  and use its task ID as if `--task <that-id>` was passed

To find ledger files, glob: `.claude/agent-logs/*/suggestions-ledger.json`
To find log files for a task, glob: `.claude/agent-logs/*/<task-id>.md`
To find the most recent log, glob all `.claude/agent-logs/*/*.md` and pick the newest by
checking file modification context.

## Step 2: Load and Filter Ledgers

For each ledger in scope:
- Read the JSON file
- Filter suggestions to those where `count >= 2` AND `status == "pending"`
- If a ledger has no qualifying suggestions, skip it silently

If no qualifying suggestions exist across any ledger in scope, report:
> "No suggestions with 2+ occurrences found. Keep running tasks — the system is still
> learning. Run `/russell-agent-improver:agent-md-improver --all` to see all tracked patterns."

Then stop.

## Step 3: Load Supporting Evidence

For each qualifying suggestion, load 1–2 observation log files from that agent's log
directory to use as supporting evidence. Prefer the most recent log files.

Also read the current content of each affected agent's `.md` file from `.claude/agents/`.

## Step 4: Organize Proposals

Group proposals by agent. Within each agent, present:
1. Negative suggestions first (improvements), sorted by count descending
2. Positive reinforcements second, sorted by count descending

## Step 5: Present and Apply Changes

For each agent with qualifying suggestions, print a header:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent: <agent-name>  |  .claude/agents/<agent-name>.md
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Then for each suggestion, show a proposal block:

```
### [IMPROVEMENT] <summary>

Seen in <count> tasks: <comma-separated task_ids>
Category: <category>

Supporting evidence:
> <one or two sentences from the observation log explaining what happened>

Proposed change to .claude/agents/<agent-name>.md:

  + <exact text to add, formatted as it would appear in the file>

Apply this change? [y] yes  [n] no  [s] skip all remaining for this agent
```

For positive reinforcements, use `### [REINFORCE]` instead of `### [IMPROVEMENT]`.

After showing each proposal, wait for input:
- `y` → apply the change using Edit tool, then mark the ledger entry `status: "applied"`
- `n` → mark the ledger entry `status: "dismissed"`, move to next proposal
- `s` → skip remaining proposals for this agent, move to next agent

When editing an agent `.md` file:
- Add improvement instructions to the most relevant existing section (e.g. a "Rules" or
  "Before You Finish" section). If no natural section exists, append a new `## Completion
  Checklist` section at the end of the file.
- Add reinforcement text near the behavior it reinforces, or in a `## Strengths to Preserve`
  section.
- Keep additions concise — one to three lines per suggestion. CLAUDE.md files are part of
  the prompt; brevity matters.

## Step 6: Update Ledgers

After processing all proposals for an agent, write the updated ledger back to disk with the
new `status` values and `last_updated` date set to today.

## Step 7: Summary

When all agents have been processed, print a summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent MD Improver — Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Applied:    <N> changes across <M> agent files
Dismissed:  <N> suggestions
Pending:    <N> suggestions (count < 2, still accumulating)

Run again after more tasks to surface additional patterns.
```
