---
description: Create a new feature scope, scaffold its directory, set it as active, and dispatch the first-phase agent. Asks which pipeline (standard or design-heavy).
argument-hint: "<feature-name> [\"one-line description\"]"
---

# Start Feature — Create Scope and Dispatch First Phase

User invoked with: $ARGUMENTS

This creates a **feature scope** for adding a feature to an existing project. Two pipeline options:

- **`standard`** (default) — 4 phases: `research → engineering → plan → implement`. The feature-architect produces a combined product + engineering spec. Right for most features.
- **`design-heavy`** — 5 phases: `research → ux → engineering → plan → implement`. Adds a dedicated UX phase (the ux-designer produces a full UX spec — design language scoped to the feature, components, screens, flows, a11y, microcopy), then the feature-architect produces the engineering spec against that. Right for UX-led features like onboarding rewrites, editor modes, search redesign, settings overhauls.

Both pipelines open with a `research` phase (prior art, feasibility, how the feature fits the existing codebase) that feeds the engineering (or UX) phase. For a feature on well-trodden patterns, the researcher can write a short "no external research needed" doc to clear the gate.

## Pre-flight checks

1. **Parse arguments.** First token = feature name. Remaining tokens = optional one-line description. If feature name is missing, ask the user.

1b. **Validate feature name.** The feature name must match `^[a-z0-9][a-z0-9-]*$` (lowercase alphanumeric, hyphens allowed, must start with alphanumeric). Reject names with spaces, uppercase, slashes, dots, or other special characters. This prevents path-traversal footguns and filesystem issues. Examples: `auth-revamp` ✅, `billing-v2` ✅, `my feature` ❌, `Foo` ❌, `../../etc` ❌.

2. **Confirm the project is initialized.** `docs/.phased-dev/state.json` and `docs/.phased-dev/scopes/project.json` must exist. If not, tell the user to run `/phased-dev:init-project` first (or, if applying feature mode to an unrelated project, scaffold those files manually).

3. **Confirm the project has an engineering spec** — at least one file under `docs/project/engineering/`. If missing, warn the user: the feature-architect can still proceed but will have to infer architecture from code, which is lower quality.

4. **Check for existing feature scope.** If `docs/.phased-dev/scopes/feature/<feature-name>.json` or `docs/feature/<feature-name>/` already exists, ask the user: (a) revising the existing design, (b) starting over (delete first, then re-run), or (c) wrong name. Do not silently overwrite.

5. **Ask which pipeline to use.** Present `standard` vs `design-heavy` (use the AskUserQuestion tool if available, otherwise ask plainly). If the user replies "skip" or just confirms the default, use `standard`. Recommend `standard` unless the feature is clearly UX-led.

## Scaffold

1. **Create directories:**
   - `docs/feature/<feature-name>/`
   - `docs/feature/<feature-name>/research/` (the `research` phase runs first in both pipelines)
   - `docs/feature/<feature-name>/engineering/` (the feature-architect writes the engineering spec here)
   - `docs/feature/<feature-name>/plan/` (the planner writes implementation-plan.md here)
   - `docs/feature/<feature-name>/tasks/` (empty for now)
   - `docs/.phased-dev/scopes/feature/` (if not present)
   - **If pipeline = `design-heavy`, also create:** `docs/feature/<feature-name>/ux/`

2. **Create the feature scope config.** Pick the template based on pipeline:

   | If pipeline | Source | Destination |
   |---|---|---|
   | `standard` | `${CLAUDE_PLUGIN_ROOT}/templates/scopes/feature.json` | `./docs/.phased-dev/scopes/feature/<feature-name>.json` |
   | `design-heavy` | `${CLAUDE_PLUGIN_ROOT}/templates/scopes/feature-design-heavy.json` | `./docs/.phased-dev/scopes/feature/<feature-name>.json` |

   Substitute `{{FEATURE_NAME}}` with the parsed name. The scope ID is `feature/<feature-name>` in both cases; the pipeline difference is recorded in the JSON's `type` field (`feature` vs `feature-design-heavy`).

3. **Copy feature directory templates:**
   - `${CLAUDE_PLUGIN_ROOT}/templates/feature-engineering-template.md` → `docs/feature/<feature-name>/engineering/engineering.md` (skeleton inside the engineering folder; the feature-architect rewrites it or replaces it with a dated spec)
   - `${CLAUDE_PLUGIN_ROOT}/templates/decisions.md` → `docs/feature/<feature-name>/decisions.md` (feature-scoped decision log)
   - `${CLAUDE_PLUGIN_ROOT}/templates/feature-status.md` → `docs/feature/<feature-name>/STATUS.md`. Substitute both placeholders:
     - `{{FEATURE_NAME}}` → the parsed feature name
     - `{{INITIAL_PHASE}}` → the first phase of the chosen pipeline (`research` for both `standard` and `design-heavy`). Do not hardcode — read it from the scope JSON's `pipeline[0].phase` you just wrote. STATUS.md must agree with the scope JSON's `currentPhase`.

4. **Register and activate the scope.** Update `docs/.phased-dev/state.json`:
   - Add `"feature/<feature-name>"` to `scopes` (if not already present)
   - Set `activeScope` to `"feature/<feature-name>"`

## Dispatch

The agent for the first phase comes from the scope's `pipeline[0].agent`. Both pipelines now open with the `research` phase, so dispatch `researcher` in both cases (the `feature-architect` and, for design-heavy, the `ux-designer` come later, after `/phased-dev:advance-phase`).

Pass the agent:

- The active scope ID: `feature/<feature-name>`
- An instruction to read `docs/.phased-dev/scopes/feature/<feature-name>.json` for its output paths and to confirm `currentPhase` matches its role
- The user's one-line description ($ARGUMENTS)
- An instruction to read the project's `CLAUDE.md`, the latest engineering spec under `docs/project/engineering/`, the project-level `docs/project/decisions.md` (if present, for cross-cutting constraints), and to Glob/Grep for code the feature is likely to touch — research is grounded in the existing system
- An instruction to ask 2-4 sharpening questions if the brief is too vague to research, before sweeping

Set `phaseStatus` to `in_progress` in the scope file before dispatching. After the agent returns done, set it to `complete_awaiting_approval` (the orchestrator owns scope-state writes — agents do not).

## After dispatch

When the agent returns:

- If it asked questions, relay them to the user verbatim and wait for answers, then re-dispatch (scope stays `in_progress`). **Critical:** the new agent instance has no memory of the previous round — you must embed the user's verbatim answers in the re-dispatch prompt along with the original task context.
- If the work is done, summarize the output path and headline decisions. Tell the user to review and run `/phased-dev:advance-phase` to move to the next phase, then `/phased-dev:start-phase` to dispatch the next agent.

## Notes

- Switching to a different scope later: `/phased-dev:switch-scope project` (or any other scope ID).
- The feature scope operates independently from the project scope's phase machinery. Different scopes have different `currentPhase` values that progress independently.
- **Switching pipelines later:** there is no migration command. To switch from `standard` to `design-heavy` (or vice-versa) before any phase work has started, delete the feature scope JSON and re-run this command.
