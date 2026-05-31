---
description: Scaffold the phased-dev structure (state.json, project scope, methodologies, templates, AGENTS.md) into the current project. Asks the user which pipeline to use.
argument-hint: "[project-name] [\"one-line description\"]"
---

# Initialize Phased-Dev Project

You are scaffolding the phased-dev structure into the current working directory. The user invoked this with: $ARGUMENTS

## Steps

1. **Parse arguments.** If `$ARGUMENTS` is non-empty, treat the first token as the project name and the rest (joined) as the one-line description. If empty or partial, ask the user for what's missing.

2. **Check for existing scaffolding.** If any of `AGENTS.md`, `docs/STATUS.md`, `docs/.phased-dev/state.json`, or `docs/plan/planning-methodology.md` already exist, **stop and ask the user** before overwriting. Do not silently overwrite.

3. **Ask which pipeline to use.** Present the choice to the user and wait for their answer (ask the user directly and wait for their reply):

   - **`standard`** (default) — 4 phases: `brainstorm → engineering → plan → implement`. Right for most projects: backend services, tooling, CLIs, dev tools, infra, anything where UX is light or naturally fits inside the engineering phase.
   - **`design-heavy`** — 5 phases: `brainstorm → ux → engineering → plan → implement`. Adds a dedicated `ux` phase that produces a full UX spec (design language, component inventory, screen wireframes, accessibility contract, microcopy library) before the engineering phase. Right for consumer apps, editors, design tools, marketing sites — anything where UX *is* the differentiator.

   If the user replies "skip" or just confirms the default, use `standard`. Recommend `standard` unless the project is clearly design-driven.

4. **Create directories** (idempotent):
   - `docs/`
   - `docs/.phased-dev/`
   - `docs/.phased-dev/scopes/`
   - `docs/brainstorm/`
   - `docs/engineering/`
   - `docs/plan/`
   - `docs/tasks/`
   - **If pipeline = `design-heavy`, also create:** `docs/ux/`

5. **Copy templates.** The phased-dev templates live at `templates/` (relative to the repo root where phased-dev's `.github/` and `templates/` directories are installed). If `templates/` is not present, tell the user to copy the phased-dev `templates/` directory into their repo and stop. Substitute `{{PROJECT_NAME}}` and `{{ONE_LINE_DESCRIPTION}}` in `AGENTS.md` only.

   **Common (both pipelines):**

   | Source | Destination |
   |--------|-------------|
   | `templates/CLAUDE.md` | `./AGENTS.md` (substitute placeholders) |
   | `templates/STATUS.md` | `./docs/STATUS.md` |
   | `templates/state.json` | `./docs/.phased-dev/state.json` |
   | `templates/scope-model.md` | `./docs/.phased-dev/scope-model.md` |
   | `templates/decisions.md` | `./docs/decisions.md` |
   | `templates/known-issues.md` | `./docs/known-issues.md` |
   | `templates/planning-methodology.md` | `./docs/plan/planning-methodology.md` |
   | `templates/execution-methodology.md` | `./docs/plan/execution-methodology.md` |
   | `templates/log-template.md` | `./docs/plan/log-template.md` |
   | `templates/task-brief-template.md` | `./docs/plan/task-brief-template.md` |
   | `templates/plan-template.md` | `./docs/plan/plan-template.md` |
   | `templates/task-completion-template.md` | `./docs/plan/task-completion-template.md` |

   **Pipeline-specific scope template:**

   | If pipeline | Source | Destination |
   |---|---|---|
   | `standard` | `templates/scopes/project.json` | `./docs/.phased-dev/scopes/project.json` |
   | `design-heavy` | `templates/scopes/project-design-heavy.json` | `./docs/.phased-dev/scopes/project.json` |

   Note: the destination is `project.json` regardless of pipeline — the scope ID is always `project` for the project-level scope. The pipeline type is recorded in the scope JSON's `type` field (`project` vs `project-design-heavy`).

   Read each template, then write it to the destination. Do this in parallel where possible.

6. **Report.** Tell the user:
   - Which files were created
   - Which pipeline was selected and what its phase sequence is
   - That the `project` scope is active, currently in phase `brainstorm` (not started)
   - Suggest: run `/start-phase` to begin the brainstorm phase

## Notes

- Do not create `.gitignore`, `package.json`, or any code files. This command only scaffolds the *methodology* docs and scope state. The first task in the `implement` phase will scaffold the actual project.
- The `docs/.phased-dev/` directory is the machine-readable source of truth for phase/scope state. `docs/STATUS.md` is the human-readable mirror — agents that need state should read the JSON, not the markdown.
- If the current directory is not a git repo, mention this in the final report — the user may want to `git init` before proceeding.
- **Switching pipelines later:** there is no migration command. To switch from `standard` to `design-heavy` (or vice-versa) before any phase work has started, the user can delete `docs/.phased-dev/scopes/project.json` and re-run this command. Switching after brainstorm has been written requires manual scope-JSON editing — flag this if the user asks.
