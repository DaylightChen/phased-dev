# phased-dev

> Agent-driven phased development workflow for Claude Code, built on a **scope abstraction**. Each scope declares its own pipeline; commands operate on the active scope.

## What it does

`phased-dev` encodes a documented methodology for building non-trivial software with AI agents. The methodology is unchanged from v0.1; v0.2 reorganizes it around scopes.

A **scope** is a unit of phased work. The shipped scope types:

- **Project, standard** (`project`, `type: "project"`) — a whole greenfield project where UX fits naturally inside the engineering phase. Pipeline: `research → brainstorm → engineering → plan → implement`. Right for backends, tooling, CLIs, dev tools, infra.
- **Project, design-heavy** (`project`, `type: "project-design-heavy"`) — a greenfield project where UX *is* the differentiator. Pipeline: `research → brainstorm → ux → engineering → plan → implement`. Adds a dedicated `ux` phase with a `ux-designer` agent producing a full UX spec (design language, component inventory, screen wireframes, accessibility contract, microcopy library) before the engineering phase. Opt in by choosing `design-heavy` when `/phased-dev:init-project` asks.
- **Feature, standard** (`feature/<name>`, `type: "feature"`) — a feature added to an existing project. Pipeline: `research → engineering → plan → implement` (product framing + engineering collapse into a single spec since the upstream architecture is already given).
- **Feature, design-heavy** (`feature/<name>`, `type: "feature-design-heavy"`) — a UX-led feature (onboarding rewrite, editor mode, search redesign). Pipeline: `research → ux → engineering → plan → implement`. The `ux-designer` extends the existing project design system; the `feature-architect` then produces an engineering-focused spec that references the UX spec. Opt in by choosing `design-heavy` when `/phased-dev:start-feature` asks.

Every pipeline opens with a `research` phase: the `researcher` agent gathers prior art, technical feasibility, and domain constraints as upstream evidence for the phases that follow. It makes no product or engineering decisions. On well-trodden ground a short "no external research needed" doc clears the phase gate.

Each scope's pipeline, current phase, and output paths are recorded in `docs/.phased-dev/scopes/<id>.json`. A single global `docs/.phased-dev/state.json` records which scope is **active**. All commands implicitly operate on the active scope. Use `/phased-dev:switch-scope <id>` to change focus.

Each phase produces durable documentation. Each `implement`-phase task produces an execution log with command output, decisions, and deviations. The user approves before each phase advance.

## Why a scope abstraction

The v0.1 design split commands into project-mode and feature-mode pairs (`start-task` + `start-feature-task`, etc.). That duplicated logic and didn't generalize — adding a third mode (bugfix, refactor) would have meant another copy of every command. v0.2 unifies the commands around scope. A scope's `pipeline` field is data, so new pipelines are configuration, not code.

The methodology principles still hold:

1. **Phase gates** — you can't start coding before there's a plan, and you can't plan before the architecture is settled.
2. **Sub-agent role separation** — the implementer doesn't write tests, the tester doesn't review, the reviewer reads the full plan and catches local-but-globally-wrong decisions.
3. **Evidence over claims** — every task's `log.md` contains real test output, not "all tests pass".

It is heavyweight. Not every project warrants it. Use it for greenfield projects with non-trivial scope, or features that touch 3+ files / cross module boundaries.

## Install

This plugin is not yet on a marketplace. Install locally:

```bash
claude --plugin-dir /path/to/phased-dev
```

Or symlink into your Claude plugins directory.

## Commands

The full command set (10 commands, scope-agnostic):

| Command | Purpose |
|---------|---------|
| `/phased-dev:init-project [name] [description]` | Scaffold the project structure: `docs/`, `docs/.phased-dev/`, project scope, methodology files |
| `/phased-dev:phase-status` | Show the active scope's current phase and what's next |
| `/phased-dev:list-scopes` | Show all scopes in this project with their current phase |
| `/phased-dev:switch-scope <id>` | Change the active scope (e.g., `project` or `feature/auth`) |
| `/phased-dev:start-phase` | Dispatch the agent for the active scope's current phase (for any non-`implement` phase) |
| `/phased-dev:start-task <NN>` | Run the per-task dev loop during the `implement` phase |
| `/phased-dev:advance-phase` | Gated transition to next phase (requires explicit user approval) |
| `/phased-dev:rewind-phase <phase>` | Rewind the active scope to a previous phase (resets status, truncates history) |
| `/phased-dev:generate-ux-preview` | Generate static HTML preview from the UX markdown spec (independent of phase gates) |
| `/phased-dev:start-feature <name> [description]` | Create a new feature scope (asks: standard or design-heavy), scaffold its directory, set it active, dispatch the first-phase agent |

