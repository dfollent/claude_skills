---
name: ship-research-codebase
description: Ship pipeline stage 2. Research and document the codebase as-is, driven by per-repo research-questions files. Researches one repo in-thread, or all pending repos in parallel via an orchestrated Workflow fan-out (one agent per question theme). Writes research-<repo>.md per repo into the per-ticket manifest folder. Pure codebase archaeology - no opinions, no implementation suggestions. Never sees the ticket.
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

This stage researches EVERY pending repo in the confirmed scope. With MORE THAN ONE pending repo
it orchestrates an in-thread Workflow fan-out: the main thread stays a lean orchestrator while the
heavy findings live in subagent contexts and on disk, never in the orchestrator. With EXACTLY ONE
pending repo it researches in-thread directly. Either way it writes `research-<repo>.md` per repo,
and the Workflow research/synthesis agents NEVER write `manifest.md`.

Standalone fallback (still supported): you may instead run this once per repo in separate
terminals (one repo each). Those runs stay PURE ARTIFACT PRODUCERS — they write only
`research-<repo>.md` and never `manifest.md`, which keeps them parallel-safe.

Folder + manifest conventions: `~/.claude/skills/_ship-shared/manifest-format.md`.

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
- If EXACTLY ONE repo is pending → take the **Single-repo path** (in-thread, Steps 1-5).
- If MORE THAN ONE repo is pending → take the **Multi-repo path** (orchestrated Workflow, M1-M3).
- If the user tries to provide a ticket/task description, redirect them to
  `/ship-generate-questions` first.

## Single-repo path (in-thread)

Used when exactly one repo is pending (and by standalone per-terminal fallback runs). `<repo>` is
that pending repo. This path is a PURE ARTIFACT PRODUCER: it writes only `research-<repo>.md` and
never `manifest.md`, so it stays parallel-safe across terminals.

### Step 1: Read questions, CLAUDE.md, and any referenced files

Read the questions file fully. Then, BEFORE decomposing, locate and read this repo's `CLAUDE.md`
file(s) in the main context: the repo-root `CLAUDE.md` plus any nested module-level `CLAUDE.md`
covering the areas the questions touch. In a multi-repo workspace the cwd is the workspace root,
so a repo's own `CLAUDE.md` is NOT auto-loaded — you MUST read it explicitly. Treat these as the
highest-signal orientation for the whole run (architecture, conventions, terminology, gotchas).
If the questions mention specific files, read them in the main context too.

### Step 2: Decompose into vertical research tasks

Break questions into parallel research tasks. Bias toward vertical slices - trace flows
end-to-end rather than cataloging horizontal layers.

Spawn parallel sub-agents:
- **Explorer agents** to find where components live
- **Tracer agents** to follow a flow from entry point to data store
- **Pattern agents** to find how the codebase handles similar concerns elsewhere

Each sub-agent prompt must:
- Include only its specific questions
- Name the repo's `CLAUDE.md` file(s) (root + any nested `CLAUDE.md` in its target area) and
  instruct the agent to read them FIRST and let them guide where it looks and the terminology it uses
- State: "Document what exists. No opinions, no suggestions, no improvements."
- Request file:line references for all findings
- Request actual code snippets for patterns found

### Step 3: Synthesize findings

Wait for ALL sub-agents to complete. Then compile results into a self-contained research document.

### Step 4: Write output

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

Reusable patterns discovered in the codebase, with code snippets:

### [Pattern Name]
**Where**: `path/to/file.ext:L42-L60`
**Used by**: [list of consumers]
```[language]
// actual code snippet
```

## Data Flow
[How data moves through the relevant systems, traced end-to-end]

## Open Questions
[Things that could not be determined from code alone]
```

### Step 5: Handoff (display only - NOTHING persisted)

Write ONLY `research-<repo>.md`. Do NOT edit `manifest.md` - research is parallel-safe (it may
run in several terminals at once, one repo each) and concurrent manifest writes would race. The
`Research (repo: <name>)` row is a derived view, reconciled downstream from artifact presence.

Print the handoff block from in-memory observation:
- If other seeded research rows still lack their `research-<repo>.md`, `Next` says
  "run the remaining repo(s) - parallel terminals OK".
- When this run observes ALL `research-<repo>.md` present, `Next` is `/ship-solution-design`.

## Multi-repo path (orchestrated Workflow)

Used when more than one repo is pending. The main thread is the ORCHESTRATOR: build the work-list,
run ONE Workflow, post-process. It must NOT research anything itself — this is what keeps its
context clean. (A skill instructing this Workflow call is a legitimate opt-in.)

### M1: Build the work-list (cheap reads in the main thread)

For each PENDING repo: read `research-questions-<repo>.md`, parse it into themes (one entry per
`##` heading) with that theme's questions, and record the repo's absolute path (under the
workspace root) and its root `CLAUDE.md` path. Assemble `args`:

```json
{ "folder": "<abs ticket folder>",
  "repos": [ { "repo": "api", "repoPath": "<abs path>",
               "claudeMd": ["<repoPath>/CLAUDE.md"],
               "themes": [ { "name": "Data models", "questions": ["..."] } ] } ] }
```

### M2: Run the Workflow (one call)

Invoke the `Workflow` tool with the script below and the `args` from M1. Per repo it pipelines:
research every theme in parallel (one agent per theme), then synthesize + write
`research-<repo>.md`. Findings stay in agent/script memory and on disk — only one-line summaries
return to the main thread.

