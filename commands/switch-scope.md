---
description: Change the active scope. All subsequent commands operate on the new scope.
argument-hint: "<scope-id> (e.g., project, feature/auth-revamp)"
---

# Switch Active Scope

User invoked with: $ARGUMENTS

## Steps

1. **Parse arguments.** `$ARGUMENTS` should be a scope ID. Valid forms:
   - `project`
   - `feature/<name>`

   If empty, list available scopes (read `docs/.phased-dev/state.json` → `scopes`) and ask the user which one.

2. **Confirm scope state exists.** If `docs/.phased-dev/state.json` is missing, tell the user to run `/phased-dev:init-project` first and stop.

3. **Verify the target scope exists.** Read `state.json` → `scopes`. If the requested ID is not in the list, stop and tell the user. Suggest:
   - For project: nothing (it should exist after init).
   - For feature: `/phased-dev:start-feature <name>` to create it.

4. **Update `state.json`:** set `activeScope` to the requested ID. Write the file back.

5. **Report.** Read the new active scope file and tell the user:
   - The scope ID and type
   - Its current phase and phase status
   - What command to run next (based on current phase — `/start-phase` for non-implement, `/start-task NN` for implement)

## Notes

- This command does not modify any scope file — it only changes the pointer in `state.json`.
- The scope you switched away from keeps all its state. Switching back later resumes exactly where it was.
