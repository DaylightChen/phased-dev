---
description: Walk the user through every open question, risk, known issue, and item needing confirmation in the active scope's docs — one at a time, with details and options — and record each resolution
argument-hint: "[optional: a phase name or doc path to limit the walkthrough to]"
---

# Resolve Open Questions — Interactive Walkthrough

User invoked with: $ARGUMENTS

Long specs accumulate open questions, risks, known issues, and assumptions that need the user's input. Reading a whole spec to find and answer them is tedious. Your job is to **collect every item that needs the user's decision and walk them through one at a time** — each with enough detail to decide and a set of concrete options — then **record each resolution** in the right place.

This command operates on the **active scope**. It is independent of phase gates: run it any time a phase has produced docs (typically while `phaseStatus` is `complete_awaiting_approval`, before `/phased-dev:advance-phase`), but it works whenever decision points exist.

## Pre-flight checks

1. **Confirm scope state exists.** Read `docs/.phased-dev/state.json`. If missing, tell the user to run `/phased-dev:init-project` first and stop.

2. **Read the active scope.** Get `activeScope` from `state.json`, then read `docs/.phased-dev/scopes/<activeScope>.json`. Pull `currentPhase`, `pipeline`, and `paths`.

## Step 1 — Collect the decision points

Gather candidate documents from the active scope's `paths`, plus project-level logs. Only include paths that are set on the scope and exist on disk:

- `paths.brainstormDir` (project scopes) — product design spec(s)
- `paths.uxDir` (design-heavy scopes) — the UX spec. Read only the dated markdown file(s); **ignore `paths.uxDir/preview/`** (HTML preview, not a spec).
- `paths.engineeringDir` (project-style) or `paths.engineering` (feature-style) — engineering spec(s)
- `paths.plan` — the implementation plan
- `paths.decisions` — the scope's decision log (look for entries flagged as tentative or needing confirmation)
- `docs/known-issues.md` — deferred bugs / debt
- For feature scopes, also the project's `docs/decisions.md` for cross-cutting items

**If `$ARGUMENTS` names a phase or a specific doc path, restrict the scan to that phase's output (or that file) only.** Otherwise scan all of the above.

From each document, extract items that need the user's input. Look for, at minimum:

- **Open questions** — sections titled "Open questions" (or similar) and any inline question the author left for the user.
- **Risks** — sections titled "Risks" and any flagged "what could go wrong" item.
- **Known issues** — entries in `docs/known-issues.md` and any "known issue" / deferred-bug note.
- **Anything else needing confirmation or clarity** — assumptions stated as tentative, `TODO`/`TBD`/`?` markers, "needs decision", "we should confirm", choices the author left open, and design tensions the author surfaced.

Build an ordered list of decision points. For each, capture: a short title, the source file and section, and the surrounding context. **De-duplicate** items that appear in more than one doc (e.g., a UX open question echoed in the engineering spec) — fold them into one decision point that cites both sources.

If you find **zero** decision points, tell the user there are no open questions, risks, or items needing confirmation in scope and stop.

Otherwise, tell the user how many decision points you found and that you'll walk them through one at a time, then begin.

## Step 2 — Walk through them one at a time

Present **exactly one** decision point per turn. Do not dump the whole list. For each one:

1. **Header** — `Question N of M — <short title>` and the source (`<file> › <section>`).
2. **Details** — restate the question/risk/issue in plain language with enough context that the user can decide without opening the file. Quote the relevant lines from the source. Explain *why it's open* and *what depends on it* (which later phase, task, or component is blocked or affected).
3. **Options** — enumerate the concrete ways to resolve it (`A`, `B`, `C`, …). For each option give the tradeoffs (what it costs, what it buys, what it rules out). When you have a reasoned default, mark it **(recommended)** and say why in one line. Always allow the user to answer in their own words instead of picking a listed option, and allow "skip" (leave unresolved) and "defer" (accept and log as a known issue).
4. **Ask** — ask the user to choose an option, write their own answer, skip, or defer. Then **wait** for their response before moving on.

After the user answers, briefly confirm your understanding of the decision in one sentence, then move to the next decision point. Keep a running tally (`Question N of M`).

Do not batch questions. Do not advance to the next item until the current one is answered or explicitly skipped/deferred.

## Step 3 — Record each resolution

As decisions are made (either immediately after each answer or batched at the end — your choice, but do not lose any), persist them so the resolution is durable and downstream agents see it:

- **Design/engineering/product/UX decisions** → append a dated entry to the appropriate decision log: `paths.decisions` for scope-local decisions; the project's `docs/decisions.md` only for genuinely cross-cutting ones (call this out when you do it). Record the question, the chosen option, and the rationale.
- **Accepted risks and deferrals** → append to `docs/known-issues.md` (or the appropriate known-issues log) with the risk, why it's accepted/deferred, and any trigger that should reopen it.
- **Update the source document** so the item no longer reads as open: edit the originating spec to replace the open question with its resolution (e.g., turn an "Open questions" bullet into a resolved note that points at the decision log entry). **Exception:** never edit a task `brief.md` if the scope is mid-`implement` — briefs are immutable during execution (per execution-methodology §5); record the resolution in the decision log and surface it to the user instead.
- **Skipped items** stay open — note them so the user knows what remains.

Make these edits as **content** changes only. Do **not** modify the scope JSON, `state.json`, or `docs/STATUS.md` — state transitions are owned by `/phased-dev:advance-phase` and the other state-writing commands.

## Step 4 — Summarize

When all decision points have been handled, give the user a compact summary:

- A table or list of every decision point → resolution (chosen option or "skipped"/"deferred").
- The files you wrote to (decision logs, known-issues, updated specs).
- Any items left **unresolved** (skipped) and what they block.
- The suggested next step: if the current phase is `complete_awaiting_approval` and nothing critical is unresolved, point them at `/phased-dev:advance-phase`. If critical questions were skipped, recommend resolving them (or rewinding) before advancing.

## Notes

- This command **reads** specs and **writes** content (decision logs, known-issues, resolved questions). It never writes scope state — that's the job of the gated commands.
- One question at a time is the whole point: it turns a long spec review into a guided conversation. Resist the urge to summarize everything up front.
- Always give the user an escape hatch (skip/defer/own-answer). Their judgment overrides any recommendation you make.
