---
name: ship-generate-questions
description: Ship pipeline stage 1. Scopes which repos in a multi-repo workspace the ticket touches (it legitimately sees the ticket), then writes one intent-neutral research-questions file per confirmed repo plus the per-ticket manifest. First half of the untainted research pattern - this stage sees the intent, ship-research-codebase does not.
triggers:
  - /ship-generate-questions
model: opus
---

# Ship Generate Research Questions (repo scoping + per-repo questions)

Triggered by: `/ship-generate-questions [ticket-or-description]`
Example: `/ship-generate-questions SP-42`

Generate research questions that drive objective codebase research. This stage sees the
ticket. The subsequent `/ship-research-codebase` stage only sees the per-repo questions files -
never the ticket. This separation prevents implementation opinions from tainting research.

This is the ONLY pipeline stage that legitimately sees the ticket, so repo scoping (routing
repos against the ticket) lives HERE. The per-repo question files it emits stay intent-neutral,
so downstream research never sees the ticket.

Folder + manifest conventions: `~/.claude/skills/_ship-shared/manifest-format.md`.

## Inputs

A ticket ID or task description. If a Jira ticket ID is provided (e.g. SP-42), fetch it with
`mcp__claude_ai_Atlassian__getJiraIssue`.

## Process

### Step 1: Understand the task

Read the ticket/description. Identify:
- What needs to change
- Which systems are likely involved
- What you don't know about the codebase yet

### Step 2: Scope the repos (workspace detect + CLAUDE.md-driven suggestion + confirm)

1. **Resolve the workspace root.** Treat cwd (or its parent) as the workspace root when it
   contains multiple git subdirectories. If no sibling repos exist (cwd is itself the only
   repo), SINGLE-REPO DEGRADE: skip the shortlist UI and proceed with the current repo only
   (one questions file), behaving like the original generate-questions.
2. **Discover candidate repos.** Every git directory directly under the workspace root is a
   candidate.
3. **Read each candidate's `CLAUDE.md` if present** (cheap signal: purpose, stack, domain).
   Repos WITHOUT a `CLAUDE.md` are still candidates, described by name/path only. Nothing is
   hidden.
4. **Rank candidates against the TICKET** (this stage legitimately sees it) and produce a
   shortlist, each with a one-line relevance reason.
5. **Present the FULL candidate list** to the user with the shortlist pre-selected; ask them to
   confirm or redirect (multi-select). The user has final say over which repos are in scope.
   Also present the detected workspace root for confirm/redirect.
6. **Resolve/init the ticket folder** per `_ship-shared/manifest-format.md` (walk up + confirm;
   init if missing). Write the `## Repo Scope` section: the confirmed workspace root and the
   `| Repo | Selected | Reason |` table listing EVERY candidate (selected and dropped, with
   reasons).

### Step 3: Generate per-repo questions

For each confirmed repo, FIRST re-read that repo's `CLAUDE.md` (the root one read during scoping
plus any nested module-level `CLAUDE.md` covering the themes in play) and let it guide the
questions: anchor them to the subsystems, conventions, terminology, and gotchas it names, so the
questions point research at the right places using the repo's own vocabulary. This sharpens
specificity; it does NOT relax the intent-neutral firewall (`CLAUDE.md` is codebase context, not
ticket intent).

For each confirmed repo, produce 5-15 research questions organized by theme. Each question:
- **Answerable by reading code** - not product/business questions
- **Intent-neutral** - someone reading the questions alone should not be able to infer what
  feature is being built. (The questions MUST stay intent-neutral even though scoping used the
  ticket - this is what keeps downstream research untainted.)
- **Specific** - "How does the auth middleware chain work?" not "How does auth work?"

Categories to consider: existing patterns/conventions; data models and schemas; integration
points and interfaces; test patterns; error handling and edge cases; configuration/environment.

### Step 4: Output + manifest seeding

- Write one file per confirmed repo: `research-questions-<repo>.md` into the ticket folder.

```markdown
# Research Questions: [Topic Area]

## [Theme 1]
1. [Question]
2. [Question]

## [Theme 2]
3. [Question]
...
```

- Seed the manifest `## Stages` table:
  - one `Questions (repo: <name>)` row per confirmed repo marked `done` with its artifact, and
  - one `Research (repo: <name>)` row per confirmed repo marked `pending` (so research knows the
    expected set). Repos NOT selected get `n/a` rows.
- Set `Next` to `/ship-research-codebase` with the fresh-thread instruction.
- Present the questions to the user for review (they may add/remove/refine before research).
- Print the handoff block.

## Rules

- **No implementation opinions.** Questions only, no suggestions about how to build anything.
- **Intent-neutral framing.** Strip the "why". Ask "how does X work?" not "how should we modify
  X to do Y?" This firewall holds even though scoping read the ticket.
- **Bias toward vertical slices.** Trace end-to-end flows, not horizontal layers.
- **Repo scoping lives here, not in research** - it is the only stage that sees the ticket.
- **Single-writer**: this is a single-threaded stage, so it MAY write `manifest.md`.
