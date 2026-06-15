---
name: ship-solution-design
description: Ship pipeline stage 3. Produces a solution design that demonstrably covers the coverage map, driven by an interactive grill-me decision-tree question walk and hardened by tiered antagonistic review (full per-lens fan-out / combined single-pass / skip, chosen at a checkpoint). Reads all per-repo research + the ticket; writes design.md + design-questions.md into the manifest folder.
triggers:
  - /ship-solution-design
model: opus
---

# Ship Solution Design

Triggered by: `/ship-solution-design [ticket-or-description]`

Produce a design that demonstrably COVERS the coverage map. The METHOD is your own - do NOT
recreate a human design ritual to "hit" each box. The coverage map is an OUTPUT CONTRACT:
`~/.claude/skills/_ship-shared/coverage-map.md`.

Folder + manifest conventions: `~/.claude/skills/_ship-shared/manifest-format.md`.

## Inputs

1. **All per-repo research** - `research-<repo>.md` files in the ticket folder.
2. **Ticket/task description** - what we're building (fetch Jira with
   `mcp__claude_ai_Atlassian__getJiraIssue` if an ID is given).

## Process

### Step 1: Resolve folder + reconcile research + gather inputs

- Resolve the ticket folder per `_ship-shared/manifest-format.md`.
- RECONCILE the `Research (repo: <name>)` rows from artifact presence and PERSIST the manifest
  (single-threaded write is safe here - design runs once, after research): mark `done` each row
  whose `research-<repo>.md` exists.
- Read ALL `research-<repo>.md` files fully (no limit/offset) and the ticket.
- If research is missing for a seeded repo, tell the user and offer to wait / run it.

### Step 2: Identify patterns to follow

Using the research file references, read the specific code that represents patterns relevant to
this task. Extract 3-5 concrete snippets (with file:line) the implementation should follow.

### Step 3: Decision-tree question walk (grill-me style)

Replace a light "open questions" step with an INTERACTIVE walk. Draw on `grill-me`:

- Ask ONE question at a time. For each: FIRST write the question + recommended answer + branch
  to `design-questions.md` (leave chosen answer blank), THEN present it to the user. Writing
  before asking lets the team share the doc and collaborate on the answer.
- Walk down the decision tree, resolving dependencies between decisions one-by-one.
- If a question can be answered by exploring the codebase, explore instead of asking.
- **Span the coverage map**: the walk must touch every RELEVANT dimension. Start with
  product/requirements (technical decisions depend on it - dependency ordering, not ritual).
- **On reversal**: if the user reverses an earlier answer, INVALIDATE downstream answers on that
  branch and re-walk them.
- **Update** each question's entry in `design-questions.md` with the chosen answer once the
  user (or team) settles it. The question + recommended answer + branch were already written
  before asking (see above); the chosen answer is filled in here.
- **Pause/resume**: support async team fill-out. The user can leave and say "done" to resume.
- **Stop condition** (offer to stop ONLY when ALL hold): every relevant dimension touched AND
  >=8 questions asked AND 3 consecutive questions where the user took the recommendation. Then
  ask: "we seem aligned - stop or continue?".

### Step 4: Author the design

Write `design.md` to the ticket folder from the settled answers. Each coverage-map dimension
gets substantive coverage OR an explicit `N/A — <reason>`. The Open Questions section should be
near-empty (front-loaded by the walk).

```markdown
# Design: [Feature Name]

## Current State
[What exists today. file:line references.]

## Desired End State
[Observable behavior after implementation.]

## Patterns to Follow
### Pattern 1: [Name]
**File**: `path/to/file.ext:L42-L60`
**Why this pattern**: [reason]
```[language]
// snippet
```

## Coverage
For each dimension: substantive coverage OR `N/A — <reason>`.
### Product / requirements
### Architecture & codebase-pattern fit
### Security
### Performance & scalability
### Reliability & operations
### Data & integration

## Design Decisions
| Decision | Choice | Reasoning |
|---|---|---|

## Out of Scope
## Risks
## Open Questions
[Near-empty; only genuine residue from the walk.]
```

### Step 5: Antagonistic review (tiered)

**5a — Select lenses.** Pick the RELEVANT lenses from the coverage map (option b - your judgment).

**5b — Choose depth.** The per-lens fan-out is thorough but heavy (up to 6 Opus agents); not every
ticket needs it. Judge the ticket's size/risk, then offer the depth via `AskUserQuestion` (header
`Review depth`), recommended option FIRST with a `(Recommended)` suffix (mirror the `/ship`
stop-question convention):
- **Full** — one `antagonistic-reviewer` per relevant lens, in parallel. Deepest.
- **Combined** — one `antagonistic-reviewer` over all relevant lenses in a single pass. Lighter;
  a real depth tradeoff (focus diluted across lenses).
- **Skip** — no review. Trivial/mechanical change only.
Recommend **Full** when a risk lens (security / data / reliability) is relevant or scope is broad;
lean **Combined/Skip** for narrow, low-risk, mechanical work. The user always decides.

**5c — Dispatch.** Read the pertinent CLAUDE.md rules first (subagents do NOT auto-read CLAUDE.md).
Invoke `antagonistic-reviewer` (Task, `subagent_type: antagonistic-reviewer`), passing `design.md`,
the upstream research, the LENS, and the injected CLAUDE.md rules verbatim:
- **Full**: one invocation per lens, IN PARALLEL (LENS = a single lens each).
- **Combined**: ONE invocation, LENS = the full relevant-lens list.
- **Skip**: no invocation.

**5d — Incorporate.** Assess the findings; incorporate the valid ones into `design.md`.

**5e — Log to the manifest** under `### design.md` (formats: `_ship-shared/manifest-format.md`):
- Full: one `- Review [antagonistic:<lens>]: <outcome>` per lens.
- Combined: `- Review [antagonistic:combined(<lens,lens,...>)]: <outcome>`.
- Skip: `- Review [antagonistic:skipped]: <reason>`.
- EVERY tier: record any lens NOT reviewed (dropped by relevance or skipped) + its reason.
- EVERY tier: leave a `- Review [manual-cross-thread]: pending` line; update it if the user later
  pastes manual review feedback.

### Step 6: Manifest + handoff

- Mark the Design stage `done` with artifacts (`design.md`, `design-questions.md`).
- Set `Next` to `/ship-work-breakdown`.
- Print the handoff block (fresh thread).

## Rules

- **Output contract, not a process.** Cover the map; the method is yours.
- **Patterns must be real code** with file:line. Never invent examples.
- **No opinions in disguise.** A recommendation is a stated decision with reasoning.
- **Single-threaded stage** - may write `manifest.md`.
- **Stay within the instruction budget.**
