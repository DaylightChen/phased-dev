---
description: Run the per-task dev loop (implement → test → review → fix) for the specified task in the active scope, with you acting as orchestrator
argument-hint: "<task-number> (e.g., 03 or 8a)"
---

# Start Task — Dev Loop Orchestration

User invoked with task number: $ARGUMENTS

You are the **orchestrator** for the dev loop on this task in the **active scope**. Your job is to dispatch the implementer, tester, and reviewer subagents in the correct sequence, decide when to iterate, record everything in `log.md`, and commit when done.

The scope (project vs feature) is determined by `docs/.phased-dev/state.json`. This command works identically for both — only the task directory and reviewer context differ, both of which are read from the active scope's config.

## Pre-flight checks

1. **Confirm scope state exists.** Read `docs/.phased-dev/state.json`. If missing, tell the user to run `/phased-dev:init-project` first.

2. **Read the active scope.** Get `activeScope` from `state.json`, then read `docs/.phased-dev/scopes/<activeScope>.json`. Note:
   - `currentPhase` — must be `implement`. If not, stop and tell the user the active scope is in phase `<currentPhase>`; they should run `/phased-dev:start-phase` for that phase first, or switch scopes.
   - `paths.tasks` — the tasks directory for this scope (e.g., `docs/tasks/` or `docs/features/<name>/tasks/`)
   - `paths.plan` — the plan file (used by the reviewer)
   - `paths.decisions` — the decision log for this scope
   - `type` — recorded in the scope JSON for reference; commands branch on path presence, not type strings.
   - Check `paths.engineeringDir` (project-style) vs `paths.engineering` (feature-style) to determine scope shape.

3. **Locate the task directory.** Glob `<paths.tasks>task-${ARGUMENTS}-*/`. If zero matches, stop and tell the user the task number doesn't exist in this scope (suggest they check the plan). If multiple matches, list them and ask the user to disambiguate.

4. **Read the task `brief.md` end to end.** Verify it has the required sections (Goal, Context files, Downstream dependencies, Steps, Acceptance criteria, Output files). If anything's missing, stop and tell the user. Also note the brief's **Loop profile** (`full` or `lightweight`) if present — it defaults to `full` when absent. This selects the dev-loop variant below.

5. **Verify upstream output exists.** Check that the files the reviewer will need are present:
   - `paths.plan` — the implementation plan. If missing, the plan phase output is gone; tell the user to re-run `/phased-dev:start-phase` for the `plan` phase.
   - If `paths.engineeringDir` is set (project-style scope): at least one file under it. If missing, the engineering spec is gone.
   - If `paths.engineering` is set (feature-style scope): that file must exist. If missing, the feature engineering spec is gone.
   If any upstream output is missing, stop and tell the user which files are absent.

6. **Initialize the log.** If `log.md` doesn't exist in the task directory, copy `docs/plan/log-template.md` (or `${CLAUDE_PLUGIN_ROOT}/templates/log-template.md` if the project copy is missing) to `<task-dir>/log.md` and customize the heading to reference the scope and task.

7. **Read `docs/plan/execution-methodology.md`** — it defines the rules for the dev loop. This command defines dispatch details; the methodology defines the protocol. If they conflict, the methodology wins.

## The dev loop

Follow the handoff protocol from execution-methodology §2.3. The steps below define **what to dispatch and how to record** — for **why** each step exists and **when to terminate**, see the methodology.

### Select the loop profile

Per execution-methodology §1.5, pick the profile before Iteration 1:

- **`full`** (default) — implement → test → review → fix. Use this unless the task clearly qualifies for lightweight.
- **`lightweight`** — implement → review → fix, **skipping the test-authoring (Step B) stage**. Eligible only when the task introduces **no executable runtime behavior** (pure type/interface declarations, constants, configuration, or docs).

Determine the profile from the brief's **Loop profile** field (default `full`). Then **independently confirm eligibility** — do not trust the label blindly:

- Skim the brief's Steps and Output files. If anything adds runtime logic, branching, I/O, or observable behavior, override to **`full`** and tell the user you did so and why.
- If the brief says `lightweight` but you are not confident it qualifies, use **`full`**. Lightweight is only for the obviously-trivial case.

Record the chosen profile (and any override) in `log.md` under a one-line "Loop profile" note before Iteration 1.

