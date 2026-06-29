---
description: Dispatch the agent for the active scope's current phase. Replaces /start-brainstorm, /start-design, /start-plan, /plan-feature.
argument-hint: "[optional context for the agent — e.g., initial idea, constraints]"
---

# Start Current Phase

User invoked with: $ARGUMENTS

You are dispatching the agent that owns the **active scope's current phase**. The active scope and pipeline are stored in `docs/.phased-dev/`. Read them first; do not assume.

## Pre-flight checks

1. **Confirm scope state exists.** If `docs/.phased-dev/state.json` is missing, tell the user to run `/phased-dev:init-project` first and stop.

2. **Read the active scope.**
   - Read `docs/.phased-dev/state.json` → get `activeScope`
   - Read `docs/.phased-dev/scopes/<activeScope>.json` → get `currentPhase`, `phaseStatus`, `pipeline`, `paths`

3. **Locate the pipeline entry** for `currentPhase`. It contains: `phase`, `agent`, `outputCheck`.

4. **If `agent` is null** (this is the `implement` phase), stop and tell the user:
   > "The active scope `<id>` is in the `implement` phase. Use `/phased-dev:start-task <NN>` to run the dev loop on a specific task, not `/phased-dev:start-phase`."

5. **Check `phaseStatus`:**
   - `complete_awaiting_approval` — tell the user this phase already produced output and is awaiting their approval. Suggest `/phased-dev:advance-phase` or, if they want to redo the work, ask them to confirm before re-dispatching.
   - `in_progress` — warn the user that an agent was previously dispatched; ask if they want to re-dispatch (this discards any partial work in the agent's context, not on disk).
   - `not_started` — proceed.

6. **Verify upstream output exists.** For every pipeline entry *before* the current phase, confirm its `outputCheck` matches the filesystem. `outputCheck` may be a single glob string or an array of globs — when an array, **every** glob must match at least one file. If any check fails, tell the user that an upstream phase's output is missing and stop.

## Dispatch

1. **Mark in-progress.** Set `phaseStatus` to `in_progress` in the active scope file before dispatching. (You are the writer of scope state — agents only read.)

2. **Validate the agent.** Confirm the agent name from the pipeline entry corresponds to an actual agent file at `${CLAUDE_PLUGIN_ROOT}/agents/<agent-name>.md`. If the file doesn't exist, stop and tell the user the pipeline references a non-existent agent — the scope JSON may be corrupted or a custom template has a typo.

3. **Launch the agent.** Use the Agent tool to launch the subagent named in the pipeline entry. Pass it:
   - The active scope ID
   - An instruction: "Read `docs/.phased-dev/scopes/<id>.json` for your output paths (`paths` field). Produce your output files there. Do NOT modify the scope JSON or `docs/project/STATUS.md` — the orchestrator handles state."
   - Any user-supplied context (`$ARGUMENTS`)

4. **When the agent returns:**

   - **If it asked questions:** relay them to the user verbatim, wait for answers, then re-dispatch. **Critical:** the new agent instance has no memory of the previous round — you must embed the user's verbatim answers in the re-dispatch prompt along with the original task context. Example: "The user's answers to your questions: [paste answers]. Now produce your output."

   - **If it reported completion:**
     1. Verify the pipeline entry's `outputCheck` now matches the filesystem (single glob or array — when an array, every glob must match at least one file). If not, the agent thought it was done but produced nothing (or only part of it — e.g., the plan file without any brief) — report this to the user and leave `phaseStatus` as `in_progress`.
     2. Set `phaseStatus` to `complete_awaiting_approval` in the active scope file.
     3. Regenerate the status mirror. For project scope, update `docs/project/STATUS.md`. For feature scope, update `docs/feature/<name>/STATUS.md`.
     4. Tell the user what was produced (paths from `paths` field) and remind them to run `/phased-dev:advance-phase` after review.

## Notes

- This command knows nothing about specific phases (`brainstorm` vs `ux` vs `engineering` vs `plan`). It only knows: "look up the pipeline, dispatch the named agent." Phase-specific instructions live in the agent prompts.
- For the `implement` phase, the per-task loop is handled by `/phased-dev:start-task`. This command refuses to dispatch in that case (step 4).
- **Writer discipline:** commands write scope state; agents read it. This keeps the state machine in one place. If you find yourself wanting an agent to update the JSON, push the update into the command that dispatched it.
