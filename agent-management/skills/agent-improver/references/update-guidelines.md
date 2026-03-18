# Agent Definition Update Guidelines

## Core Principle

Only change instructions that will measurably improve agent performance. Agent definitions are prompts — every line costs attention. Additions must earn their place.

## What TO Change

### 1. Instructions the Agent Ignored

```diff
 ## Rules
 - Do not write tests — that is the tester agent's responsibility
+- Do NOT create test files, test directories, or any file with "test" or "spec"
+  in the name. If you notice missing test coverage, note it in your return
+  report instead.
```

Why: If an agent repeatedly violates a rule, the rule needs to be stronger, more specific, or repeated in a different context within the definition.

### 2. Missing Process Steps

```diff
 ## Your Process
 1. Read the requirements and architecture files
 2. Inspect the existing codebase structure
+3. Check the project's package.json (or equivalent) for existing dependencies
+   before adding new ones — prefer what's already installed
 4. Work through each assigned task in dependency order
```

Why: If the agent consistently misses a step that causes rework, add it explicitly.

### 3. Output Format Mismatches

```diff
 ## Output
-Return a summary of what you did.
+Return a structured report to the orchestrator:
+
+- **Tasks completed**: list each task ID with the files affected
+- **Files created**: full paths
+- **Files modified**: full paths
+- **Blockers**: any unresolved items (if none, say "None")
```

Why: Vague output instructions produce inconsistent returns that the orchestrator can't parse.

### 4. Missing Guardrails Discovered During Execution

```diff
 ## Rules
+- Do NOT run `git commit`, `git push`, or any git write operations — the
+  orchestrator handles all git operations
+- Do NOT install new dependencies without noting them in your return report
```

Why: If an agent did something it shouldn't have, add an explicit prohibition.

### 5. Frontmatter Tuning

```diff
-maxTurns: 15
+maxTurns: 25
```

Why: If an agent consistently hits its turn limit before completing work, increase it. If it finishes in 5 turns with maxTurns: 40, consider reducing it.

### 6. Ambiguity Handling

```diff
 ## Rules
+- If the architecture.md references a file or pattern that doesn't exist in
+  the codebase, STOP and return to the orchestrator with the question.
+  Do not create placeholder implementations.
```

Why: Agents that guess instead of asking create rework.

## What NOT to Change

### 1. Working Agents

If an agent consistently produces good results, leave it alone. Don't "improve" something that isn't broken.

### 2. Generic Advice

Bad addition:
```diff
+- Write clean, maintainable code
+- Follow best practices
+- Use meaningful variable names
```

The model already knows this. These lines waste prompt space.

### 3. One-Off Fixes

Bad addition:
```diff
+- When working with the UserService, make sure to import from src/services not src/legacy
```

This is project-specific knowledge that belongs in CLAUDE.md, not in a reusable agent definition.

### 4. Over-Specification

Bad addition:
```diff
+- Use 2-space indentation for JavaScript files
+- Use single quotes for strings
+- Always add trailing commas
```

The agent already inspects existing code style. These clutter the definition and may conflict with actual project conventions.

### 5. Duplicating the Orchestrator

Bad addition:
```diff
+- After completing your work, the orchestrator will run code review, then
+  security review, then testing
```

The agent doesn't need to know the full pipeline — it just needs to know its own role and handoff points.

## Diff Format for Updates

For each suggested change:

### 1. Identify the File

```
File: .claude/agents/dev-backend.md
Section: Rules
```

### 2. Show the Change

```diff
 ## Rules
 - Do not write tests — that is the tester agent's responsibility
+- Do NOT create any files in test/ or __tests__/ directories
```

### 3. Explain Why

> **Why this helps:** During the last pipeline run, dev-backend created
> 3 test files that conflicted with what tester-unit later generated.
> This stronger guardrail prevents the duplication.

## Validation Checklist

Before finalizing an update, verify:

- [ ] The change addresses a real, observed problem (not hypothetical)
- [ ] The instruction is specific and actionable (not vague)
- [ ] It doesn't duplicate instructions already in the definition
- [ ] It doesn't conflict with existing rules
- [ ] It belongs in the agent definition (not in CLAUDE.md or the orchestrator)
- [ ] The agent would behave differently with this change vs. without it
- [ ] Frontmatter changes are consistent with the body

## When to Update Agents vs. Other Files

| Observation | Update |
|-------------|--------|
| Agent ignores its own instructions | Strengthen the agent definition |
| Agent lacks project-specific context | Add to CLAUDE.md, not the agent |
| Orchestrator passes wrong info | Fix the orchestrator, not the agent |
| Agent's output doesn't match orchestrator expectations | Fix both — align the contract |
| Agent does work belonging to another agent | Add guardrail to the overstepping agent |
| Pipeline phase ordering is wrong | Fix the orchestrator |
