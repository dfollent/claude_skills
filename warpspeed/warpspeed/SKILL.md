---
name: warpspeed
description: Warpspeed pipeline coordinator. Autonomous on-rails orchestrator that resolves the active per-ticket manifest and runs the NEXT pipeline step, deciding for itself whether to continue in-thread or hand off to a fresh thread. The ONLY human touchpoint in the whole pipeline is the grill-me decision walk inside the design stage. Convenience entry point over the independently-runnable pipeline skills.
triggers:
  - /warpspeed
model: opus
---

# Warpspeed Coordinator

Triggered by: `/warpspeed [ticket-or-description]` or `/warpspeed --status`

Autonomous on-rails coordinator for the Warpspeed pipeline. Runs the next pipeline step from the
manifest and decides for itself whether to continue or stop. It is convenience, not a dependency:
every pipeline skill remains independently runnable.

**Autonomy contract.** Warpspeed runs UNATTENDED. It NEVER asks the user "what next?", never asks
for approval, and never asks the user to choose a path. It decides continuation itself (see
Continue rules). The SINGLE human touchpoint in the entire pipeline is the interactive grill-me
decision walk inside the `design` stage — everything else (repo scoping, review depth, hand-offs,
continue-vs-stop) is decided autonomously and recorded in the manifest for transparency.

Folder + manifest conventions: `~/.claude/skills/_warpspeed-shared/manifest-format.md`.

## Pipeline stages (in order)

| # | Stage | Skill |
|---|-------|-------|
| 1 | Questions | `generate-questions` (repo scoping + per-repo questions) |
| 2 | Research | `research-codebase` (one run per repo, parallel-safe) |
| 3 | Solution design | `design` (grill-me walk + antagonistic review) |
| 4 | Work breakdown | `work-breakdown` |
| 5 | Create plan | `create-plan` |
| 6 | Implement plan | `implement-plan` (terminal execution stage — not warpspeed-aware) |

The planning **trio** is stages 3-5 (`design`, `work-breakdown`, `create-plan`). Stage 6
(`implement-plan`) is the terminal execution stage; warpspeed hands off to it and stops.

## On every invocation

1. Resolve the ticket folder per `_warpspeed-shared/manifest-format.md` (walk up from cwd for an
   existing `*-warpspeed/`; confirm with the user; init folder + `manifest.md` if missing).
2. When several `*-warpspeed/` folders exist, read `.warpspeed-active` at the workspace root to
   pick the active one; if cwd context does not make the target obvious, prompt the user to
   choose. Append the resolved folder name to `.warpspeed-active` as a new line if not already
   present (do NOT overwrite existing entries).
3. Reconcile the `Research (repo: <name>)` rows from artifact presence (a `research-<repo>.md`
   present ⇒ `done`). Persist this reconcile to `manifest.md` ONLY when no concurrent research
   could be running (i.e. all expected `research-<repo>.md` are present). While any research row
   is still pending, derive the reconciled view IN-MEMORY and do NOT persist (single-writer rule).
   When you DO persist, also rewrite the `## Status` block (`Current stage` + `Next`) to the
   newly-determined next stage.
4. Print current position: `stage X/Y (<current stage>)`, what just completed, and what is upcoming.

## `--status` argument

If invoked as `/warpspeed --status`: resolve the folder, reconcile in-memory, and print position
only. Run NO pipeline step. Fully read-only — skip BOTH the `.warpspeed-active` write and the
manifest persist from "On every invocation" (steps 2 and 3); never write any file.

## One-step execution

Determine the next stage from the manifest `## Stages` table (first non-`done`, non-`n/a` row in
pipeline order) and run EXACTLY that one step by invoking the matching skill. If the manifest is
fresh/empty, the first stage is `generate-questions`.

Keep an in-context counter `trio_run_this_thread`, incremented each time a trio stage (`design`,
`work-breakdown`, `create-plan`) is run in THIS thread. It resets to 0 in every fresh thread (it
is in-context only, never persisted).

## Continue rules (applied AUTONOMOUSLY after a step completes)

Warpspeed decides continue-in-thread vs stop-for-fresh-thread ITSELF — it does NOT ask the user.
The decision balances the user's "small context windows" goal against avoiding needless thread
churn. Default behaviour per stage:

- **questions** → auto-continue into research IN THIS THREAD. Research keeps the orchestrator lean
  (multi-repo fans out via Workflow; single-repo is one in-thread pass), so this is cheap.

- **research** → STOP. Design is heavy (interactive grill-me walk + up to 6 antagonistic agents)
  and deserves a fresh thread. Print the handoff with `Next: /warpspeed` and stop.

- **trio stage** → if `trio_run_this_thread < 2` AND the statusline context is under ~130k,
  auto-continue to the next stage IN THIS THREAD. Otherwise STOP and hand off to a fresh thread.
  This naturally groups `design` + `work-breakdown` into one thread, then forces a fresh thread
  for `create-plan`.

- **trio stage with `trio_run_this_thread == 2`** → FORCE a stop regardless of context. Two
  planning stages have run in this thread; print the handoff and stop.

- **create-plan** → STOP and hand off to stage 6, `/implement-plan`. Warpspeed does NOT drive
  implementation; the user runs `/implement-plan` in a fresh thread.

NEVER auto-advance outside these rules. The design stage's grill-me walk runs interactively
whenever stage 3 executes — that is the sole point where warpspeed waits on the user.

## On every stop

Print the handoff block (`_warpspeed-shared/manifest-format.md`) with the next command as
`/warpspeed` (or `/implement-plan` after create-plan), and recommend starting a FRESH thread
before the next stage. Do NOT ask the user a follow-up question — the handoff IS the instruction.
Once the pipeline reaches stage 6, warpspeed's job is complete.
