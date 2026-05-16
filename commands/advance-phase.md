---
description: Gated phase transition — advance the active scope to the next phase only after the user explicitly approves the current phase's output
---

# Advance Phase

This is a **gated** transition. Phases do not advance silently. The user must explicitly approve the current phase's output before you update the active scope's state.

## Steps

1. **Read the active scope.**
   - Read `docs/.phased-dev/state.json` → get `activeScope`. If `state.json` doesn't exist, tell the user to run `/phased-dev:init-project` and stop.
   - Read `docs/.phased-dev/scopes/<activeScope>.json` → get `currentPhase`, `phaseStatus`, `pipeline`, `paths`, `history`.

2. **Determine the next phase.** Find `currentPhase` in `pipeline`. The next phase is the entry immediately after it.
   - If `currentPhase` is the last entry in `pipeline`, this scope has no further advancement. Tell the user the workflow for this scope is complete.

3. **Verify phase status.** `phaseStatus` must be `complete_awaiting_approval`. If not:
   - `not_started` → tell the user to run `/phased-dev:start-phase` first
   - `in_progress` → tell the user the current phase's agent hasn't reported completion yet
   - `complete_awaiting_approval` → proceed
   - `complete` → already advanced; nothing to do

4. **Verify the current phase's output exists.** Use the current pipeline entry's `outputCheck` to verify the filesystem. `outputCheck` may be a single glob string or an array of globs — when an array, **every** glob must match at least one file. If any check fails, stop and tell the user precisely which expected output is missing (e.g., for the plan phase: "plan file exists but no task briefs were written").

   **Implement-phase additional check (pairing).** The default `outputCheck` semantics only require "every glob matches at least one file," which is too weak for the implement phase: 10 task briefs with a single `completion.md` would silently pass. So when `currentPhase` is `implement`, additionally verify that **every task directory has both `brief.md` and `completion.md`**:
   - Glob `<paths.tasks>task-*/brief.md` and `<paths.tasks>task-*/completion.md`.
   - For each task directory that contains a `brief.md`, confirm the same directory contains a `completion.md`.
   - If any task directory has a brief but no completion, list the missing task names (e.g., `task-03-foo`, `task-07-bar`) and stop. Tell the user to either `/phased-dev:start-task <NN>` those tasks, or, if they've decided to defer them, document that decision and remove the brief — partial-but-claimed-complete is not a valid state.

5. **Show the user the output and ask for approval.** List the files produced in the current phase (with paths) and ask:

   > "Have you reviewed the `<currentPhase>` output for scope `<activeScope>` and are you ready to advance to `<nextPhase>`? Reply 'yes' to proceed, or describe what needs to change."

   Wait for an explicit yes. Do not proceed on ambiguous responses.

6. **On approval, update the scope file:**
   - Append `{ "phase": "<currentPhase>", "completedAt": "<ISO timestamp>", "approvedAt": "<ISO timestamp>" }` to `history`
   - Set `currentPhase` to the next phase's name
   - Set `phaseStatus` to `not_started`

   Write the scope file back.

7. **Update status mirror** (project: `docs/STATUS.md`, feature: `docs/features/<name>/STATUS.md`):
   - Current Phase, Last Completed, What's Next.
   - **If the new phase is `implement`:** initialize the Task Progress table. Read `paths.plan` from the scope JSON to get the full task list, then populate the table with all tasks as "pending":
     ```
     | # | Task | Status |
     |---|------|--------|
     | 01 | task-name | pending |
     | 02 | task-name | pending |
     | ... | ... | ... |
     ```
     This gives the user a clear view of the work ahead before they run the first task.

8. **Tell the user** which phase they're now in and what command to run next:
   - If the new phase's `agent` is non-null → `/phased-dev:start-phase`
   - If the new phase is `implement` → `/phased-dev:start-task <NN>`

## Notes

- Never advance without an explicit "yes" from the user. "Looks good" is enough; a follow-up question is not.
- This command never modifies `state.json` — only the active scope's file (and `STATUS.md` if project scope). To switch active scope, use `/phased-dev:switch-scope`.
- To **go back** a phase (e.g., redo design after seeing the plan), use `/phased-dev:rewind-phase <target-phase>`. This safely resets the scope state and truncates history.
