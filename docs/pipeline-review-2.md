# Pipeline Review 2

Second full review of the multi-agent development pipeline. Verifies that all
findings from the first review (`docs/pipeline-review.md`) were addressed, then
identifies new issues and improvement opportunities.

---

## Status of First Review Findings

All 18 items from the first review have been addressed. The setup script has been
regenerated and is now in sync with the deployed agent files.

| # | Finding | Status |
|---|---------|--------|
| 1 | Orchestrator Phase 4 said "Phase 5" instead of "Phase 4.5" | **Fixed** — `orchestrator.md:58` now correctly says "Phase 4.5" |
| 2 | Setup script out of sync with deployed files | **Fixed** — script regenerated, all files match |
| 3 | Both dev agents edit `docs/architecture.md` simultaneously | **Fixed** — dev agents no longer mark checkboxes; return summary reports instead |
| 4 | Security reviewer had Write/Edit tools | **Fixed** — `security-reviewer.md:8` now has `tools: Read, Bash, Grep, Glob` |
| 5 | No `maxTurns` on any agent | **Fixed** — all 9 agents now have `maxTurns` set |
| 6 | No maximum retry count on bug loop | **Fixed** — `orchestrator.md:94` caps at 3 fix-test cycles |
| 7 | Orchestrator didn't verify git is initialized | **Fixed** — `orchestrator.md:22-24` now checks `git status` first |
| 8 | `CLAUDE.md` missing review agents phase | **Fixed** — now lists all 7 phases including review agents |
| 9 | Shared/Config tasks had no assigned owner | **Fixed** — `orchestrator.md:48-49` assigns SH-xx tasks appropriately |
| 10 | Jira command didn't validate HTTP response status | **Fixed** — `jira.md:36` now includes `-w "\nHTTP_STATUS:%{http_code}"` |
| 11 | Review agents couldn't see git diff | **Fixed** — `orchestrator.md:63-66` now runs `git diff main` and passes output |
| 12 | Consider `model: opus` for orchestrator | **Not applied** — still `model: sonnet` (intentional choice, see item 1 below) |
| 13 | No `memory: project` on any agent | **Not applied** — still no memory set (see item 2 below) |
| 14 | Tester edit clarification | **Not applied** — minor, original wording is adequate |
| 15 | Playwright MCP array vs object YAML syntax | **Not applied** — still uses array syntax (see item 5 below) |
| 16 | No `.gitignore` considerations for docs/ artifacts | **Not applied** — see item 7 below |
| 17 | Pipeline doesn't handle backend-only or frontend-only tasks | **Fixed** — `orchestrator.md:51-52` now skips agents with no tasks |
| 18 | Push command missing `-u` flag | **Fixed** — `orchestrator.md:111` now uses `git push -u origin` |

---

## New Issues

### Critical

_None found. The pipeline structure is sound after the first round of fixes._

---

### High Priority

#### 1. No intermediate commits — full pipeline work is uncommitted

**Files**: `orchestrator.md`, `dev-frontend.md`, `dev-backend.md`

The orchestrator creates a branch in Phase 1 but never commits until Phase 7
(where it asks to push). All code written by dev agents, all test files written
by testers — everything sits as uncommitted working tree changes throughout the
entire pipeline run.

**Risks**:
- If Claude Code crashes, the session is interrupted, or context is compressed
  mid-pipeline, all work is lost with no recovery point
- `git diff main` in Phase 4.5 shows the diff correctly (working tree vs main),
  but if the user has other uncommitted changes in the repo, those get mixed in
- No ability to roll back to "after dev, before tests" if something goes wrong

**Fix**: Add a commit step to the orchestrator between phases:

```
### After Phase 4 (Development) completes:
Stage and commit all changes:
  git add -A
  git commit -m "feat: implement <task-slug> — dev agents complete"

### After Phase 6 (Bug Loop) completes / all tests pass:
Stage and commit test files and any bug fixes:
  git add -A
  git commit -m "test: add tests for <task-slug>"
```

This gives recovery points and cleaner git history.

---

#### 2. Untracked files invisible to `git diff main`

**File**: `orchestrator.md:63-65`

Phase 4.5 runs `git diff main --name-only` and `git diff main` to collect
changes for reviewers. However, `git diff` only shows modifications to
**tracked** files. New files created by dev agents (which is most of what they
do) are **untracked** and won't appear in this diff.

**Impact**: Review agents receive an incomplete or empty diff, missing all newly
created files. They fall back to reading individual files but without diff
context, defeating the purpose of the change.

**Fix**: Either:
- **Option A** (best): Add intermediate commits as described in item 1 above.
  Then `git diff main` captures everything.
- **Option B**: Change the diff commands to also capture untracked files:
  ```bash
  git add -N .  # mark untracked files as "intent to add" (no content staged)
  git diff main --name-only
  git diff main
  ```
- **Option C**: Add `git status --short` alongside the diff to at least list
  new files, and instruct reviewers to read those files directly.

---

#### 3. Dev agents lack `Agent` tool — cannot ask user questions