## Subagents

| Agent | Used in | Role |
|-------|---------|------|
| `researcher` | `research` phase (all scopes) | Evidence-gathering: prior art, feasibility, domain constraints, open questions. No product or engineering decisions. |
| `brainstormer` | `brainstorm` phase (project scopes) | Product design |
| `ux-designer` | `ux` phase (design-heavy project and feature scopes) | UX spec: design language, components, screens, flows, a11y contract, microcopy. For feature scope, extends the existing project design system. |
| `ux-preview` | Any time after UX spec exists | Generates static HTML preview (3-7 key screens + tokens page) from the UX markdown spec. For human review only — not consumed by downstream agents. Invoked via `/phased-dev:generate-ux-preview`. |
| `architect` | `engineering` phase (project scopes) | Engineering spec; respects upstream UX spec if present |
| `feature-architect` | `engineering` phase (feature scopes) | Standard variant: combined product + engineering. Design-heavy variant: engineering-focused, referencing the UX spec. |
| `planner` | `plan` phase (all scopes) | Task breakdown; covers UX deliverables when `paths.uxDir` is set |
| `implementer` | `implement` phase (all scopes) | Production code only — no tests |
| `tester` | `implement` phase (all scopes) | Tests + full suite + type-check |
| `reviewer` | `implement` phase (all scopes) | Reviews against brief, full plan, and (for features) project architecture |

Agents read scope state from `docs/.phased-dev/scopes/<id>.json`. They never write to it — commands do.

## Workflow — project scope

```
/phased-dev:init-project myproject "A widget for X"
  → creates docs/.phased-dev/state.json + scopes/project.json
  → active scope: project; current phase: research
  ↓
/phased-dev:start-phase            → researcher gathers prior art, feasibility, constraints
  ↓ (user reviews)
/phased-dev:advance-phase          → currentPhase becomes "brainstorm"
  ↓
/phased-dev:start-phase            → brainstormer drafts product spec (reads research findings)
  ↓ (user reviews)
/phased-dev:advance-phase          → currentPhase becomes "engineering"
  ↓
/phased-dev:start-phase            → architect drafts engineering spec
  ↓ (user reviews)
/phased-dev:advance-phase          → currentPhase becomes "plan"
  ↓
/phased-dev:start-phase            → planner produces task list + briefs
  ↓ (user reviews)
/phased-dev:advance-phase          → currentPhase becomes "implement"
  ↓
/phased-dev:start-task 01          → orchestrator runs dev loop on Task 1
/phased-dev:start-task 02          → ... Task 2 ...
...
```

## Workflow — design-heavy project scope (opt-in)

Choose `design-heavy` when `/phased-dev:init-project` asks. Same shape as standard, with a `ux` phase inserted between `brainstorm` and `engineering` (and `research` first, as in every pipeline):

```
/phased-dev:init-project myapp "A consumer X app"
  → asks: standard or design-heavy?
  → user picks design-heavy
  → creates docs/.phased-dev/state.json + scopes/project.json (type: project-design-heavy)
  → also creates docs/research/ and docs/ux/
  ↓
/phased-dev:start-phase            → researcher gathers prior art, feasibility, constraints
  ↓ (user reviews + /phased-dev:advance-phase)
/phased-dev:start-phase            → brainstormer drafts product spec (reads research findings)
  ↓ (user reviews + /phased-dev:advance-phase)
/phased-dev:start-phase            → ux-designer drafts UX spec (design language, components,
                                      screens, flows, a11y contract, microcopy library)
  ↓ (user reviews + advance)
/phased-dev:start-phase            → architect drafts engineering spec, respecting the UX spec
  ↓ (user reviews + advance)
/phased-dev:start-phase            → planner produces task list; coverage extends to every
                                      UX deliverable (component, screen, microcopy, a11y item)
  ↓ (user reviews + advance)
/phased-dev:start-task 01          → ... dev loop ...
```

