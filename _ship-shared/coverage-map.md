# Ship Coverage Map (shared reference)

The dimensions a design must address. Referenced by `ship-solution-design` (question walk +
authoring + review fan-out) and by `ship-create-plan`. NOT a skill.

## Dimensions

1. **Product / requirements** — the problem, the users, the acceptance criteria, scope
   boundaries. Technical dimensions depend on this, so it is addressed first.
2. **Architecture & codebase-pattern fit** — how the change sits in the existing system;
   reuse of existing patterns over inventing new ones; module boundaries.
3. **Security** — authn/authz, data exposure, input validation, secrets, schema-edge typing.
4. **Performance & scalability** — hot paths, query/load patterns, limits, growth.
5. **Reliability & operations** — observability/logging, failure modes, rollback, runbooks,
   SLAs, alerting.
6. **Data & integration** — schemas, contracts, events, migrations, backward compatibility.

## Framing rule: OUTPUT CONTRACT, not a process

This is a contract on the OUTPUT, not a prescribed workflow. The author is a black box; only
the output coverage is mandated. Do NOT recreate a human design ritual to "hit" each box.

## Relevance by judgment (option b)

The agent assesses each dimension's relevance for the specific ticket. Each dimension gets
EITHER substantive coverage OR an explicit `N/A — <reason>`. Both count as coverage. A
dimension is never silently omitted.

## Same set seeds both the walk and the review

The dimension set above seeds:
1. the `ship-solution-design` decision-tree question walk (touch every relevant dimension), and
2. the antagonistic review of the design, scoped to the relevant dimensions.

The review is TIERED, chosen at a checkpoint (see `ship-solution-design` Step 5): **full**
(one reviewer per relevant lens, parallel), **combined** (one reviewer over all relevant lenses
in a single pass), or **skip** (trivial/mechanical change). In EVERY tier, any lens/dimension not
reviewed — dropped by relevance OR skipped — has its reason recorded to the manifest. Nothing is
silently omitted.
