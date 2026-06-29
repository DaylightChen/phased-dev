---
description: Scaffold the phased-dev structure (state.json, project scope, methodologies, templates, CLAUDE.md) into the current project. Asks the user which pipeline to use.
argument-hint: "[project-name] [\"one-line description\"]"
---

# Initialize Phased-Dev Project

You are scaffolding the phased-dev structure into the current working directory. The user invoked this with: $ARGUMENTS

## Steps

1. **Parse arguments.** If `$ARGUMENTS` is non-empty, treat the first token as the project name and the rest (joined) as the one-line description. If empty or partial, ask the user for what's missing.

2. **Check for existing scaffolding.** If any of `CLAUDE.md`, `docs/project/STATUS.md`, `docs/.phased-dev/state.json`, or `docs/methodology/planning-methodology.md` already exist, **stop and ask the user** before overwriting. Do not silently overwrite.

3. **Ask which pipeline to use.** Present the choice to the user and wait for their answer (use the AskUserQuestion tool if available, otherwise ask plainly):

   - **`standard`** (default) — 5 phases: `research → brainstorm → engineering → plan → implement`. Right for most projects: backend services, tooling, CLIs, dev tools, infra, anything where UX is light or naturally fits inside the engineering phase.
   - **`design-heavy`** — 6 phases: `research → brainstorm → ux → engineering → plan → implement`. Adds a dedicated `ux` phase that produces a full UX spec (design language, component inventory, screen wireframes, accessibility contract, microcopy library) before the engineering phase. Right for consumer apps, editors, design tools, marketing sites — anything where UX *is* the differentiator.

   Both pipelines now open with a `research` phase (evidence-gathering: prior art, feasibility, domain constraints) that feeds the brainstorm. For a project building on well-trodden patterns, the researcher can write a short "no external research needed" doc to satisfy the phase gate quickly.

   If the user replies "skip" or just confirms the default, use `standard`. Recommend `standard` unless the project is clearly design-driven.

4. **Create directories** (idempotent):
   - `docs/`
   - `docs/.phased-dev/`
   - `docs/.phased-dev/scopes/`
   - `docs/project/research/`
   - `docs/project/brainstorm/`
   - `docs/project/engineering/`
   - `docs/project/plan/`
   - `docs/methodology/`
   - `docs/templates/`
   - `docs/project/tasks/`
   - **If pipeline = `design-heavy`, also create:** `docs/project/ux/`

5. **Copy templates from the plugin.** The plugin's templates live at `${CLAUDE_PLUGIN_ROOT}/templates/`. Substitute `{{PROJECT_NAME}}` and `{{ONE_LINE_DESCRIPTION}}` in `CLAUDE.md` only.

   **Common (both pipelines):**

   | Source | Destination |
   |--------|-------------|
   | `${CLAUDE_PLUGIN_ROOT}/templates/CLAUDE.md` | `./CLAUDE.md` (substitute placeholders) |
   | `${CLAUDE_PLUGIN_ROOT}/templates/STATUS.md` | `./docs/project/STATUS.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/state.json` | `./docs/.phased-dev/state.json` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/scope-model.md` | `./docs/.phased-dev/scope-model.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/decisions.md` | `./docs/project/decisions.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/known-issues.md` | `./docs/project/known-issues.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/planning-methodology.md` | `./docs/methodology/planning-methodology.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/execution-methodology.md` | `./docs/methodology/execution-methodology.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/log-template.md` | `./docs/templates/log-template.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/task-brief-template.md` | `./docs/templates/task-brief-template.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/plan-template.md` | `./docs/templates/plan-template.md` |
   | `${CLAUDE_PLUGIN_ROOT}/templates/task-completion-template.md` | `./docs/templates/task-completion-template.md` |

   **Pipeline-specific scope template:**

   | If pipeline | Source | Destination |
   |---|---|---|
   | `standard` | `${CLAUDE_PLUGIN_ROOT}/templates/scopes/project.json` | `./docs/.phased-dev/scopes/project.json` |
   | `design-heavy` | `${CLAUDE_PLUGIN_ROOT}/templates/scopes/project-design-heavy.json` | `./docs/.phased-dev/scopes/project.json` |

   Note: the destination is `project.json` regardless of pipeline — the scope ID is always `project` for the project-level scope. The pipeline type is recorded in the scope JSON's `type` field (`project` vs `project-design-heavy`).

   Use the `Read` tool to read each template, then `Write` to the destination. Do this in parallel where possible.

6. **Report.** Tell the user:
   - Which files were created
   - Which pipeline was selected and what its phase sequence is
   - That the `project` scope is active, currently in phase `research` (not started)
   - Suggest: run `/phased-dev:start-phase` to begin the research phase

## Notes

- Do not create `.gitignore`, `package.json`, or any code files. This command only scaffolds the *methodology* docs and scope state. The first task in the `implement` phase will scaffold the actual project.
- The `docs/.phased-dev/` directory is the machine-readable source of truth for phase/scope state. `docs/project/STATUS.md` is the human-readable mirror — agents that need state should read the JSON, not the markdown.
- If the current directory is not a git repo, mention this in the final report — the user may want to `git init` before proceeding.
- **Switching pipelines later:** there is no migration command. To switch from `standard` to `design-heavy` (or vice-versa) before any phase work has started, the user can delete `docs/.phased-dev/scopes/project.json` and re-run this command. Switching after brainstorm has been written requires manual scope-JSON editing — flag this if the user asks.