The architect and planner both check for `paths.uxDir` on the scope JSON — if present, they treat the UX spec as upstream design and binding constraint. Nothing in commands has to branch on pipeline type.

## Workflow — feature scope (standard)

```
(starting from an existing phased-dev project, active scope = project)

/phased-dev:start-feature my-feature "What it does"
  → asks: standard or design-heavy? user picks standard
  → creates docs/.phased-dev/scopes/feature/my-feature.json (type: feature)
  → sets active scope to feature/my-feature
  → dispatches researcher (the first phase is "research")
  ↓ (user reviews docs/features/my-feature/research/)
/phased-dev:advance-phase          → currentPhase becomes "engineering"
  ↓
/phased-dev:start-phase            → feature-architect drafts engineering.md (reads research findings)
  ↓ (user reviews docs/features/my-feature/engineering.md)
/phased-dev:advance-phase          → currentPhase becomes "plan"
  ↓
/phased-dev:start-phase            → planner produces feature plan + briefs
  ↓ (user reviews)
/phased-dev:advance-phase          → currentPhase becomes "implement"
  ↓
/phased-dev:start-task 01
/phased-dev:start-task 02
...

(when done, switch back: /phased-dev:switch-scope project)
```

## Workflow — feature scope (design-heavy, opt-in)

For UX-led features. Same shape as standard feature but with a `ux` phase before engineering (`research` runs first, as everywhere):

```
/phased-dev:start-feature onboarding-rewrite "Redesign onboarding"
  → asks: standard or design-heavy? user picks design-heavy
  → creates docs/.phased-dev/scopes/feature/onboarding-rewrite.json
                                                 (type: feature-design-heavy)
  → also creates docs/features/onboarding-rewrite/research/ and .../ux/
  → dispatches researcher (the first phase is "research")
  ↓ (user reviews research at docs/features/onboarding-rewrite/research/)
/phased-dev:advance-phase          → currentPhase becomes "ux"
  ↓
/phased-dev:start-phase            → ux-designer drafts UX spec
  ↓ (user reviews UX spec at docs/features/onboarding-rewrite/ux/)
/phased-dev:advance-phase          → currentPhase becomes "engineering"
  ↓
/phased-dev:start-phase            → feature-architect produces engineering-focused
                                     engineering.md that references the UX spec
  ↓ (user reviews + advance)
/phased-dev:start-phase            → planner produces tasks; coverage walks the
                                     full UX spec (components, screens, microcopy, a11y)
  ↓ (user reviews + advance)
/phased-dev:start-task 01          → ... dev loop ...
```

The architect, feature-architect, and planner all check for `paths.uxDir` and adapt automatically — no command branches on pipeline type.

## What gets created in your project

**After `/phased-dev:init-project`:**

```
your-project/
├── CLAUDE.md                                 # Workflow + methodology pointer
└── docs/
    ├── .phased-dev/
    │   ├── state.json                        # Active scope pointer
    │   ├── scope-model.md                    # Reference for the scope abstraction
    │   └── scopes/
    │       └── project.json                  # Project scope config
    ├── STATUS.md                             # Human-readable mirror of project scope
    ├── decisions.md                          # Project-level decision log
    ├── known-issues.md                       # Deferred bugs / debt
    ├── research/
    │   └── YYYY-MM-DD-<slug>-research.md     # research phase output (evidence, prior art, feasibility)
    ├── brainstorm/
    │   └── YYYY-MM-DD-<slug>-design.md       # brainstorm phase output (product design spec)
    ├── engineering/
    │   └── YYYY-MM-DD-engineering-spec.md    # engineering phase output
    ├── methodology/
    │   ├── planning-methodology.md           # Rules for the plan phase
    │   └── execution-methodology.md          # Rules for the implement phase
    ├── templates/
    │   ├── plan-template.md
    │   ├── task-brief-template.md
    │   ├── log-template.md
    │   └── task-completion-template.md
    ├── plan/
    │   └── implementation-plan.md            # plan phase output
    └── tasks/
        └── task-NN-name/                     # One per task
            ├── brief.md                      # Immutable during execution
            ├── log.md                        # Created at task start
            └── completion.md                 # Written after task commit (marks task as done)
```

