# Ship Manifest Format (shared reference)

This file is the single source of truth for the per-ticket folder, the `manifest.md`
schema, the review-log line format, and the handoff block. It is referenced by every
`ship-` skill and by the `/ship` dispatcher. It is NOT a skill.

## Per-ticket folder

Folder naming convention: `TICKET-ship` (written `<TICKET>-ship/` with the real id substituted).
All artifacts for one ticket live in a single folder at the workspace root:

```
<workspace-root>/<TICKET>-ship/
```

- `<TICKET>` is the ticket id (e.g. `PLAT-1000`). If there is no ticket, use a short
  kebab-case `<slug>-ship/` describing the work.
- The folder holds `manifest.md` plus ALL artifacts. Filenames inside the folder DROP the
  ticket prefix (e.g. `design.md`, not `PLAT-1000-design.md`).

### Folder resolution (every skill + the dispatcher)

1. Walk up from cwd looking for an existing `*-ship/` folder.
2. If found, present the path and ask the user to confirm it is the right ticket folder.
3. If none found, create `<TICKET>-ship/` (or `<slug>-ship/`) at the resolved workspace root
   and confirm the location with the user.

### Init-if-missing rule

Whichever `ship-` skill (or the dispatcher) runs first creates the folder and `manifest.md`.
Later stages assume it exists but MUST init defensively if it does not.

## `.ship-active` pointer

A `.ship-active` file at the workspace root names the active `<TICKET>-ship/` folder (a single
line: the folder name). Written by the dispatcher; read by the statusline and the dispatcher
to locate the active ticket when several `*-ship/` folders exist.

Single-active model (known limitation): `.ship-active` is one global pointer, so only ONE ticket
is "active" per workspace at a time. `/ship` overwrites it on each run (last-write-wins); switch
tickets with `/ship <ticket>`. The statusline reads this global pointer and is NOT thread-aware,
so with multiple tickets in flight it reflects the last-activated ticket, which may differ from
what a given thread is actually working on. Multiple ticket folders coexist safely; only the
active-pointer/statusline is single-valued.

## `manifest.md` schema

```markdown
# Ship Manifest: <TICKET>

## Status
- Current stage: <stage name>
- Next: <exact next command> (run in a FRESH thread)

## Repo Scope
Workspace root: <absolute path>

| Repo | Selected | Reason |
|------|----------|--------|
| api  | yes      | Hosts the GraphQL layer the ticket touches (from CLAUDE.md) |
| web  | no       | Frontend only; ticket is backend |
| infra| no       | No CLAUDE.md; name/path only, not relevant |

## Stages
| # | Stage | Status | Artifact |
|---|-------|--------|----------|
| 1a | Questions (repo: api) | done | research-questions-api.md |
| 1b | Questions (repo: web) | n/a  | - |
| 2a | Research (repo: api)  | pending | research-api.md |
| 2b | Research (repo: web)  | n/a  | - |
| 3 | Solution design | pending | design.md, design-questions.md |
| 4 | Work breakdown | pending | work-breakdown.md |
| 5 | Create plan | pending | plan.md |
| 6 | Implement plan | pending | - |

## Artifacts & Reviews
### design.md
- Authored: <stage>
- Review [antagonistic:architecture]: 2 issues, incorporated
- Review [antagonistic:security]: clean
- Review [manual-cross-thread]: pending
```

### Section rules

- **`## Status`** — `Current stage` and `Next`. `Next` is the EXACT next command plus the
  reminder "run in a FRESH thread".
- **`## Repo Scope`** — set by `ship-generate-questions`. Records the detected/confirmed
  workspace root and a `| Repo | Selected | Reason |` table listing EVERY candidate repo,
  whether it was selected, and a one-line relevance reason (from its `CLAUDE.md` when present,
  else name/path only). Records what was considered AND what was dropped. Nothing is hidden.
- **`## Stages`** — `| # | Stage | Status | Artifact |`. Both questions and research are
  multi-instance: rows `1a`, `1b`, ... labelled `Questions (repo: <name>)`, and rows `2a`,
  `2b`, ... labelled `Research (repo: <name>)`. The research rows are SEEDED (as `pending`) by
  `ship-generate-questions` from the confirmed scope, so research knows which repos to expect.
  Status enum: `pending | in-progress | done | n/a`.
- **`## Artifacts & Reviews`** — per artifact a `### <artifact>` heading, an `Authored:` line,
  then zero or more review lines.

### Research rows are a DERIVED VIEW

Each `Research (repo: <name>)` row is reconciled from artifact presence:
`research-<repo>.md` exists ⇒ `done`. Research artifact producers (Workflow research/synthesis
agents + standalone single-repo runs) NEVER write `manifest.md` — this is what keeps research
parallel-safe across terminals. Only the multi-repo orchestrator persists the reconciled rows,
and only after every artifact has landed.

## Single-writer rule

`manifest.md` is written ONLY by single-threaded contexts:
- `ship-generate-questions`
- the `ship-research-codebase` multi-repo ORCHESTRATOR (its main thread, single-threaded), ONLY to
  persist the `Research (repo: <name>)` reconcile after every `research-<repo>.md` has landed
- the planning trio (`ship-solution-design`, `ship-work-breakdown`, `ship-create-plan`)
- `ship-implement-plan` (ONLY to mark the `Implement plan` stage `done`, and ONLY when its plan file
  lives inside a `*-ship/` folder; standalone runs never touch a manifest)
- dispatcher transitions

Workflow research/synthesis agents and standalone single-repo research runs are PURE artifact
producers and MUST NOT write `manifest.md`. The multi-repo orchestrator above is the sole
research-stage writer, and only post-completion.

### Reconcile (idempotent)

Read the seeded `Research (repo: <name>)` rows; mark `done` those whose `research-<repo>.md`
exists in the folder. Safe to run repeatedly. Only single-threaded stages persist the result;
while concurrent terminals may still be running, derive the view in-memory and do NOT persist.

## Review-log line format

Used for per-lens antagonistic reviews and for manual cross-thread reviews:

```
- Review [<type>]: <outcome>
```

- `<type>` is one of:
  - `antagonistic:<lens>` (e.g. `antagonistic:security`) — full tier, one line per lens.
  - `antagonistic:combined(<lens,lens,...>)` — combined tier, one line listing every lens the
    single review pass covered.
  - `antagonistic:skipped` — skip tier; `<outcome>` is the reason (e.g. `trivial/mechanical change`).
  - `manual-cross-thread`.
- `<outcome>` is a short result, e.g. `2 issues, incorporated` / `clean` / `pending`.
- In EVERY tier, lenses that were not reviewed (dropped by relevance, or skipped) are recorded
  with their reason — nothing is silently omitted.

## Handoff block (printed by every skill on exit)

```
── Ship handoff ──
Stage: <X>/<Y> (<stage name>) complete
Artifacts: <files produced>
Next: <exact next command>
→ Start a FRESH thread before running the next stage.
```