Even under `lightweight`, the quality gate is unchanged: the type-checker and the full **existing** test suite still run, and the reviewer still runs every iteration (see Completion and Step D below).

### Iteration 1

**Step A — Implement.** Dispatch the `implementer` subagent with:
- The full path to the task `brief.md`
- An instruction to read the brief and execute it per its specification

When it returns, append its report to `log.md` under "Iteration 1 → Implement" per execution-methodology §6 (what to capture).

**Step B — Test.** *(Full profile only — skip this step entirely for `lightweight`.)* Dispatch the `tester` subagent with:
- The full path to the brief
- The list of files the implementer touched
- An instruction to write tests, run the full suite, and run type-checking

Append its report to `log.md` under "Iteration 1 → Test" per execution-methodology §6.

**For the `lightweight` profile, instead of Step B:** run the project's type-checker and the existing full test suite yourself via Bash (discover the commands the same way the tester would — `package.json` scripts, `Makefile`, `CLAUDE.md`, or the brief). Paste the real output into `log.md` under "Iteration 1 → Check (lightweight)". These must be clean before proceeding — a lightweight task may add no tests, but it must not break the type-check or any existing test.

**Step C — Decide.** Per execution-methodology §1.2: if the tester (full) or the type-check / existing suite (lightweight) reports failures or regressions → go to Fix iteration. Otherwise:

**Step D — Review.** Before dispatching the reviewer, **prepare a reviewer brief** by reading `paths.plan` yourself and extracting:
  - The task list (numbered, with one-line summaries)
  - The dependency ordering rationale
  - The current task's position in the sequence
  - The briefs of the next 2-3 tasks under `paths.tasks`

  Write this as a temporary summary (you can paste it directly in the dispatch prompt). This keeps the reviewer focused without loading the full plan into its context.

  Dispatch the `reviewer` subagent with:
- The full path to the brief
- The diff (run `git diff` to capture it)
- The tester's report (full profile), or your type-check + existing-suite output (lightweight profile)
- **The reviewer brief** (task list + dependency rationale + next 2-3 task briefs) instead of the full plan file. The reviewer uses this to catch local-but-globally-wrong decisions without needing the entire plan in context.
- The active scope's engineering spec — if `paths.engineeringDir` is set, use it (project-style); if `paths.engineering` is set, use that file (feature-style). The reviewer must confirm the implementation respects the architecture, not just the plan.
- **If `paths.uxDir` is set:** also instruct the reviewer to read the UX spec (the most recent markdown file under `paths.uxDir`). This catches microcopy, a11y, and component deviations.
- **If `paths.engineering` is set (feature-style scope):** also instruct the reviewer to read the project's most recent engineering spec under `docs/engineering/` and the project root `CLAUDE.md`. This catches "works in isolation, breaks the existing project" cases.
- **If the profile is `lightweight`:** explicitly instruct the reviewer to confirm the task introduced **no executable runtime behavior** (and is therefore correctly classified). If the reviewer finds runtime logic that warrants tests, it must say so — the task is then **upgraded to `full`** (see Step E).

Append its report to `log.md` under "Iteration 1 → Review".

**Step E — Decide.** Per execution-methodology §1.2:
- If the reviewer reports issues → go to Fix iteration.
- **If the profile was `lightweight` and the reviewer says the task actually needs tests** (it introduced testable runtime behavior) → **upgrade to `full`**: note the upgrade in `log.md`, dispatch the `tester` (Step B) now, and only proceed once it reports green and the reviewer re-approves. Do not commit a mis-classified lightweight task without tests.
- Otherwise → go to Completion.

### Fix iteration

Per execution-methodology §2.3 steps 5-8:

1. Add a new "Iteration N+1" section to `log.md`.
2. Dispatch a **new** `implementer` subagent (fresh context) with:
   - The brief path
   - The specific failures or review issues to address (be precise — quote the test names or issue numbers)
   - An instruction to make targeted fixes only, no unrelated changes
3. Append its fix report to `log.md` under "Iteration N+1 → Fix".
4. Re-run the test stage: full profile → re-dispatch the tester and append to "Iteration N+1 → Test"; lightweight profile → re-run the type-check + existing suite yourself and append to "Iteration N+1 → Check (lightweight)".
5. If green, re-dispatch the reviewer. Append to "Iteration N+1 → Review".
6. Loop until reviewer approves — subject to execution-methodology §1.3 (iteration cap of 5).

