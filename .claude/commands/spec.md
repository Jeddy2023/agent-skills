---
description: Start spec-driven development — write a structured specification before writing code
disable-model-invocation: true
---

Invoke the agent-skills:spec-driven-development skill.

Begin by understanding what the user wants to build. Ask clarifying questions about:
1. The objective and target users
2. Core features and acceptance criteria
3. Tech stack preferences and constraints
4. Known boundaries (what to always do, ask first about, and never do)

Then generate a structured spec using the skill's template: objective, non-goals, the six core areas, success criteria, and open questions. Where the project's rules files (CLAUDE.md, AGENTS.md, CONSTRAINTS.md) already define commands, structure, style, testing thresholds, or boundaries, link to them instead of restating them.

If the request bundles several independently testable capabilities, first propose a capability map (module ids, dependency direction, build order) per the skill's Phase 0 and get it approved, then spec each module in dependency order. Save the map as SPEC.md and each module spec as spec/<module-id>.md.

Save a single-capability spec as SPEC.md in the project root. Then summarize it, list open questions, ask for approval, and stop; do not plan or write code in the same turn.
