# Agent Observer — Assessment Criteria

Use this reference during Step 3 of the observation skill. For each criterion, check whether
the signal is present based on the arguments passed by the orchestrator, the bug log, and
the agent's current .md definition.

---

## Negative Signals

These indicate the agent's .md definition could be clearer, more directive, or better scoped.
Each signal should produce at least one concrete suggestion.

### 1. Retries Required
**Condition:** `retries >= 1`
**What it means:** The agent did not complete its work correctly on the first delegation.
**What to look for:**
- Did the agent miss a step it should know to always do (e.g. running tests, verifying imports)?
- Did the agent fail to check something that could have prevented the retry?
- Was the agent's scope unclear, causing it to under-deliver?
**Suggestion shape:** Add an explicit checklist item or "before you complete" instruction to the agent's .md.

### 2. Build Failure Caused by Agent
**Condition:** Bug log contains an entry referencing this agent and this task where the failure
was caused by the agent's output (e.g. missing import, syntax error, wrong API call).
**What it means:** The agent submitted work that broke the build.
**What to look for:**
- Does the agent's .md instruct it to verify the build before declaring complete?
- Does the agent's .md instruct it to run the test suite before finishing?
**Suggestion shape:** Add explicit pre-completion verification steps to the agent's .md.

### 3. Agent Exceeded Its Scope
**Condition:** `returned` summary indicates the agent made changes outside its defined responsibility.
**What it means:** The agent's scope boundaries are not clear enough in its .md.
**What to look for:**
- Did a backend agent touch frontend files, or vice versa?
- Did a reviewer make edits instead of just reporting?
- Did a tester write implementation code?
**Suggestion shape:** Add explicit scope boundary statements to the agent's .md.

### 4. Agent Under-Delivered on Scope
**Condition:** `returned` summary indicates incomplete work, or orchestrator had to pick up tasks
the agent should have handled.
**What it means:** The agent's responsibilities are not clearly enough defined.
**What to look for:**
- Are there tasks in the agent's defined scope that it consistently skips?
- Is the agent's .md too vague about what "done" means?
**Suggestion shape:** Add an explicit definition of done or completion checklist to the agent's .md.

### 5. Agent Behavior Diverged from Its .md Description
**Condition:** Comparing `requested`/`returned` against the agent's .md reveals the agent did
something its definition does not describe, or failed to do something its definition says it should.
**What it means:** The agent's .md is drifting from how the agent actually operates.
**What to look for:**
- Steps listed in the .md that were not followed
- Steps the agent took that are not in the .md
**Suggestion shape:** Update the .md to reflect actual behavior, or add missing instructions.

### 6. Agent Produced Output Requiring Heavy Downstream Correction
**Condition:** A reviewer or tester agent for the same task reported CRITICAL/HIGH issues that
required re-delegation back to this agent, and these issues were in the original output.
**What it means:** The agent is missing quality or completeness checks.
**What to look for:**
- Does the agent's .md include self-review steps before submission?
- Does the agent's .md mention common error categories it should check for?
**Suggestion shape:** Add a self-review checklist relevant to the agent's domain.

### 7. Agent Asked Questions It Could Have Inferred
**Condition:** `returned` or `errors` indicates the agent asked clarifying questions that were
already answerable from `docs/requirements.md` or `docs/architecture.md`.
**What it means:** The agent is not being instructed clearly enough to read available context.
**What to look for:**
- Does the agent's .md instruct it to read requirements and architecture before starting?
- Is there missing domain knowledge that should be codified in the agent's .md?
**Suggestion shape:** Add instructions to read specific context files before asking questions.

---

## Positive Signals

These indicate behaviors worth explicitly preserving in the agent's .md definition.

### P1. Clean Completion
**Condition:** `retries == 0` AND `errors == "none"` AND no bug log entries for this agent/task.
**What it means:** The agent performed its full scope correctly on the first attempt.
**Reinforcement shape:** Note which behaviors were present (e.g. "ran tests before completing",
"stayed within scope") and reinforce them explicitly in the .md if not already stated.

### P2. Self-Correction Without Escalation
**Condition:** `retries >= 1` but all errors were resolved by the agent itself — orchestrator
did not need to re-delegate due to an external blocker.
**What it means:** The agent has good self-recovery behavior.
**Reinforcement shape:** Add an explicit instruction to attempt self-diagnosis before escalating.

### P3. Caught a Downstream Risk
**Condition:** `returned` summary indicates the agent flagged an issue that would have caused
problems in a later phase (e.g. a dev agent flagged a security concern, a reviewer caught
a regression before testing).
**What it means:** The agent is thinking ahead, beyond its immediate scope.
**Reinforcement shape:** Explicitly encourage this cross-phase awareness in the agent's .md.

---

## Suggestion Quality Rules

Every suggestion must meet these standards before being written to the ledger:

1. **Specific** — "Add a step to run `npm test` before marking work complete" not "should test more"
2. **Actionable** — must describe an addition or change to the agent's .md, not a vague observation
3. **Scoped** — must be about this agent's behavior, not the pipeline in general
4. **Non-redundant** — do not suggest something already explicitly stated in the agent's .md
5. **Evidence-based** — must be traceable to a specific signal from this observation