### Iteration cap

Per execution-methodology §1.3: the dev loop is bounded at **5 iterations**. If iteration 5 ends without reviewer approval, stop and follow the escalation protocol (execution-methodology §3). Record an Escalation section in `log.md` and surface concrete options to the user:
- Revise the brief (the planner's spec for this task may be wrong)
- Revise upstream design (the engineering spec / UX spec may be wrong)
- Defer to `docs/known-issues.md` and proceed (only acceptable for non-blocking issues)
- Force-approve and commit (only if the user explicitly decides the recurring finding is bikeshedding)

Wait for the user's decision before any further dispatch.

### Completion

When the reviewer approves:

1. **Verification.** Per execution-methodology §4: run tests and type-check one more time via Bash to capture clean evidence. Paste the output into the "Completion" section of `log.md`. Walk through every acceptance criterion in the brief and tick it off with how it was verified.

2. **Confirm with the user before committing.** Show them: scope ID, task number, files touched, iteration count, and the proposed commit message. Wait for their go-ahead.

3. **Commit.** Per execution-methodology §1.4. Use a message like:
   - Project scope: `Task NN: <one-line summary>`
   - Feature scope: `feat(<feature-name>): task NN — <one-line summary>`
   The commit message should NOT include AI-generated trailers unless the user has asked for them.

4. **Update `log.md`.** After the commit lands, update the "Completion → Commit" line with the actual SHA.

5. **Write the task completion marker.** Create a `completion.md` file in the task directory (same directory as `brief.md` and `log.md`). Use the template at `docs/plan/task-completion-template.md` (or `${CLAUDE_PLUGIN_ROOT}/templates/task-completion-template.md` if the project copy is missing). Populate the YAML frontmatter explicitly:

   ```yaml
   ---
   status: complete
   commit: <actual commit SHA>
   completedAt: <ISO-8601 timestamp, e.g., 2026-05-14T18:23:00+08:00>
   iterations: <integer N>
   ---
   ```

   Replace the placeholder name in the H1 heading with the actual task name. The body is human prose only — do not duplicate frontmatter fields into the body. This file is required by the implement phase's `outputCheck`; without it, `/phased-dev:advance-phase` cannot verify the implement phase is complete.

6. **Update status mirror** (project: `docs/STATUS.md`, feature: `docs/features/<name>/STATUS.md`):
   - "Last Completed": add a line for this task with the commit SHA
   - "What's Next": update to point at the next task, or "All tasks complete" if this was the last one
   - **Task Progress table:** the table was initialized by `/phased-dev:advance-phase` when entering the implement phase (all tasks "pending"). Update the current task's row to "done (`<SHA>`)" and the next task's row to "in progress". If the table doesn't exist, do **not** reconstruct it inline — that path was a known drift surface. Stop and tell the user the implement phase was entered without going through `/phased-dev:advance-phase`; they should run `/phased-dev:advance-phase` (or, if already in implement, regenerate the status mirror by re-running it after a no-op rewind) before continuing. The table format is owned by `advance-phase`; only `advance-phase` writes it.

7. **Check if all tasks are done.** Use filesystem globs, not plan-markdown parsing:
   - `briefs = glob('<paths.tasks>task-*/brief.md')` — total task count
   - `completions = glob('<paths.tasks>task-*/completion.md')` — completed task count

   If every task directory containing a `brief.md` also contains a `completion.md`, tell the user:
   > "All tasks in scope `<id>` are complete. You can now run `/phased-dev:advance-phase` to finalize the `implement` phase."

   Otherwise, list the task directories that have a `brief.md` but no `completion.md` (these are the remaining tasks) and tell the user which one to run next — typically the lowest-numbered remaining task.

## Escalation

Per execution-methodology §3. If the implementer or tester surfaces a cross-boundary problem (a library API mismatch, a missing upstream interface, a perf issue requiring architectural changes):

1. **Stop the loop.** Do not dispatch more agents.
2. Add an "Escalation" section to `log.md` capturing what broke, why, and what's affected.
3. Surface to the user with the specifics and proposed options. Wait for their decision.

## Reporting

While the loop runs, give the user concise progress updates: which agent you're dispatching, what it reported (1-2 sentences), and what's next. Do not flood the chat with full subagent reports — those go in `log.md`.
