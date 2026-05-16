---
description: List all scopes in this project with their current phase, status, and which one is active.
---

# List Scopes

## Steps

1. **Confirm scope state exists.** If `docs/.phased-dev/state.json` is missing, tell the user to run `/phased-dev:init-project` first.

2. **Read `state.json`:**
   - `activeScope` — the currently active scope ID
   - `scopes` — list of all scope IDs

3. **For each scope ID in `scopes`,** read `docs/.phased-dev/scopes/<id>.json` and extract:
   - `id`
   - `type`
   - `currentPhase`
   - `phaseStatus`

4. **Print a table** with one row per scope. Mark the active scope with a leading `*`. Example:

   ```
   * project              | plan       | complete_awaiting_approval
     feature/auth-revamp  | design     | in_progress
     feature/billing      | implement  | in_progress
   ```

5. **Suggest a next action** based on the active scope's phase:
   - `not_started` or `in_progress` (non-implement) → `/phased-dev:start-phase`
   - `complete_awaiting_approval` → `/phased-dev:advance-phase`
   - `implement` phase → `/phased-dev:start-task <NN>`

Keep the whole output to ~10 lines.
