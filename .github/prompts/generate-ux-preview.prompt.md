---
description: Generate a static HTML preview from the active scope's UX markdown spec. Independent of phase gates — can be run any time after the UX spec exists.
---

# Generate UX Preview

You are generating a static HTML preview for the user to visually validate the UX design in a browser. This is independent of phase gates — you can run this any time after the UX markdown spec exists.

## Pre-flight checks

1. **Read scope state.** Read `docs/.phased-dev/state.json`. If missing, tell the user to run `/init-project` first and stop.

2. **Read the active scope.** Get `activeScope`, then read `docs/.phased-dev/scopes/<activeScope>.json`. Note `paths.uxDir` and `type`.

3. **Verify this scope has a UX phase.** If `paths.uxDir` is absent from the scope JSON, stop — this scope doesn't have a UX phase. Only `project-design-heavy` and `feature-design-heavy` scopes have one.

4. **Verify the markdown spec exists.** Glob `<paths.uxDir>*.md`. If no markdown files exist, stop — the UX spec hasn't been written yet. Tell the user to run `/start-phase` for the `ux` phase first (or, if the scope is past the `ux` phase, the spec may have been deleted).

5. **Check for existing preview.** If `<paths.uxDir>preview/` already exists, tell the user: "A preview already exists at `<path>`. Re-running will overwrite it." Ask for confirmation before proceeding.

## Dispatch

Delegate to the `ux-preview` agent (via the `agent` tool) with:
- The active scope ID
- An instruction to read `docs/.phased-dev/scopes/<id>.json` for `paths.uxDir` and `type`
- An instruction to read the most recent markdown spec under `paths.uxDir`
- An instruction to generate the HTML preview under `paths.uxDir>preview/`

## After dispatch

When the agent returns:
- If it produced files, list them and tell the user to open `preview/index.html` in a browser.
- If it failed (partial output, context limit, tool error), report what was produced and note the user can re-run this command to retry. The markdown spec is unaffected.

## Notes

- This command does **not** modify scope state (`currentPhase`, `phaseStatus`, `history`). It's a utility command, not a phase transition.
- The HTML preview is for human review only. Downstream agents (architect, planner, reviewer) ignore it. The markdown spec is the only binding artifact.
- The preview can be regenerated at any time — it overwrites the previous `preview/` directory.
