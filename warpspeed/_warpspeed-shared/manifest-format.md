# Warpspeed Manifest Format (shared reference)

This file is the single source of truth for the per-ticket folder, the `manifest.md`
schema, the review-log line format, and the handoff block. It is referenced by every
warpspeed pipeline skill and by the `/warpspeed` coordinator. It is NOT a skill.

## The per-ticket folder

All artifacts for one ticket — `manifest.md` plus every stage output — live in a single per-ticket
folder ("the folder" throughout this doc). Filenames inside it DROP any ticket prefix (e.g.
`design.md`, not `PLAT-1000-design.md`).

The folder is a `<TICKET>-warpspeed/` directory at the workspace root (`<TICKET>` = ticket id, e.g.
`PLAT-1000`; no ticket ⇒ a short kebab-case `<slug>-warpspeed/`). Resolve it by:

1. Walk up from cwd looking for an existing `*-warpspeed/` folder.
2. If found, present the path and ask the user to confirm it is the right ticket folder.
3. If none found, create it at the resolved workspace root and confirm the location.

### Init-if-missing rule

Whichever skill (or the coordinator) runs first creates the folder and `manifest.md`. Later stages
assume it exists but MUST init defensively if it does not.

## `.warpspeed-active` registry

A `.warpspeed-active` file at the workspace root lists the known `<TICKET>-warpspeed/` folders, one
folder name per line. The coordinator APPENDS a folder's name the first time it works that ticket
(it never overwrites existing entries) and reads the file to locate ticket folders when several
`*-warpspeed/` folders exist.

It is a registry of known tickets, not a single "active" pointer: which ticket a run acts on is
decided by cwd context, falling back to asking the user when the target is ambiguous. Multiple
ticket folders coexist safely; entries are append-only.

## `manifest.md` schema

```markdown
# Warpspeed Manifest: <TICKET>

## Status
- Current stage: <stage name>
- Next: <exact next command> (run in a FRESH thread)
- Next-Command: <one of: /research-codebase | /design | /work-breakdown | /create-plan | /implement-plan | DONE>

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
```

### Section rules

- **`## Status`** — `Current stage`, `Next`, and `Next-Command`. `Next` is the human-readable next
  command plus the "run in a FRESH thread" reminder. `Next-Command` is the MACHINE-READABLE form: a
  single token on its own line — one of `/research-codebase | /design | /work-breakdown |
  /create-plan | /implement-plan | DONE` — so tooling can read the next step unambiguously without
  parsing prose. Every stage, on successful completion, sets BOTH `Next` and `Next-Command` to point
  at the next stage.
- **`## Repo Scope`** — set by `generate-questions`. Records the detected workspace root and a
  `| Repo | Selected | Reason |` table listing EVERY candidate repo, whether it was selected,
  and a one-line relevance reason (from its `CLAUDE.md` when present, else name/path only). Repo
  selection is AUTONOMOUS (no user confirmation); this table is the transparency record — the
  user can edit it to re-scope. Records what was considered AND what was dropped. Nothing is hidden.
- **`## Stages`** — `| # | Stage | Status | Artifact |`. Both questions and research are
  multi-instance: rows `1a`, `1b`, ... labelled `Questions (repo: <name>)`, and rows `2a`,
  `2b`, ... labelled `Research (repo: <name>)`. The research rows are SEEDED (as `pending`) by
  `generate-questions` from the confirmed scope, so research knows which repos to expect.
  Status enum: `pending | in-progress | done | n/a`.
- **`## Artifacts & Reviews`** — per artifact a `### <artifact>` heading, an `Authored:` line,
  then zero or more review lines. There is NO manual-review line — warpspeed runs unattended
  except for the design grill-me walk; review is the autonomous antagonistic pass only.

### Research rows are a DERIVED VIEW

Each `Research (repo: <name>)` row is reconciled from artifact presence:
`research-<repo>.md` exists ⇒ `done`. Research artifact producers (Workflow research/synthesis
agents + standalone single-repo runs) NEVER write `manifest.md` — this is what keeps research
parallel-safe across terminals. Only the multi-repo orchestrator persists the reconciled rows,
and only after every artifact has landed.

## Single-writer rule

`manifest.md` is written ONLY by single-threaded contexts:
- `generate-questions`
- the `research-codebase` multi-repo ORCHESTRATOR (its main thread, single-threaded), ONLY to
  persist the `Research (repo: <name>)` reconcile after every `research-<repo>.md` has landed
- the planning trio (`design`, `work-breakdown`, `create-plan`)
- coordinator transitions

Workflow research/synthesis agents and standalone single-repo research runs are PURE artifact
producers and MUST NOT write `manifest.md`. The multi-repo orchestrator above is the sole
research-stage writer, and only post-completion.

`implement-plan` (stage 6) is the terminal execution stage and is NOT warpspeed-aware: the
`/implement-plan` skill itself does NOT write `manifest.md`. The coordinator marks stage 6 as the
handoff target and stops; the pipeline is considered "in implementation" once it hands off.

### Reconcile (idempotent)

Read the seeded `Research (repo: <name>)` rows; mark `done` those whose `research-<repo>.md`
exists in the folder. Safe to run repeatedly. Only single-threaded stages persist the result;
while concurrent terminals may still be running, derive the view in-memory and do NOT persist.

## Review-log line format

Used for the autonomous antagonistic reviews:

```
- Review [<type>]: <outcome>
```

- `<type>` is one of:
  - `antagonistic:<lens>` (e.g. `antagonistic:security`) — full tier, one line per lens.
  - `antagonistic:combined(<lens,lens,...>)` — combined tier, one line listing every lens the
    single review pass covered.
  - `antagonistic:skipped` — skip tier; `<outcome>` is the reason (e.g. `trivial/mechanical change`).
  - `antagonistic:design-lens` — the create-plan adversarial review pass.
  - `design-author` — the create-plan design-author review pass: reconstructs the design author's
    context from `design.md` + `design-questions.md` (incl. rejected alternatives) and checks the
    plan for faithfulness to design intent + refinements. Runs alongside the adversarial pass.
- `<outcome>` is a short result, e.g. `2 issues, incorporated` / `clean`.
- In EVERY tier, lenses that were not reviewed (dropped by relevance, or skipped) are recorded
  with their reason — nothing is silently omitted.

## Handoff block (printed by every skill on exit)

```
── Warpspeed handoff ──
Stage: <X>/<Y> (<stage name>) complete
Artifacts: <files produced>
Next: <exact next command>
→ Start a FRESH thread before running the next stage.
```