```js
export const meta = {
  name: 'ship-research-multi-repo',
  description: 'Research pending repos in parallel: per repo, one agent per question theme, then synthesize research-<repo>.md',
  phases: [
    { title: 'Research', detail: 'one agent per (repo, theme)' },
    { title: 'Synthesize', detail: 'one agent per repo writes research-<repo>.md' },
  ],
}
// args: { folder, repos: [{ repo, repoPath, claudeMd:[paths], themes:[{name, questions:[str]}] }] }
const SUMMARY = {
  type: 'object',
  properties: {
    repo: { type: 'string' },
    findingCount: { type: 'integer' },
    openQuestionCount: { type: 'integer' },
  },
  required: ['repo', 'findingCount', 'openQuestionCount'],
  additionalProperties: false,
}
// args may arrive parsed OR as a JSON string depending on the Workflow runtime; normalise so
// the script works either way and never trips pipeline()'s array check on a raw string.
const cfg = typeof args === 'string' ? JSON.parse(args) : args
const summaries = await pipeline(
  cfg.repos,
  // Stage 1: research every theme of this repo in parallel
  (repo) => parallel(repo.themes.map(theme => () =>
    agent(
      `Research the repo at ${repo.repoPath}. FIRST read its CLAUDE.md (${repo.claudeMd.join(', ')}) ` +
      `and any nested CLAUDE.md in the area you investigate; let it guide where you look and the terms you use.\n\n` +
      `Investigate ONLY these questions (theme "${theme.name}"):\n` +
      theme.questions.map((q, i) => `${i + 1}. ${q}`).join('\n') +
      `\n\nDocument what EXISTS. No opinions, no suggestions, no improvements. Give file:line refs for every ` +
      `finding and real code snippets for every pattern. Where a question touches a flow, trace it end-to-end ` +
      `even across into other areas. Return your findings as a markdown fragment.`,
      { label: `research:${repo.repo}:${theme.name}`, phase: 'Research' }
    ).then(text => ({ theme: theme.name, text }))
  )),
  // Stage 2: synthesize this repo's findings into research-<repo>.md
  (themeFindings, repo) =>
    agent(
      `Write a self-contained research document for the repo at ${repo.repoPath} to ` +
      `${cfg.folder}/research-${repo.repo}.md (overwrite if present). Read its CLAUDE.md ` +
      `(${repo.claudeMd.join(', ')}) for framing and terminology. Use EXACTLY these sections: ` +
      `Questions Investigated; Summary (3-5 bullets); Detailed Findings (file:line refs); ` +
      `Patterns Found (file:line + real snippets); Data Flow (connect findings across themes into ` +
      `end-to-end flows); Open Questions. Document what IS, not what SHOULD BE — no opinions. ` +
      `Do NOT write manifest.md. Compile these per-theme findings:\n\n` +
      themeFindings.filter(Boolean).map(f => `### Theme: ${f.theme}\n${f.text}`).join('\n\n') +
      `\n\nAfter writing the file, return the summary counts for this repo.`,
      { label: `synth:${repo.repo}`, phase: 'Synthesize', agentType: 'general-purpose', schema: SUMMARY }
    )
)
return summaries
```

### M3: Post-process (main thread, after the Workflow returns)

- Reconcile from artifact presence (NOT from the returned `summaries`): a repo with
  `research-<repo>.md` present ⇒ `done`. A synth agent that threw is `null` in `summaries`, so
  `.filter(Boolean)` it before building the summary table.
- Any repo still MISSING its artifact = failed; report it and leave its row pending. The user can
  re-run `/ship-research-codebase` to retry ONLY the missing repos (M1 re-includes only pending).
- When ALL pending artifacts are present, persist the manifest reconcile (you are single-threaded
  here, so this is safe): mark the `Research (repo: <name>)` rows `done` and rewrite `## Status`
  with `Next: /ship-solution-design (run in a FRESH thread)`.
- Print the handoff block (`Next` = `/ship-solution-design`) plus a per-repo summary table (repo,
  findings, open questions, artifact path).

## Rules

- **CLAUDE.md is mandatory orientation.** The main thread AND every sub-agent must read the repo's `CLAUDE.md` (root + relevant nested) before researching; it is the highest-signal codebase context and is not auto-loaded from a nested repo.
- **Document what IS, not what SHOULD BE.** You are a cartographer, not an architect.
- **No implementation opinions.** If you catch yourself writing "should", "could be improved", or "consider", delete it.
- **Patterns must include real code.** Every pattern needs a snippet with file:line. Never invent examples.
- **Self-contained output.** The research doc must be usable in a fresh context window with zero prior conversation.
- **Vertical over horizontal.** Trace flows end-to-end.
- **Research agents never write `manifest.md`.** All Workflow research/synthesis agents and the
  single-repo/standalone paths are pure artifact producers — they write only `research-<repo>.md`.
  ONLY the multi-repo orchestrator (single-threaded main thread) persists the manifest reconcile,
  and ONLY after every artifact has landed.
- **Orchestrator stays lean.** On the multi-repo path the synthesis agents WRITE the artifact and
  return only one-line summaries; full findings must NEVER flow back through the Workflow return
  into the main thread.
- **Stay under 55 instructions total.**
