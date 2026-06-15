# Claude Code Skills

A collection of [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) for structured software delivery, built around the **Ship pipeline**.

## Quick Reference

| Skill | Description | Install |
|-------|-------------|---------|
| `/ship` | Pipeline dispatcher — runs the next stage from the manifest | `npx skills@latest add dfollent/skills/ship` |
| `/ship-generate-questions` | Repo scoping + intent-neutral research questions (stage 1) | `npx skills@latest add dfollent/skills/ship-generate-questions` |
| `/ship-research-codebase` | Objective codebase research from questions (stage 2) | `npx skills@latest add dfollent/skills/ship-research-codebase` |
| `/ship-solution-design` | Solution design with grill-me walk + antagonistic review (stage 3) | `npx skills@latest add dfollent/skills/ship-solution-design` |
| `/ship-work-breakdown` | High-level phase breakdown for human review (stage 4) | `npx skills@latest add dfollent/skills/ship-work-breakdown` |
| `/ship-create-plan` | Detailed tactical plan, hardened by antagonistic review (stage 5) | `npx skills@latest add dfollent/skills/ship-create-plan` |
| `/ship-implement-plan` | Phase-by-phase plan execution with verification (stage 6) | `npx skills@latest add dfollent/skills/ship-implement-plan` |
| `/grill-me` | Stress-test a plan or design | `npx skills@latest add dfollent/skills/grill-me` |
| `/project-orient` | Load project context from Confluence | `npx skills@latest add dfollent/skills/project-orient` |
| `/project-update` | Write findings back to Confluence | `npx skills@latest add dfollent/skills/project-update` |
| `/project-lint` | Audit Confluence doc tree for quality | `npx skills@latest add dfollent/skills/project-lint` |

Install all skills at once:

```bash
npx skills@latest add dfollent/skills
```

## Required: subagent + shared assets

The Ship skills depend on two things the skills installer does **not** copy (they have no `SKILL.md`). Install them manually:

```bash
# 1. The antagonistic-reviewer subagent (used by stages 3 and 5).
#    Subagents live in agents/, NOT inside skill folders — Claude Code discovers
#    them from ~/.claude/agents/ (personal) or .claude/agents/ (project).
cp agents/antagonistic-reviewer.md ~/.claude/agents/

# 2. The shared manifest/coverage references every ship skill reads by path.
cp -R _ship-shared ~/.claude/skills/_ship-shared
```

Skills and subagents are separate mechanisms: a skill cannot bundle a subagent. The ship skills invoke `antagonistic-reviewer` by name (`subagent_type: antagonistic-reviewer`), which only resolves if the agent file is installed under an `agents/` directory. The ship skills reference the shared files at `~/.claude/skills/_ship-shared/`, so that folder must live there.

## The Ship pipeline

Skills that break delivery into focused stages. Each stage runs in its own context window, produces a static markdown artefact in a per-ticket `<TICKET>-ship/` folder, and feeds into the next. Human review concentrates where it has the most leverage.

The `/ship` dispatcher is an on-rails orchestrator: it resolves the active per-ticket manifest and runs **exactly one** stage per invocation. Every `ship-` skill is also independently runnable on its own.

### The flow

**1. `/ship-generate-questions`** — Scopes which repos in a multi-repo workspace the ticket touches (it legitimately sees the ticket), then writes one intent-neutral research-questions file per repo plus the per-ticket manifest.

**2. `/ship-research-codebase`** — Takes only the questions (never sees the ticket). Researches each repo in-thread, or fans out a Workflow (one agent per question theme). Outputs an objective `research-<repo>.md` per repo. No opinions.

**3. `/ship-solution-design`** — Takes research + ticket. Produces a solution design covering the coverage map, driven by an interactive grill-me decision-tree walk and hardened by tiered antagonistic review. First review point.

**4. `/ship-work-breakdown`** — Takes the design. Produces a high-level phase breakdown: order of changes, what each phase delivers, testing strategy. Second review point.

**5. `/ship-create-plan`** — Takes work breakdown + design + research. Produces a detailed tactical plan the agent executes, hardened by a design-lens antagonistic-reviewer pass. Skim, no full review needed.

**6. `/ship-implement-plan`** — Execute the plan, phase by phase, with verification and a pause for manual checks between phases.

### Why it works

- **You review the design and breakdown, not 1000 lines of plan or code.** That's where leverage is highest.
- **Each skill stays under 40 instructions.** Smaller prompts = fewer skipped steps.
- **Research is unbiased.** The questions stage sees the intent; the research stage doesn't. Prevents cherry-picking facts that support what the model already wants to build.
- **Every stage produces a standalone markdown doc** in the per-ticket folder. Resume from any point in a fresh session.
- **Antagonistic review** stress-tests the design and plan through adversarial single-lens passes before any code is written.

### Skipping steps

Not every task needs all six stages. Use your judgement:

- **Small bug fix?** Skip straight to implementation.
- **Well-understood change?** Start at `/ship-solution-design` or `/ship-create-plan`.
- **Complex feature in unfamiliar code?** Run the full flow via `/ship`.

## Project Skills (Confluence)

Skills for project work in Confluence. One reads from the project surface, the others write back to it or audit it.

### `/project-orient`

Load context from Confluence without a meeting or handover.

- "What projects do we have?" — lists all projects in a space
- "Orient me to project X" — project tree, open questions, blockers, unvalidated assumptions, recent decisions
- "I want to work on the API task" — loads the target doc + parent + project root, surfaces anything that might affect the work

### `/project-update`

Write findings back so the project surface stays current.

- Appends to existing docs or creates child pages, with provenance (who, when, which tool)
- Inline tags for open questions, assumptions, blockers, decisions, actions
- Tags get resolved in-place: `[ASSUMPTION -> CONFIRMED]`, `[BLOCKER -> CLEARED]`
- Captures implementation write-backs: PR links, branch names, learnings, rejected assumptions

### `/project-lint`

Audits a project's Confluence document tree for quality, consistency, and structure. Finds stale items, contradictions, orphan pages, and missing tags.

### Why this matters

- No one needs a handover meeting to pick up work. `/project-orient` self-serves into any project.
- No one writes status reports. `/project-update` captures state as a side effect of doing the work.
- AI sessions can scan the tags programmatically. The project surface is machine- and human-readable.

## Utility Skills

### `/grill-me`

Stress-test a plan or design. Claude interviews you relentlessly about every aspect, walking down each branch of the decision tree and resolving dependencies one by one. Best used after `/ship-solution-design` or `/ship-work-breakdown`.

Attribution: [Matt Pocock](https://github.com/mattpocock/skills/blob/main/grill-me/SKILL.md) (MIT License).

## Acknowledgements

The coding workflow is informed by [Dex Horthy's](https://github.com/humanlayer) CRISPY methodology and the evolution from Research-Plan-Implement (RPI). Several skills are derived from or inspired by [HumanLayer's](https://github.com/humanlayer/humanlayer) Claude Code commands (Apache 2.0). See [NOTICE](NOTICE) for full attribution.
