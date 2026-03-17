# Pipeline Review

Full review of the multi-agent development pipeline. Covers correctness, gaps,
drift between the setup script and deployed files, and suggestions for improvement.

---

## Critical Issues

### 1. Orchestrator Phase 4 skips review agents

**File**: `.claude/agents/orchestrator.md:46`
**Issue**: Phase 4 (Development) ends with:

> "Wait for both to complete before proceeding to **Phase 5**."

It should say **Phase 4.5**. As written, the orchestrator is instructed to skip
the Code Review & Security Review phase entirely and jump straight to testing.

The setup script (`setup-claude-pipeline.sh:129`) has the correct text
("Phase 4.5"), so this is a drift issue — the actual deployed file differs from
what the script generates.

**Fix**: Change line 46 of `orchestrator.md` from "Phase 5" to "Phase 4.5".

---

### 2. Setup script is out of sync with deployed files

The setup script and the actual `.claude/` files have diverged in multiple places.
Running the script on a new project will produce **different** (and less complete)
files than what currently exists in this repo.

| File | Deployed version | Setup script version |
|---|---|---|
| `orchestrator.md:46` | Says "Phase 5" (wrong) | Says "Phase 4.5" (correct) |
| `configure-pipeline.md` | 7 steps, includes MCP server selection (Step 4) with 16 server options and full YAML snippets | 6 steps, **no MCP server selection at all** |
| `CLAUDE.md` | 6-item pipeline overview, missing review agents | 7-item pipeline overview, includes review agents |

**Impact**: If someone runs the setup script on a new project today, they get a
`configure-pipeline` command that cannot configure MCP servers, and a `CLAUDE.md`
that doesn't mention review agents. The setup script needs to be regenerated from
the current deployed files.

**Fix**: Regenerate `setup-claude-pipeline.sh` from the current state of all files
in `.claude/` and `CLAUDE.md`. Treat the deployed files as the source of truth.

---

### 3. Both dev agents edit `docs/architecture.md` simultaneously

**Files**: `dev-frontend.md:19-20`, `dev-backend.md:19-20`

Both agents are told to mark tasks as done by changing `- [ ]` to `- [x]` in
`docs/architecture.md`. Since they run in **parallel as background agents**, they
will both be reading and writing the same file concurrently.

**Risk**: One agent's changes overwrite the other's. Claude Code's file editing
operates through the same file system — parallel writes to the same file from
separate agent processes can cause lost edits.

**Fix options** (pick one):
- **Option A**: Remove the checkbox-marking instruction from both dev agents.
  Instead, have the orchestrator update the architecture doc after both agents
  return their completion reports.
- **Option B**: Have each dev agent write its own completion log
  (e.g. `docs/dev-frontend-done.md` / `docs/dev-backend-done.md`) and let the
  orchestrator reconcile them into architecture.md afterward.
- **Option C**: Accept the risk — if tasks are clearly split (FE vs BE prefixed),
  they'll rarely edit the same line, and the last writer wins. This is the simplest
  option but can cause silent data loss.

**Recommendation**: Option A is cleanest.

---

## High-Priority Issues

### 4. Security reviewer has Write and Edit tools it should never use

**File**: `security-reviewer.md:8`

The frontmatter grants `tools: Read, Write, Edit, Bash, Grep, Glob`, but the
agent's own rules (line 83) say:

> "Do NOT modify source files during review — report only"

Granting Write and Edit tools to a report-only agent creates a risk that the LLM
modifies files despite the instruction — especially under ambiguous conditions.

**Fix**: Change the tools line to: `tools: Read, Bash, Grep, Glob`
(matching the code-reviewer agent, which correctly has Read-only + search tools).

---

### 5. No `maxTurns` set on any agent

**Files**: All agent `.md` files

None of the 9 agents specify `maxTurns`. The Handoff doc confirms this is a
supported frontmatter field. Without it, a confused or stuck agent could loop
indefinitely, consuming tokens.