**Files**: `dev-frontend.md:6`, `dev-backend.md:6`

Both dev agents have `tools: Read, Write, Edit, Bash, Glob, Grep`. If a dev
agent encounters an ambiguity or blocker, the prompt says "note it clearly in
your return report." But by that point, the work is already done (or not done).

**Consideration**: This is actually the correct design — sub-agents should not
independently prompt the user since the orchestrator manages user communication.
However, neither agent has a way to **early-exit** and return to the orchestrator
with a question. They will either guess and continue (risky) or waste their
remaining turns trying.

**Fix**: Add to both dev agent prompts:

```
- If you encounter a genuine ambiguity that would significantly change the
  implementation, STOP immediately and return to the orchestrator with a clear
  description of the question. Do not guess. Do not continue with assumptions.
```

The orchestrator should then be told:

```
If a dev agent returns with an unresolved question instead of completed work,
relay the question to the user, get the answer, and re-delegate to the same
dev agent with the answer included.
```

---

### Medium Priority

#### 4. BA agent `permissionMode: default` causes unnecessary prompts

**File**: `ba.md:9`

The BA agent's only write operation is creating `docs/requirements.md`. With
`permissionMode: default`, the user gets prompted to approve this write every
time the BA runs. Since this is expected behavior, it's a friction point.

**Fix**: Change to `permissionMode: acceptEdits`. The BA agent only writes to
`docs/` — there's no risk of unwanted file modifications.

Same applies to `architect.md:9` which also only writes to `docs/`.

---

#### 5. Playwright MCP YAML array syntax inconsistency

**File**: `tester-ui.md:11-14`

```yaml
mcpServers:
  - playwright:
      type: stdio
```

This uses **array** syntax (note the `-` prefix). But all MCP server snippets in
`configure-pipeline.md` use **object** syntax:

```yaml
mcpServers:
  playwright:
    type: stdio
```

If Claude Code expects one format, the other will silently fail to load the MCP
server. The Handoff doc shows array format. The configure-pipeline command writes
object format.

**Fix**: Test which format Claude Code actually accepts and standardize across
all files. If both work, prefer the simpler object format and update `tester-ui.md`.

---

#### 6. Orchestrator doesn't stage files before collecting diff for reviewers

**File**: `orchestrator.md:63-66`

Related to item 2 above. Even if intermediate commits are added, the orchestrator
should verify the working tree is clean before running Phase 4.5. If dev agents
left unstaged changes (e.g. due to a partial maxTurns timeout), those changes
would be invisible to reviewers.

**Fix**: Add to Phase 4.5:

```
Before collecting the diff, verify all changes are committed:
  git status --short
If there are uncommitted changes, stage and commit them first.
```

---

#### 7. No guidance on committing docs/ artifacts

**Files**: `orchestrator.md` (Phase 7)

