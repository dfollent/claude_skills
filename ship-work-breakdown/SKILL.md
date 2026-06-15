---
name: ship-work-breakdown
description: Ship pipeline stage 4. Takes design.md from the per-ticket manifest folder and produces a high-level work breakdown (phases, testing strategy, order of changes). The human review point before detailed planning. Writes work-breakdown.md into the manifest folder.
triggers:
  - /ship-work-breakdown
model: opus
---

# Ship Work Breakdown

Triggered by: `/ship-work-breakdown`

Produce a high-level work breakdown that shows how to get from current state to desired end
state. This is the last human review point before detailed planning. If the structure is wrong,
everything downstream is wasted work.

Analogy: if the plan is the C implementation, the work breakdown is the header file - just
signatures and types.

This skill adapts to the release model (Step 1): a single PR for one ticket, or independently
released vertical slices for a larger initiative. Phases are the agile cut for incremental delivery
and verification either way; the release model only changes which review gates apply.

Folder + manifest conventions: `~/.claude/skills/_ship-shared/manifest-format.md`.

## Inputs

- Resolve the ticket folder per `_ship-shared/manifest-format.md`.
- Read `design.md` (and the manifest) from the ticket folder fully before proceeding.

## Process

### Step 1: Determine the release model

Guess which model fits, from the design scope, the repo/subsystem breadth in the manifest, and any
phased-rollout / migration / feature-flag cues in `design.md`:

- **Single PR** - one ticket, all phases land in one branch and ship together. Phases are
  build-and-verify increments, not independent releases. Ordering is an implementation detail.
- **Independently released slices** - a larger initiative where phases (or groups of phases) are
  reviewed and released on their own. Ordering, dependencies, and stand-alone safety matter.

Default to Single PR unless the evidence points to an initiative. State your guess and the evidence
in one or two sentences, then ask the user to confirm or correct it. Record the chosen model in the
outline's Approach section. The model governs the review gates in Step 4.

### Step 2: Identify phases

Break the implementation into vertical slices - each phase a thin, working end-to-end increment,
never "Phase 1: all models, Phase 2: all controllers".

Size each phase to fit ONE `/ship-implement-plan` run in a fresh thread: reading the plan + design +
the files it touches, writing tests and code, and verifying, all comfortably under ~150k tokens of
context. If you cannot see a phase finishing in one such session, it is too big - split it.

When a vertical slice is bigger than one session:
1. First split it into THINNER vertical slices (a smaller end-to-end increment).
2. Only if no thinner vertical slice exists, fall back to sequential sub-phases (e.g. scaffold ->
   wire up -> integrate). These are verified by automated tests, not full end-to-end behaviour.

Ordering depends on the release model:
- **Single PR**: sequence phases so earlier ones do not force rework of later ones. Set this order
  sensibly yourself; it is NOT a question for the human.
- **Independently released slices**: ordering and inter-phase dependencies matter, and each
  released slice must stand alone (safe to ship without later phases). Note release boundaries.

### Step 3: Write the work breakdown

Write `work-breakdown.md` into the ticket folder.

```markdown
# Work Breakdown: [Feature Name]

> Based on design: design.md

## Approach
[2-3 sentences on the overall strategy]
[Release model: Single PR | Independently released slices - confirmed with user]

## Phases

### Phase 1: [Name]
**Goal**: [What this phase accomplishes - one sentence]
**Touches**: [Files/components affected - also the proxy for whether it fits one session]
**Verify**: [How to check this phase works - end-to-end where the slice allows, else automated tests]
**Depends on**: [Earlier phases - only when releasing slices independently]

### Phase 2: [Name]
**Goal**: ...
**Touches**: ...
**Verify**: ...

...

[Multi-release only: group phases under `## Release N` headings, or annotate which phases ship together.]

## Testing Strategy
- [How tests are structured across phases]
- [What gets unit tested vs integration tested vs manually verified]

## Risks
- [Risk and mitigation, only if not already covered in the design]

## Open Questions
- [Only questions that emerged from structuring, not rehashed from design]
```

### Step 4: Present for review

Show the outline and ask the user to confirm. Always:
1. Nothing is missing
2. Each phase is sized for a single implement session (none too big)
3. Testing strategy is adequate

Independently released slices additionally:
4. Release boundaries are right and each released slice stands alone
5. Ordering and inter-phase dependencies are correct

Do NOT proceed to `/ship-create-plan` until approved.

### Step 5: Manifest + handoff

- Mark the Work-breakdown stage `done` with its artifact path.
- Set `Next` to `/ship-create-plan`.
- Print the handoff block (fresh thread).

## Rules

- **Keep it under 200 lines.** This is a structure doc, not a plan. No code, no implementation details.
- **Match the release model.** Confirm it in Step 1; gate the review (Step 4) accordingly. Don't impose initiative-scale ceremony (ordering, release-independence) on a single-PR ticket, or skip it on an initiative.
- **Vertical slices, session-sized.** Each phase is a thin end-to-end increment that one `/ship-implement-plan` run can finish in a fresh ~150k-token thread. When splitting, prefer thinner vertical slices; fall back to sequential sub-phases only when no thinner slice exists.
- **No code snippets.** File paths yes, code no. That's the plan's job.
- **Don't rehash the design.** Reference it, don't repeat it.
- **Single-threaded stage** - may write `manifest.md`.
- **Stay under 30 instructions total.**
