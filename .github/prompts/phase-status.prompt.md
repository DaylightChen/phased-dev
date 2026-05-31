---
description: Show the active scope's current phase, last completed milestone, and what's next. Use /list-scopes to see all scopes.
---

# Phase Status

## Steps

1. **Read scope state.** Read `docs/.phased-dev/state.json`. If missing, tell the user this project hasn't been scaffolded yet and suggest `/init-project`.

2. **Read the active scope** at `docs/.phased-dev/scopes/<activeScope>.json`. Extract:
   - `id`, `type`, `name`
   - `currentPhase`, `phaseStatus`
   - `history` — the last entry indicates what was most recently completed
   - `pipeline` — to compute "what's next"

3. **Print a concise status summary (4-6 lines):**
   - **Active scope** — `<id>` (`<type>`)
   - **Current phase** — `<currentPhase>` (`<phaseStatus>`)
   - **Last completed** — most recent entry from `history`, or `none` if empty
   - **What's next** — based on `phaseStatus`:
     - `not_started` (non-implement) → `/start-phase`
     - `in_progress` (non-implement) → wait for the current agent or re-run `/start-phase`
     - `complete_awaiting_approval` → `/advance-phase`
     - `implement` phase → `/start-task <NN>`
   - **Other scopes** — if `state.json.scopes` has more than one entry, suggest `/list-scopes` to see them

Keep it terse — the user can read the JSON themselves for detail.