The pipeline generates `docs/requirements.md`, `docs/architecture.md`, and
potentially `docs/bug-log.md`. Phase 7 only mentions pushing — there's no
instruction on whether these files should be:
- Committed (useful for PR reviewers to understand the design)
- Cleaned up / deleted before commit (if they're considered ephemeral)
- Left as-is (risk of stale docs accumulating)

**Fix**: Add to the orchestrator's Phase 7:

```
Before asking the user to push:
- Stage docs/requirements.md and docs/architecture.md for commit (they provide
  useful context in the PR)
- Delete docs/bug-log.md if it exists and all tests pass (it's no longer relevant)
- Commit: "docs: add requirements and architecture for <task-slug>"
```

---

#### 8. Tester-ui agent missing Glob and Grep tools

**File**: `tester-ui.md:7`

The tester-ui agent has `tools: Read, Write, Bash` but no `Glob` or `Grep`.
If the agent needs to find component files, locate existing test patterns, or
search for route definitions, it must use `Bash` with `find`/`grep` — which is
less reliable and bypasses Claude Code's built-in search tools.

**Fix**: Add `Glob, Grep` to the tools list:
```yaml
tools: Read, Write, Bash, Glob, Grep
```

---

### Low Priority / Suggestions

#### 9. Consider `model: opus` for orchestrator

Carried forward from the first review. The orchestrator makes the most
consequential decisions (task splitting, evaluating review severity, managing the
bug loop, deciding when to prompt the user). Using `opus` for this one agent
while keeping `sonnet` for specialists is a good cost/quality tradeoff.

**Counter-argument**: Sonnet is faster and cheaper. If the orchestrator's
decisions have been reliable in practice, keep sonnet. This is a judgment call
after real-world testing.

---

#### 10. Consider `memory: project` for key agents

Carried forward from the first review. Adding `memory: project` to the
orchestrator, architect, and dev agents would let them learn project conventions
across sessions — e.g., the architect remembering past design patterns, dev
agents remembering coding style preferences.

**Suggested**: `orchestrator`, `architect`, `dev-frontend`, `dev-backend`.

---

#### 11. No explicit encoding/line-ending instructions for dev agents

If the project has specific requirements (e.g., LF-only line endings, UTF-8
without BOM), dev agents may introduce inconsistencies. This is a minor risk that
becomes relevant for cross-platform teams.

**Fix**: Add to configure-pipeline's detection step: check for `.editorconfig`
and include its settings in the dev agent `## Project Context` sections.

---

#### 12. Orchestrator doesn't handle sub-agent maxTurns timeout gracefully

**File**: `orchestrator.md`

If a dev agent hits its `maxTurns: 40` limit mid-task, it returns whatever it
has. The orchestrator has no explicit instruction for handling partial completions.
It may proceed to review with half-implemented features.

**Fix**: Add to the orchestrator after each dev agent completes:

```
When a dev agent returns, verify its report. If the report indicates incomplete
tasks or mentions hitting a limit, do NOT proceed to review. Instead:
1. Report to the user which tasks remain incomplete
2. Ask whether to re-delegate the remaining tasks or stop the pipeline
```

---

#### 13. The `/jira` command doesn't use the orchestrator agent

**File**: `commands/jira.md:59`

Step 4 says: "Pass the formatted task brief to the **orchestrator** agent to
begin the full development pipeline." But the jira command's `allowed-tools` is
`Bash, Read` — it doesn't have the `Agent` tool, so it cannot spawn the
orchestrator.

In practice, the jira command runs in the main conversation context and the
orchestrator is already defined as a proactive agent, so it likely picks up the
task. But the explicit handoff instruction is misleading.

**Fix**: Either:
- Add `Agent` to the jira command's `allowed-tools`
- Or change Step 4 to: "Present the formatted task brief in your response. The
  orchestrator agent will pick up the task automatically."

---

#### 14. Setup script doesn't check for existing `.claude/` directory

**File**: `setup-claude-pipeline.sh`

The script uses `cat > file` (overwrite) for every file. If run on a project that
already has customized agent files (e.g., after running `/configure-pipeline`),
it will silently destroy all customizations.

**Fix**: Add a check at the top of the script:

```bash
if [ -d ".claude/agents" ]; then
  echo "WARNING: .claude/agents/ already exists. Running this script will overwrite all agent files."
  echo "Press Ctrl+C to cancel, or Enter to continue."
  read
fi
```

---

#### 15. No `.claude/settings.json` for project-level tool permissions

Claude Code supports a `.claude/settings.json` file for project-level settings
(e.g., pre-approving certain tools, setting default models). The pipeline doesn't
create one. This is fine for now but could reduce permission prompts if specific
tools are pre-approved at the project level.

---

## Architecture Assessment

### What works well

- **Clear phase separation**: Each phase has explicit entry/exit criteria. The
  orchestrator knows exactly when to proceed.
- **Parallel execution where safe**: Dev agents and review agents run in parallel,
  which is the right call for independent work streams.
- **Review before test**: Running code review and security review before testing
  catches issues earlier and avoids wasting test cycles on broken code.
- **Bug loop with cap**: The 3-retry maximum prevents infinite loops while giving
  reasonable fix attempts.
- **Configure-pipeline command**: The MCP server selection menu with 16 options
  and auto-detection is thorough and well-structured.
- **Setup script**: Clean, portable, and now in sync with deployed files.

### Areas to watch during real-world use

1. **Token consumption**: A full pipeline run (9 agents, each with their own
   context) will consume significant tokens. Monitor costs per ticket.
2. **Parallel agent file conflicts**: Even without the architecture.md issue,
   dev-frontend and dev-backend could theoretically edit the same file (e.g., a
   shared config file). The orchestrator's task splitting needs to be clean.
3. **Context window pressure**: Long-running agents (dev agents at 40 turns)
   may approach context limits. The `maxTurns` settings are reasonable but watch
   for quality degradation in later turns.
4. **Jira field mapping**: The jira command extracts specific fields. Different
   Jira configurations may have custom fields for acceptance criteria, story
   points, etc. May need per-project tuning.

---

## Summary

| Priority | Count | Key themes |
|----------|-------|------------|
| High | 3 | No intermediate commits, untracked files invisible to diff, dev agent early-exit |
| Medium | 5 | BA/architect permission mode, YAML syntax, staging before review, docs cleanup, tester-ui tools |
| Low | 7 | Model choice, memory, encoding, maxTurns timeout, jira handoff, script safety, settings.json |

### Recommended fix order

1. Add intermediate commits to the orchestrator pipeline (items 1 + 2 + 6)
2. Add early-exit guidance to dev agents (item 3)
3. Fix YAML syntax inconsistency for Playwright MCP (item 5)
4. Change BA/architect `permissionMode` to `acceptEdits` (item 4)
5. Add docs/ artifact handling to Phase 7 (item 7)
6. Add Glob/Grep to tester-ui (item 8)
7. Add `Agent` to jira command's allowed-tools (item 13)
8. Add overwrite warning to setup script (item 14)
9. Remaining items by priority

### Overall assessment

The pipeline is in good shape after the first review's fixes. The architecture is
sound, the agent separation is clean, and the orchestrator flow is well-defined.
The highest-impact improvement is adding intermediate git commits between pipeline
phases — this is the main gap between "works in a demo" and "reliable in
production use."
