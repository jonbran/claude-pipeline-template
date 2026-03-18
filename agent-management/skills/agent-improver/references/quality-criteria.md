# Agent Definition Quality Criteria

## Scoring Rubric

### 1. Role Clarity (15 points)

**15 points**: Agent has a crystal-clear identity
- Opening line states exactly what the agent is
- Responsibilities are explicitly scoped (what it does AND what it doesn't)
- No overlap with other agents in the pipeline

**10 points**: Role is clear but boundaries are soft
- Identity is stated but scope could overlap with other agents

**5 points**: Role is vague or overly broad

**0 points**: No clear role definition

### 2. Process Completeness (25 points)

**25 points**: Process is a complete, unambiguous recipe
- Every step is sequenced and numbered
- Inputs are specified (what the orchestrator passes)
- Decision points are explicit (if X then Y, else Z)
- Dependencies between steps are stated
- No gaps where the agent would need to guess

**20 points**: Process is solid with minor ambiguities

**15 points**: Process covers the happy path but misses edge cases

**10 points**: Process is a rough outline, significant gaps

**5 points**: Process is a vague description

**0 points**: No process documented

### 3. Tool Selection (10 points)

**10 points**: Exactly the right tools, no more, no less
- Tools match the agent's responsibilities
- No tools that enable out-of-scope work (e.g., Write on a read-only reviewer)
- Missing tools would block the agent from completing its work

**7 points**: Tools are mostly right, one extra or one missing

**4 points**: Several unnecessary tools or missing critical ones

**0 points**: Tool list is clearly wrong for the role

### 4. Output Contract (20 points)

**20 points**: Output format is fully specified
- Exact structure the orchestrator expects is documented
- Success and failure outputs are both defined
- Examples or templates provided
- Agent knows what to return and when

**15 points**: Output format is described but incomplete
- Main output is clear but edge cases (errors, blockers) aren't specified

**10 points**: Output is vaguely described ("return a summary")

**5 points**: Output mentioned but no format

**0 points**: No output specification

### 5. Guardrails (20 points)

**20 points**: Boundaries are explicit and enforceable
- "Do NOT" rules are specific and actionable
- Common failure modes are pre-empted (e.g., "do not write tests" for dev agents)
- Scope boundaries prevent work that belongs to other agents
- Instructions for handling ambiguity (stop and ask vs. make a decision)

**15 points**: Main guardrails present but some failure modes not covered

**10 points**: Basic guardrails only

**5 points**: One or two generic rules

**0 points**: No guardrails

### 6. Frontmatter Correctness (10 points)

**10 points**: All frontmatter fields are appropriate
- `model` matches task complexity (sonnet for most, opus for complex reasoning)
- `maxTurns` is adequate for the work (not too low, not wastefully high)
- `permissionMode` matches the agent's write needs
- `tools` list is consistent with the body's described workflow
- `description` is concise and accurate

**7 points**: Mostly correct, one field could be tuned

**4 points**: Several fields are suboptimal

**0 points**: Frontmatter is missing or clearly wrong

## Assessment Process

1. Read the agent definition file completely
2. Cross-reference with the orchestrator's expectations:
   - What does the orchestrator pass to this agent?
   - What does the orchestrator expect back?
   - What phase does this agent run in?
3. Check for consistency between frontmatter and body
4. Score each criterion
5. Calculate total and assign grade
6. List specific issues found
7. Propose concrete improvements

## Red Flags

- Agent has `Write` or `Edit` tools but is supposed to be read-only (reviewers)
- Agent's process references files or paths not passed by the orchestrator
- Guardrails contradict the process steps
- `maxTurns` is lower than the number of process steps
- Agent duplicates work that another agent is responsible for
- Output format doesn't match what the orchestrator parses
- Generic instructions that add no value ("write clean code")
- Missing error/ambiguity handling — agent will guess instead of stopping

## Pipeline-Specific Checks

When assessing agents that are part of a pipeline, also verify:

| Check | Why |
|-------|-----|
| Input handoff | Does the agent know exactly what the orchestrator passes it? |
| Output handoff | Does the orchestrator know exactly what this agent returns? |
| Phase ordering | Does the agent's work fit its pipeline position? |
| Parallel safety | If run in parallel, can this agent conflict with a sibling? |
| Retry compatibility | If re-delegated after a failure, does the agent handle partial state? |
