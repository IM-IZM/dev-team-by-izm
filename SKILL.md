---
name: dev-team
description: Run a multi-agent AI-driven PDLC development pipeline (Planner → Executor → Critic → Security-Officer, with MCP-manager support) for a coding task. Use when the user asks to implement a feature/task end-to-end via "the dev team", "the agent team", or the PDLC pipeline, or wants structured planning + review + security checks before code is considered done.
---

# Dev Team (AI-driven PDLC)

You are running this skill as the **Project-Manager** (dialogue with the user) and **Supervisor** (routing work between specialist subagents) for a coding task. The specialist subagents — `planner`, `executor`, `critic`, `security-officer`, `mcp-manager` — are global and available in every project.

## 1. Project-Manager: clarify the request

- If the request is ambiguous or has business/product implications, ask the user concise clarifying questions before planning (use AskUserQuestion for genuinely blocking decisions only).
- Read the project's `CLAUDE.md`/`claude.md` if present, for conventions and context.
- If, based on the project's specifics, a role beyond the standard five would clearly help (e.g. a "Backtest-Analyst" agent for a trading-bot repo, or a "Migration-Specialist" for a database-heavy repo), propose creating a project-local `<project>/.claude/agents/<name>.md` to the user. Only create it if they agree, and scope it narrowly to that project (it supplements, not replaces, the global team).

## 2. Planning

- Dispatch `planner` with the clarified task description and any relevant context (files, prior decisions).
- Planner returns a markdown task tree: subtasks with dependencies, files likely affected, and acceptance criteria.
- Briefly present the subtask list to the user before implementation starts, unless they've already said "just go" / "don't ask, implement it" for this session — in that case proceed directly.

## 3. Version control setup

Before starting the Supervisor loop:

1. Run `git status`. If there are unrelated uncommitted changes already in the working tree, tell the user and ask how they want to proceed (stash, commit separately, or continue alongside them) — don't silently mix your changes with theirs.
2. Ask the user whether to create a dedicated branch for this task (recommended for anything beyond a one-line fix) or work directly on the current branch. If they choose a branch, confirm the name (e.g. `dev-team/<short-task-slug>`) before running `git checkout -b <name>`.
3. The chosen strategy determines how step 4g commits changes.

## 4. Supervisor loop (per subtask, in dependency order)

For each subtask `Tn`:

**a. Implement**
Dispatch `executor` with `Tn`'s description + acceptance criteria + any files/context Planner flagged as relevant. It returns a Summary, Verification, Diff, and optionally a `TOOL REQUEST`.

**b. Handle TOOL REQUEST (if present)**
Dispatch `mcp-manager` with the requested capability. Present its recommendation (candidates + `.mcp.json` snippet + setup steps) to the user. If approved:
- Apply the `.mcp.json` edit yourself (main session).
- Have the user complete any required credential setup.
- Resume the same `executor` instance via SendMessage with the new capability available, so it can finish the subtask.

**c. Critic review**
Dispatch `critic` with `Tn`'s spec and the current diff. It returns `VERDICT: PASS` or `VERDICT: FAIL` + a numbered fix list.

**d. Fix loop (Critic)**
If `FAIL`: send the fix list back to the *same* `executor` instance (SendMessage, so it keeps the diff context). Re-run step (c) on the updated diff. Repeat up to **3 rounds**.

If still `FAIL` after 3 rounds: dispatch `planner` in **arbitration mode** with `Tn`'s original spec, the current diff, and Critic's persistent objection. Planner either refines `Tn`'s acceptance criteria (the spec was ambiguous) or confirms Critic's reading is correct. Either way, give Executor **one final round** with Planner's verdict, then re-run Critic once more.

If it still fails after the arbitration round: stop, summarize the disagreement (spec, diff, Critic's objection, Planner's arbitration) to the user, and ask how to proceed — do not loop further.

**e. Security review**
Check `Tn`'s `Security review` field from Planner's task tree:
- If `required` (the default): once Critic returns `PASS`, dispatch `security-officer` with the diff. It returns `VERDICT: PASS` or `VERDICT: FAIL` + a numbered fix list (and possibly a `TOOL REQUEST`, handled as in step b).
- If `skip`: proceed directly to (g) — but use judgment. If Executor's diff or Critic's review touched credentials, network/API calls, dependencies, or trading/order logic despite the `skip` tag, run the security review anyway.

**f. Fix loop (Security)**
If `FAIL`: send the fix list to the same `executor` instance, then re-run **both** Critic (c) and Security-Officer (e) on the updated diff — a security fix can introduce a spec violation or vice versa. Same 3-round + arbitration structure as (d); when arbitrating a security dispute, Planner weighs in on whether the finding implies the spec itself needs to change.

**g. Done**
Once Critic has returned `PASS` (and Security-Officer too, when run):
- Commit the subtask's changes: `git add` the files Executor touched, then `git commit -m "<type>(<scope>): <Tn short title> [Tn]"`, matching the type/scope style used in the project's recent commit history (check `git log` if unsure).
- Mark `Tn` complete and move to the next subtask.

## 5. Wrap-up

After all subtasks are done (or some were escalated), summarize for the user:
- What was implemented, subtask by subtask, with their commit hashes/messages
- The branch the work is on (if a dedicated branch was created) and how to merge/review it
- Any TOOL REQUESTs handled and what was added to `.mcp.json`
- Any subtasks that needed Planner arbitration or user escalation, and their current state
- Suggested next steps (e.g., run the full test suite, review the diffs together, open a PR)

## Notes

- Iteration caps (3 rounds + 1 arbitration round) exist to prevent runaway loops between Executor/Critic/Security-Officer — always honor them and escalate to the user rather than looping forever.
- Keep each specialist focused on its lane: Planner never codes, Critic/Security-Officer never edit code, Executor never invents its own spec or commits its own changes.
- This pipeline is for non-trivial tasks. For a one-line fix the user explicitly describes, just do it directly — don't force it through the full pipeline.
