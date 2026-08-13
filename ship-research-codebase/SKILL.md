---
name: ship-research-codebase
description: Ship pipeline stage 2. Research and document the codebase as-is, driven by per-repo research-questions files. Defaults to ONE research agent per repo (orientation dominates cost, so questions are batched), splitting only when a repo's questions span disjoint subsystems. Writes research-<repo>.md per repo into the per-ticket manifest folder. Pure codebase archaeology - no opinions, no implementation suggestions. Never sees the ticket.
triggers:
  - /ship-research-codebase
model: opus
---

# Ship Research Codebase

Triggered by: `/ship-research-codebase`

Conduct objective codebase research driven by a repo's questions file. Produce a self-contained
research document usable in a fresh context window.

**This skill must NOT see the ticket or task description.** It receives only research questions.
This prevents implementation bias from coloring findings. Repo selection is already done upstream
by `ship-generate-questions`; this skill never sees the ticket.

This stage researches EVERY pending repo in the confirmed scope, then writes `research-<repo>.md`
per repo.

Standalone fallback (still supported): you may instead run this once per repo in separate
terminals (one repo each). Those runs stay PURE ARTIFACT PRODUCERS — they write only
`research-<repo>.md` and never `manifest.md`, which keeps them parallel-safe.

Folder + manifest conventions: `~/.claude/skills/_ship-shared/manifest-format.md`.

## Cost model (read before deciding how many agents to spawn)

An agent's cost is dominated by ORIENTATION — reading `CLAUDE.md`, mapping the repo, locating the
subsystems — not by how many questions it then answers. Once oriented, each extra question is
cheap. A second agent pointed at the same subsystem pays orientation twice and buys nothing.

Therefore: **batch questions into as few agents as possible.** Splitting is justified ONLY when
the slices need genuinely different orientation. Theme count is not a reason to split.

## Process

### Step 0: Resolve folder + pending repos

- Resolve the ticket folder per `_ship-shared/manifest-format.md` (walk up + confirm; init if
  missing).
- Read the manifest `## Repo Scope` (workspace root + selected repos) and the
  `Research (repo: <name>)` rows.
- **Pending set** = selected repos whose `research-<repo>.md` is ABSENT from the folder
  (idempotency guard: repos whose artifact already exists are skipped — they reconcile to `done`).
  Regenerate an existing artifact ONLY if the user explicitly asks.
- If the pending set is EMPTY → report all research present, set `Next` to `/ship-solution-design`,
  stop.
- If the user tries to provide a ticket/task description, redirect them to
  `/ship-generate-questions` first.

### Step 1: Build the research plan (cheap reads in the main thread)

For each PENDING repo, read `research-questions-<repo>.md` (small file — safe in the main thread)
and record: the repo's absolute path under the workspace root, its root `CLAUDE.md` path, and the
themes (`##` headings) with their questions.

Then assign agents per repo using the **split rule**:

- **Default: ONE agent for the whole repo**, carrying every theme.
- **Split only if BOTH hold**: the repo has more than 12 questions, AND its themes partition into
  subsystems that do not share files (different top-level modules/directories).
- Group WHOLE themes into slices — never split a theme across agents.
- Themes touching the same subsystem MUST land in the same slice, however many questions they add.
- **Cap: 3 agents per repo.** If a partition needs more, merge the smallest slices.
- Name each slice after the subsystem it owns, not after a theme number.

State the plan to the user before dispatching: one line per repo — repo, agent count, and, when
split, the subsystem boundary that justified it.

### Step 2: Dispatch

**2a. In-thread** — take this path when exactly one repo is pending AND its plan is a single
agent. Research it yourself in the main thread and write `research-<repo>.md` per Step 3. This
also covers standalone per-terminal fallback runs.

**2b. Parallel agents** — otherwise, dispatch every slice across every pending repo as `Agent`
calls in a SINGLE message so they run concurrently. The main thread researches NOTHING itself;
this is what keeps its context clean.

- Unsplit repo → its agent writes `research-<repo>.md` directly (Step 3 template).
- Split repo → each agent writes its fragment to `<folder>/.research-parts/<repo>-<slice>.md`.
- Every agent returns ONLY a one-line summary (finding count, open-question count, path written).
  **Findings must never flow back through an agent return into the main thread.**