**If pipeline = design-heavy, additionally:**

```
your-project/
└── docs/
    └── ux/
        └── YYYY-MM-DD-ux-spec.md             # ux phase output — source of truth
```

The HTML preview (`preview/` directory) is generated separately via `/phased-dev:generate-ux-preview` — it is not part of the phase output. Downstream agents (architect, planner, reviewer) ignore it. The markdown spec is the only binding output of the `ux` phase.

**After `/phased-dev:start-feature my-feature`** (standard) — added alongside the project structure:

```
your-project/
└── docs/
    ├── .phased-dev/
    │   └── scopes/
    │       └── feature/
    │           └── my-feature.json           # Feature scope config
    └── features/
        └── my-feature/
            ├── STATUS.md                     # Human-readable mirror of this feature scope
            ├── engineering.md                # engineering phase output (combined product + engineering)
            ├── decisions.md                  # Feature-scoped decision log
            ├── plan.md                       # plan phase output
            └── tasks/
                └── task-NN-name/
                    ├── brief.md
                    ├── log.md
                    └── completion.md
```

**If feature pipeline = design-heavy, additionally:**

```
your-project/
└── docs/
    └── features/
        └── my-feature/
            └── ux/
                └── YYYY-MM-DD-ux-spec.md     # ux phase output (feature-scoped) — source of truth
```

Same rule as project scope: the HTML preview is generated separately via `/phased-dev:generate-ux-preview`; downstream agents ignore it; only the markdown spec binds.

The feature-scoped `decisions.md` is the write target for feature-local decisions. The project-level `docs/decisions.md` is reserved for cross-cutting decisions that span multiple features.

## Adding new scope types

The scope abstraction is data-driven: scope behavior is defined by its `pipeline` array, not by command code. The shipped `project-design-heavy` pipeline is a worked example — it adds a new phase (`ux`) and a new agent (`ux-designer`), but every other command works unchanged.

To add another scope type (e.g., `bugfix`, `refactor`, `migration`):

1. Add a new template at `templates/scopes/<type>.json` declaring the pipeline.
2. If the pipeline introduces a new phase that needs an agent, add `agents/<agent-name>.md`.
3. If downstream agents (e.g., the architect, planner) need to consume the new phase's output, teach them via a `paths.<dirname>` check — same pattern as the UX spec's `paths.uxDir`. Branch on the *presence* of the path, not the pipeline type.
4. Add a scaffolding command if the new scope type needs setup beyond the project scope (e.g., `/phased-dev:start-bugfix`). Standard project pipelines can be selected through `/phased-dev:init-project`'s pipeline question.
5. Existing commands (`start-phase`, `start-task`, `advance-phase`, etc.) work unchanged — they only read the active scope's pipeline.

This is the main architectural benefit of v0.2.

## Notes

- **The methodologies live in your project, not in this plugin.** `/phased-dev:init-project` copies them in. You can edit them per-project — the subagents read them from your project's `docs/methodology/`, not from the plugin.
- **Scope state is the source of truth.** `docs/.phased-dev/scopes/<id>.json` is authoritative. `docs/STATUS.md` (project) and `docs/features/<name>/STATUS.md` (features) are human-readable mirrors and may briefly lag. Agents read JSON; commands write JSON.
- **Completion markers gate phase advancement.** Each task writes a `completion.md` after its commit lands. The implement phase's `outputCheck` requires these files — `/phased-dev:advance-phase` cannot proceed until all tasks are done.
- **No hooks yet.** A future version may add pre-commit hooks for brief immutability and log presence.
- **v0.2 is a breaking change.** Old phased-dev v0.1 projects need to re-run `/phased-dev:init-project` (or scaffold the new files manually) to use v0.2. No automated migration.