**Recommended values**:
| Agent | Suggested maxTurns | Rationale |
|---|---|---|
| orchestrator | 50 | Manages the full pipeline, needs room |
| ba | 15 | Read ticket, ask questions, write doc |
| architect | 20 | Read codebase, design, write doc |
| dev-frontend | 40 | May need many edit cycles |
| dev-backend | 40 | May need many edit cycles |
| code-reviewer | 15 | Read and report only |
| security-reviewer | 15 | Read and report only |
| tester-unit | 30 | Write tests, run them, report |
| tester-ui | 30 | Install playwright, run tests, report |

---

### 6. No maximum retry count on the bug loop

**File**: `orchestrator.md:69-74` (Phase 6)

The bug loop says "Repeat until all tests pass" with no upper bound. If a bug is
fundamentally unfixable by the dev agent (e.g. a missing dependency, an
architectural issue, a flaky test), the pipeline will loop forever.

**Fix**: Add a max retry count (suggest 3) to the orchestrator's Phase 6:

> "If a fix-test cycle has been attempted 3 times for the same failure, stop the
> loop, report the persistent failure to the user, and ask how to proceed."

---

## Medium-Priority Issues

### 7. Orchestrator doesn't verify git is initialized

**File**: `orchestrator.md:19-23` (Phase 1)

Phase 1 runs `git checkout -b feature/<slug>` without checking that the directory
is a git repository. If run in a non-git project, this will fail with an unclear
error.

**Fix**: Add a pre-check:
```
Before creating a branch, verify git is initialized:
- If `git status` fails, run `git init` and make an initial commit
- Then proceed with branch creation
```

Or at minimum: "If `git status` fails, tell the user this directory is not a git
repository and ask how to proceed."

---

### 8. `CLAUDE.md` is missing the review agents phase

**File**: `CLAUDE.md`

The deployed `CLAUDE.md` lists 6 pipeline steps without mentioning review agents
(code-reviewer and security-reviewer). The setup script's version includes them
as step 6. The delegation rules section also doesn't mention review agents.

**Fix**: Update `CLAUDE.md` to match the 7-step pipeline:
1. Orchestrator
2. BA Agent
3. Architect Agent
4. Dev Agents (parallel)
5. **Review Agents** — Code reviewer and security reviewer (parallel)
6. Test Agents
7. Bug Loop

And add to the delegation rules:
> "Review agents run **in parallel** after dev agents complete"
> "Test agents run **after** review agents complete (not after dev agents)"

---

### 9. Shared/Config tasks have no assigned owner

**File**: `architect.md:54` (Task List template)

The architecture template includes a "Shared / Config Tasks" section (SH-xx), but
neither the orchestrator nor the dev agents have instructions on who handles these.

**Fix**: Add to the orchestrator's Phase 4 instructions:

> "Shared / Config tasks (SH-xx) should be assigned to whichever dev agent is
> most appropriate based on the task content, or run sequentially before the
> parallel dev phase if both agents depend on them."

---

### 10. Jira command doesn't validate HTTP response status

**File**: `commands/jira.md:35-38`

The curl command doesn't check the HTTP status code. The instruction says "If the
request fails (non-200, or returns an error body), report the error" — but the
command relies on the LLM to interpret the JSON output. A 401/403/404 still returns
JSON that `python3 -m json.tool` will happily format.

**Fix**: Add `-w "\nHTTP_STATUS:%{http_code}"` to the curl command so the status
code is explicitly visible in the output:

```bash
curl -s -w "\nHTTP_STATUS:%{http_code}" \
  -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$ARGUMENTS"
```

---

### 11. Review agents can't see git diff — only files passed by orchestrator

**Files**: `code-reviewer.md`, `security-reviewer.md`

The orchestrator is told to "pass the list of all files created or modified during
Phase 4." But without seeing a diff or knowing what changed within those files, the
reviewer is reading entire files without context on what's new vs pre-existing.

**Fix**: Add to the orchestrator's Phase 4.5 instructions:

> "Before spawning review agents, run `git diff --name-only` to get the list of
> changed files, and `git diff` to capture the full diff. Pass both the file list
> and the diff to each reviewer so they can focus on new/changed code."

And add to each reviewer's process:

> "Focus your review on the changes (diff), not the entire file. Pre-existing code
> is out of scope unless a change interacts with it in a risky way."

---

## Low-Priority / Suggestions

### 12. Consider `model: opus` for the orchestrator

All agents use `model: sonnet`. The orchestrator makes the most consequential
decisions (task splitting, evaluating review findings, managing the bug loop).
Using `opus` for just the orchestrator while keeping `sonnet` for the specialist
agents is a good cost/quality tradeoff.

---

### 13. No `memory: project` on any agent

The Handoff doc lists "Add memory: Set `memory: project` on key agents" as a
potential next step. This would let agents learn project conventions across
sessions (e.g. the architect remembering past design decisions).

**Suggested agents for `memory: project`**: orchestrator, architect, dev-frontend,
dev-backend.

---

### 14. Tester agents could benefit from Edit tool access clarification

**File**: `tester-unit.md:6`

The tester-unit has `Edit` in its tools, which is correct since it writes test
files. However, the rules say "Do not modify source files — only write test files."
This is fine but could be strengthened:

> "You have Edit access for test files only. Never edit source code files — if tests
> fail, report the failure and let the dev agent fix the source."

---

### 15. The `tester-ui` agent's Playwright MCP uses array syntax

**File**: `tester-ui.md:11-14`

```yaml
mcpServers:
  - playwright:
      type: stdio
```

But the `configure-pipeline.md` MCP snippets use **object** syntax:

```yaml
mcpServers:
  github:
    type: stdio
```

These are two different YAML structures (array of objects vs object of objects).
Verify which format Claude Code actually expects and standardize on it. The official
docs examples in the Handoff show the **array** format, so the configure-pipeline
snippets may need updating, or vice versa.

---

### 16. No `.gitignore` considerations

The pipeline creates `docs/requirements.md`, `docs/architecture.md`, and
`docs/bug-log.md` as working artifacts. Consider whether these should be committed
(they're useful for PR context) or gitignored (they're ephemeral per-task). If they
should be committed, the orchestrator should stage and commit them as part of
Phase 7. Currently there's no instruction to do so.

---

### 17. Pipeline doesn't handle backend-only or frontend-only tasks

If a task is purely backend (no frontend work), the orchestrator will still try to
split tasks into frontend and backend. There's no instruction for what happens when
one side has no tasks.

**Fix**: Add to orchestrator Phase 4:

> "If the architecture contains only backend tasks and no frontend tasks, spawn only
> `dev-backend`. If only frontend tasks, spawn only `dev-frontend`. Do not spawn an
> agent with an empty task list."

---

### 18. Consider adding an `--upstream` flag to the push command

**File**: `orchestrator.md:86`

The push command is `git push origin <branch-name>`. For first pushes, this should
be `git push -u origin <branch-name>` to set up tracking.

---

## Summary

| Priority | Count | Summary |
|---|---|---|
| Critical | 3 | Phase skip bug, setup script drift, parallel file conflict |
| High | 3 | Security reviewer tools, no maxTurns, infinite bug loop |
| Medium | 5 | No git check, CLAUDE.md gap, shared tasks, Jira validation, reviewer context |
| Low | 7 | Model choice, memory, edit clarification, YAML format, gitignore, single-side tasks, push flag |

### Recommended fix order

1. Fix orchestrator Phase 4 reference (Phase 5 -> Phase 4.5) — 1 line change
2. Remove Write/Edit from security-reviewer tools — 1 line change
3. Add max retry count to bug loop — small paragraph addition
4. Remove architecture.md checkbox marking from dev agents — prevents data loss
5. Add maxTurns to all agents — simple frontmatter additions
6. Update CLAUDE.md to include review agents — small content update
7. Add backend-only / frontend-only handling to orchestrator
8. Regenerate setup script from current deployed files — full script rewrite
9. Everything else in priority order
