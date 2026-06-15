---
name: ship-implement-plan
description: Execute an implementation plan phase by phase, with verification between phases and pauses for manual testing. Consumes the plan from /ship-create-plan, or runs as Ship pipeline stage 6 (the plan from /ship-create-plan).
triggers:
  - /ship-implement-plan
model: opus
---

# Implement Plan

Triggered by: `/ship-implement-plan [plan-file]`
Example: `/ship-implement-plan 2026-04-14-SP-42-plan.md`

Execute an approved implementation plan phase by phase. The plan was produced by `/ship-create-plan` and already incorporates decisions from the design discussion and work breakdown. Your job is to follow the plan's intent, verify as you go, and pause for human checks between phases.

## Inputs

A plan file (output from `/ship-create-plan`). Read it fully. Also read the design discussion referenced in the plan header - it contains the patterns to follow.

If no plan file is provided, ask for one.

## Process

### Step 1: Orient

- Read the plan fully
- Read the design discussion referenced in the plan (for patterns and decisions)
- Check for existing checkmarks (`- [x]`) - if resuming, pick up from the first unchecked item
- Read all files mentioned in the first phase before writing any code

### Step 2: Implement phase by phase

For each phase:

1. Read all files involved in the phase before making changes
2. Write failing tests first (TDD), then implement to make them pass
3. Follow the patterns from the design discussion - don't invent new patterns
4. Update checkboxes in the plan file as you complete items

### Step 3: Verify each phase

After completing a phase, run the automated verification listed in the plan (tests, types, lint). Fix issues before proceeding.

Then git commit all changes from the phase (excluding the plan file itself). Use a concise commit message describing the phase intent.

Then pause using the `AskUserQuestion` tool with:
- A summary of what was completed and what automated checks passed
- The manual verification steps from the plan listed as the question
- Options: "Ready - proceed to Phase [N+1]" and "Issues found - hold"

Do not proceed until the user confirms via the tool response. Do not check off manual verification items yourself.

If instructed to run multiple phases consecutively, skip the pause and commit until the last phase.

### Step 4: Handle mismatches

If the code doesn't match what the plan expects, stop:

```
Mismatch in Phase [N]:
Expected: [what the plan says]
Found: [actual situation]
Why this matters: [impact on the plan]

Options:
1. [Adapt the current phase to work with what exists]
2. [Flag that the upstream design/structure may need revisiting]
```

Do not silently work around mismatches. The plan was built on specific assumptions from the design discussion - if those assumptions are wrong, the human needs to know.

### Step 5: Close out the Ship stage (only if applicable)

This skill is Ship pipeline stage 6. If the plan file lives inside a `*-ship/` folder that contains a `manifest.md`, then after ALL phases are complete and verified, mark the `Implement plan` stage `done` in that `manifest.md` (see `_ship-shared/manifest-format.md`). This is the ONLY manifest write this skill makes, and it is single-threaded (implementation runs in one thread).

For standalone runs (no `*-ship/` manifest), skip this step entirely - there is nothing to update.

## Rules

- **TDD: write failing tests first.** Then implement to make them pass.
- **Follow the plan's intent, not just its letter.** If the codebase has shifted since the plan was written, adapt intelligently but communicate what changed.
- **No research or exploration.** That work is done. If you're missing information, the upstream artifacts are incomplete - flag it.
- **No new patterns.** Use the patterns from the design discussion. If they don't fit, that's a mismatch to surface, not a judgment call to make alone.
- **One phase at a time.** Complete and verify before moving on.
- **Stay under 40 instructions total.**
- **Do not git commit the plan file itself.**