**2c. Synthesis (split repos only)** — once a split repo's slice agents are done, dispatch one
synthesis agent for it. Give it the fragment PATHS, not the fragment text; it reads them from
disk, writes `research-<repo>.md` per the Step 3 template, connects findings across slices into
end-to-end flows in `## Data Flow`, then deletes that repo's files under `.research-parts/`.
It returns only the one-line summary and must not write `manifest.md`.

### Research agent contract

Every research agent prompt (2a's own behaviour included) must:

- Name the repo's `CLAUDE.md` file(s) — root plus any nested `CLAUDE.md` in its target area — and
  instruct the agent to read them FIRST and let them guide where it looks and the terminology it
  uses. In a multi-repo workspace the cwd is the workspace root, so a repo's own `CLAUDE.md` is
  NOT auto-loaded.
- Carry only that agent's questions, verbatim, grouped under their theme headings.
- State: "Document what exists. No opinions, no suggestions, no improvements."
- Require a `file:line` reference for EVERY finding.
- Include a code snippet ONLY where the reference alone cannot convey the pattern, capped at ~15
  lines. Never paste a whole file, and never invent an example.
- Trace flows end-to-end where a question touches one, even across into other areas.

### Step 3: Output

Write `research-<repo>.md` INTO the ticket folder (not the loose working dir).

```markdown
# Research: [Topic Area]

> Self-contained research document. Usable without prior conversation context.

## Questions Investigated
[List of questions this research addressed]

## Summary
[3-5 bullet points answering the core questions]

## Detailed Findings

### [Finding Area 1]
[What exists, how it works, where it lives]
- `path/to/file.ext:L42` - [what this code does]

### [Finding Area 2]
...

## Patterns Found

Reusable patterns discovered in the codebase. Reference by file:line; add a short snippet only
where the reference alone is not enough:

### [Pattern Name]
**Where**: `path/to/file.ext:L42-L60`
**Used by**: [list of consumers]
```[language]
// short real snippet, only if needed
```

## Data Flow
[How data moves through the relevant systems, traced end-to-end]

## Open Questions
[Things that could not be determined from code alone]
```

### Step 4: Reconcile + handoff

**In-thread / standalone runs (2a)**: write ONLY `research-<repo>.md`. Do NOT edit `manifest.md` —
research is parallel-safe (it may run in several terminals at once, one repo each) and concurrent
manifest writes would race. Print the handoff block from in-memory observation: if other seeded
research rows still lack their artifact, `Next` says "run the remaining repo(s) - parallel
terminals OK"; when this run observes ALL artifacts present, `Next` is `/ship-solution-design`.

**Dispatching runs (2b/2c)**: you are single-threaded here, so reconciling the manifest is safe.

- Reconcile from ARTIFACT PRESENCE, not from what the agents returned: a repo with
  `research-<repo>.md` present ⇒ `done`.
- Any repo still MISSING its artifact = failed; report it and leave its row pending. The user can
  re-run `/ship-research-codebase` to retry ONLY the missing repos (Step 1 re-includes only pending).
- When ALL pending artifacts are present, mark the `Research (repo: <name>)` rows `done` and
  rewrite `## Status` with `Next: /ship-solution-design (run in a FRESH thread)`.
- Print the handoff block plus a per-repo summary table (repo, findings, open questions, artifact
  path).

## Rules

- **Batch, don't fan out.** One agent per repo is the default; the split rule is the exception and
  must be justified by a subsystem boundary, never by question or theme count.
- **CLAUDE.md is mandatory orientation.** The main thread AND every sub-agent must read the repo's `CLAUDE.md` (root + relevant nested) before researching; it is the highest-signal codebase context and is not auto-loaded from a nested repo.
- **Document what IS, not what SHOULD BE.** You are a cartographer, not an architect.
- **No implementation opinions.** If you catch yourself writing "should", "could be improved", or "consider", delete it.
- **Patterns must be real.** Reference every pattern by file:line; snippets are optional, short, and never invented.
- **Self-contained output.** The research doc must be usable in a fresh context window with zero prior conversation.
- **Vertical over horizontal.** Trace flows end-to-end.
- **Findings live on disk, not in returns.** Agents write files and return one-line summaries;
  synthesis reads fragment paths, never pasted fragment text.
- **Research agents never write `manifest.md`.** All research and synthesis agents, and the
  in-thread/standalone paths, are pure artifact producers — they write only `research-<repo>.md`
  (or their `.research-parts/` fragment). ONLY a dispatching main thread persists the manifest
  reconcile, and ONLY after every artifact has landed.
- **Stay under 55 instructions total.**
