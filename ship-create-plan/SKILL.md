---
name: ship-create-plan
description: Ship pipeline stage 5. Takes work-breakdown.md + design.md + all research from the per-ticket manifest folder and produces a detailed tactical implementation plan, hardened by an antagonistic-reviewer design-lens pass. Writes plan.md into the manifest folder and hands off to /ship-implement-plan.
triggers:
  - /ship-create-plan
model: opus
---

# Ship Create Plan

Triggered by: `/ship-create-plan`

Produce a detailed, actionable implementation plan that an agent can execute phase by phase. The
human already reviewed the design and work breakdown. This plan translates those into tactical
instructions.

Folder + manifest conventions: `~/.claude/skills/_ship-shared/manifest-format.md`.

## Inputs

Resolve the ticket folder per `_ship-shared/manifest-format.md`, then read fully:

1. **work-breakdown.md** - required
2. **design.md** - the settled design
3. **All `research-<repo>.md`** files - the codebase ground truth

If any are missing, ask the user (the upstream stage probably did not run).

## Process

### Step 1: Gather implementation details

For each phase in the work breakdown, spawn parallel sub-agents to find:
- Exact files that need changes (with line numbers)
- Code patterns to follow (from the design's "Patterns to Follow")
- Existing test patterns in the relevant areas
- Import conventions, naming conventions, config patterns

### Step 2: Write the plan

Write `plan.md` into the ticket folder.

```markdown
# Implementation Plan: [Feature Name]

> Work breakdown: work-breakdown.md
> Design: design.md

## Phase 1: [Name from work breakdown]

### 1.1 [Change group]
**File**: `path/to/file.ext`
**Action**: [Create | Modify | Delete]
**Details**:
- [Specific change with enough detail to execute]
- [Follow pattern from `path/to/example.ext:L42-L60`]

### Phase 1 Verification
#### Automated:
- [ ] `[test command]` passes
- [ ] `[typecheck command]` passes

#### Manual:
- [ ] [Specific thing to verify]

**Pause for manual verification before proceeding.**

---

## Phase 2: [Name]
...
```

### Step 3: Solution-design-lens antagonistic review

The reviewer is primed with the design + research + structure (the solution-design lens), NOT the
plan's own authoring rationalisation - so it checks the plan against intent, not against itself.

- Read the pertinent CLAUDE.md rules (subagents do not auto-read CLAUDE.md).
- Invoke the `antagonistic-reviewer` subagent (Task, `subagent_type: antagonistic-reviewer`) with
  LENS = design, passing `plan.md`, the design, the research, the work breakdown, and the
  injected CLAUDE.md rules verbatim.
- Assess findings; incorporate the valid ones into `plan.md`.
- Log to the manifest under `### plan.md`: `- Review [antagonistic:design-lens]: <outcome>`, plus a
  `- Review [manual-cross-thread]: pending` line for any later manual feedback.

### Step 4: Manifest + handoff

- Mark the Create plan stage `done` with the artifact path.
- Set `Next` to `/ship-implement-plan` (stage 6 - terminal execution stage).
- Present the plan file location; remind the user the real review happened at design + structure
  level. Print the handoff block.

## Rules

- **This plan is for agent execution.** Write it as instructions, not as a proposal.
- **No open questions.** Every decision was made upstream. If you find an unresolved question, stop and ask before continuing.
- **Reference patterns, don't reinvent.** Point to the design's code snippets. Don't discover new patterns here.
- **Pause between phases.** Every phase ends with verification and an explicit pause for manual checks.
- **No research.** The research is done. If information is missing, the upstream artifacts are incomplete - flag it rather than spawning explorers.
- **Single-threaded stage** - may write `manifest.md`.
- **Stay under 40 instructions total.**
