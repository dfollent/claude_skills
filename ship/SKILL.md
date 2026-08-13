---
name: ship
description: Ship pipeline dispatcher. On-rails orchestrator that resolves the active per-ticket manifest and runs EXACTLY one pipeline step per invocation (never a full sequencer), applying per-stage continue rules. Convenience entry point over the independently-runnable ship- skills.
triggers:
  - /ship
model: opus
---

# Ship Dispatcher

Triggered by: `/ship [ticket-or-description]` or `/ship --status`

On-rails dispatcher for the Ship pipeline. Runs the NEXT pipeline step from the manifest, one
step per invocation. It is convenience, not a dependency: every `ship-` skill remains
independently runnable. NEVER auto-advance outside the continue rules below.

Folder + manifest conventions: `~/.claude/skills/_ship-shared/manifest-format.md`.

## Pipeline stages (in order)

| # | Stage | Skill |
|---|-------|-------|
| 1 | Questions | `ship-generate-questions` (repo scoping + per-repo questions) |
| 2 | Research | `ship-research-codebase` (ONE run covers all repos; per-repo runs parallel-safe) |
| 3 | Solution design | `ship-solution-design` |
| 4 | Work breakdown | `ship-work-breakdown` |
| 5 | Create plan | `ship-create-plan` |
| 6 | Implement plan | `ship-implement-plan` (terminal execution stage) |

The planning **trio** is stages 3-5 (`ship-solution-design`, `ship-work-breakdown`, `ship-create-plan`).
Stage 6 (`ship-implement-plan`) is the terminal execution stage, run after the planning trio.

## On every invocation

1. Resolve the ticket folder per `_ship-shared/manifest-format.md` (walk up from cwd for an
   existing `*-ship/`; confirm with the user; init folder + `manifest.md` if missing).
2. When several `*-ship/` folders exist, read `.ship-active` at the workspace root to pick the
   active one; if multiple entries are listed and cwd context does not make the target obvious,
   prompt the user to choose. Append the resolved folder name to `.ship-active` as a new line if
   it is not already present (do NOT overwrite existing entries).
3. Reconcile the `Research (repo: <name>)` rows from artifact presence (a `research-<repo>.md`
   present ⇒ `done`). Persist this reconcile to `manifest.md` ONLY when no concurrent research
   could be running (i.e. all expected `research-<repo>.md` are present). While any research row
   is still pending, derive the reconciled view IN-MEMORY and do NOT persist — concurrent
   terminals may be writing artifacts and you must not race them (single-writer rule). When you
   DO persist (all present), also rewrite the `## Status` block (`Current stage` + `Next`) to the
   newly-determined next stage, so a fresh thread never reads a stale `Next` (e.g. still pointing
   at `/ship-research-codebase`) that contradicts an artifact already on disk.
4. Print current position: `stage X/Y (<current stage>)`, what just completed, and what is
   upcoming.

## `--status` argument

If invoked as `/ship --status`: resolve the folder, reconcile in-memory, and print position only.
Run NO pipeline step. Fully read-only — skip BOTH the `.ship-active` write and the manifest
persist from "On every invocation" (steps 2 and 3); never write any file.

## One-step execution

Determine the next stage from the manifest `## Stages` table (first non-`done`, non-`n/a` row in
pipeline order) and run EXACTLY that one step by invoking the matching skill (stages 1-5 are the
`ship-` skills; stage 6 is `/ship-implement-plan`). If the manifest is fresh/empty, the
first stage is `ship-generate-questions`.

Keep an in-context counter `trio_run_this_thread`, incremented each time a trio stage
(`ship-solution-design`, `ship-work-breakdown`, `ship-create-plan`) is run in THIS thread. It
resets to 0 in every fresh thread (it is in-context only, never persisted).

## Continue rules (applied after a step completes)

- **questions** → offer to continue straight into research IN THIS THREAD by running
  `/ship-research-codebase` ONCE. If the user declines, stop. (That single run covers EVERY
  pending `Research (repo: <name>)` row, so continuing here completes the whole research stage.)

- **research** → researches ALL pending repos in ONE run: parallel research agents (one per repo
  by default) when more than one repo is pending, in-thread when a single pending repo also plans
  to a single agent. On the dispatching
  path the main thread (single-threaded) persists the `Research` reconcile and advances `Next` to
  `/ship-solution-design` once every `research-<repo>.md` has landed; then stop and hand off to
  design (fresh thread). Standalone fallback still works: the user may instead run
  `/ship-research-codebase` per repo in separate terminals (artifact-only, no manifest write); on
  the next `/ship`, reconcile from artifact presence — show position read-only while any row is
  pending (manifest NOT persisted, to avoid racing terminals), and once all artifacts exist
  persist the reconciled rows (single-threaded) and advance to design.

- **trio stage with `trio_run_this_thread < 2`** → offer to continue to the next stage IN THIS
  THREAD. Include the reminder: "if the statusline shows context past ~130k, stop and start a
  fresh thread instead."

- **trio stage with `trio_run_this_thread == 2`** → FORCE a stop: "two planning stages have run
  in this thread; start a FRESH thread and run /ship again." Do not offer to continue.

- **create-plan** → stop and hand off to stage 6, `/ship-implement-plan`. Do NOT
  auto-advance into implementation; the user runs it in a FRESH thread.

- **ship-implement-plan** → terminal stage. On completion it marks the `Implement plan` stage `done`,
  at which point the pipeline is complete.

Continuation within the trio ALWAYS requires explicit user consent. NEVER auto-advance outside
these rules.

## On every stop

1. Print the handoff block (`_ship-shared/manifest-format.md`) with the next command as `/ship`,
   and recommend starting a FRESH thread before the next stage.
2. Then call `AskUserQuestion` (header `Next step`) to ask what to do next. It COMPLEMENTS the
   handoff text — do not repeat it; the question is the actionable choice. Options and the
   recommended pick MUST mirror the `Next` line and the continue rule for the just-completed
   stage. Put the recommended option FIRST and suffix its label with `(Recommended)`. Honour the
   continue rules: NEVER offer in-thread continuation when the rule forces a stop.

Recommended option per stop (recommended listed first):
- **questions** → "Continue to research in this thread" / "Stop for now".
- **research** → "Start a FRESH thread for /ship-solution-design" / "Stop for now". If repos failed (or a standalone per-terminal run left some pending), instead: "Run the remaining repos" / "Stop for now".
- **trio stage, `trio_run_this_thread < 2`** → "Continue to <next stage> in this thread" / "Stop & start a fresh thread". If the statusline shows context past ~130k, flip the recommendation to the fresh-thread option.
- **trio stage, `trio_run_this_thread == 2`** (forced stop) → do NOT offer in-thread continuation: "Start a FRESH thread and run /ship" / "Stop for now".
- **create-plan** → "Start a FRESH thread for /ship-implement-plan" / "Stop for now".
- **ship-implement-plan** (terminal) → pipeline complete; skip the question (nothing to continue).
