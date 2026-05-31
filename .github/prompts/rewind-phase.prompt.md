---
description: Rewind the active scope to a previous phase. Resets phase status and optionally cleans up partial output.
argument-hint: "<target-phase> (e.g., brainstorm, engineering, plan)"
---

# Rewind Phase

User invoked with target phase: $ARGUMENTS

This command rewinds the active scope to a previous phase. Use this when the current phase's output needs rework based on downstream discoveries (e.g., the plan reveals an architectural issue that requires redoing the engineering spec).

## Pre-flight checks

1. **Read scope state.** Read `docs/.phased-dev/state.json`. If missing, tell the user to run `/init-project` first and stop.

2. **Read the active scope.** Get `activeScope`, then read `docs/.phased-dev/scopes/<activeScope>.json`. Note `currentPhase`, `phaseStatus`, `pipeline`, and `history`.

3. **Validate the target phase.** The target phase (`$ARGUMENTS`) must exist in the scope's `pipeline` array. If it doesn't match any entry, tell the user the valid phase names for this scope and stop.

4. **Confirm the target is upstream.** The target phase must be earlier in the pipeline than `currentPhase`. You cannot rewind forward. If the target is the same as or later than `currentPhase`, tell the user and stop.

5. **Confirm with the user.** Show them:
   - Current phase and status
   - Target phase
   - What will be lost: all `history` entries from the target phase onward
   - What output files exist for phases between target and current (these may become orphaned)

   Ask: "Are you sure you want to rewind from `<currentPhase>` to `<targetPhase>`? This will discard phase history from `<targetPhase>` onward. Reply 'yes' to proceed."

   Wait for explicit confirmation. Do not proceed on ambiguous responses.

## Rewind

On user approval:

1. **Truncate history.** Remove all entries from `history` that are at or after the target phase. Keep entries before the target phase intact.

2. **Update scope state:**
   - Set `currentPhase` to the target phase
   - Set `phaseStatus` to `not_started`

3. **Write the scope file back.**

4. **Update the status mirror.** Regenerate the human-readable mirror for the *active* scope (not just project):
   - Project scope → `docs/STATUS.md`
   - Feature scope → `docs/features/<name>/STATUS.md`

   The mirror must reflect the rewound state: new Current Phase, updated What's Next, and Last Completed re-derived from the truncated `history`. **If the original `currentPhase` was `implement`** (i.e., the rewind crosses out of the implement phase), also remove the Task Progress table from the mirror — that table referred to a task list that may no longer be valid post-rewind. Per-task `completion.md` files on disk are handled in step 5.

5. **Handle orphaned artifacts from abandoned phases.** For every pipeline entry **after the target phase, up to and including the original `currentPhase`** — i.e., every phase the scope is leaving behind, whether crossed in passing or simply the one it was in — enumerate the files matching that entry's `outputCheck` (string or array of globs). These are orphaned: they will silently satisfy `outputCheck` the next time the user reaches that phase via `advance-phase`, making it appear as if new work was done. The target phase itself is preserved (the user is rewinding *to* it to revise, not to abandon it).

   If `task-*/brief.md` appears under both `plan` and `implement` (because both list it in their `outputCheck`), group it under `plan` only — that is the phase that produced it. This avoids asking the user about the same files twice.

   For each group of orphaned files, show the user:
   - The phase they belong to
   - The file paths
   - The risk: if kept, these files will satisfy `outputCheck` the next time the user reaches that phase via `advance-phase`, making it appear as if new work was done

   Ask the user explicitly for **each group**: "Delete these orphaned `<phase>` artifacts? If you keep them, the `<phase>` phase will appear already-complete the next time you reach it." Delete only on explicit confirmation. Default is to keep them as reference.

   If there are no orphaned files in any abandoned phase, skip this step silently.

6. **Report to the user:**
   - Which phase the scope is now in
   - The outcome of step 5 — which orphaned artifacts were deleted vs. kept, grouped by phase. If no orphans were found, note that.
   - What command to run next:
     - If the target phase's `agent` is non-null → `/start-phase`
     - If the target phase is `implement` → `/start-task <NN>`

## Notes

- This command does **not** delete output files from rewound phases. The user may want to reference them when redoing work. If they want a clean slate, they can delete the files manually.
- Rewinding to a phase that has no output files yet is valid — it's essentially restarting from that phase.
- There is no "rewind one phase" shorthand — always specify the target phase explicitly to avoid accidental rewinds.
- This command modifies the active scope's JSON and its status mirror (`docs/STATUS.md` for project scope; `docs/features/<name>/STATUS.md` for feature scope). It does not touch `state.json` or other scopes.
