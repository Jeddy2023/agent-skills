---
description: Break work into small verifiable tasks with acceptance criteria and dependency ordering
disable-model-invocation: true
---

Invoke the agent-skills:planning-and-task-breakdown skill.

Read the existing spec (SPEC.md or equivalent) and the relevant codebase sections. Then:

1. Work read-only — no code changes; the plan and task list are the only files you write (this is not Claude Code's plan mode)
2. Identify the dependency graph between components
3. Slice work vertically (one complete path per task, not horizontal layers)
4. Write tasks with acceptance criteria and verification steps
5. Add checkpoints between phases
6. Present the plan for human review

Save the plan to tasks/plan.md and task list to tasks/todo.md. If the spec's `Status:` line is not Approved, stop and ask the human to approve it first.

If SPEC.md is a capability map, plan one module at a time: the arguments name the module id. Read spec/<module-id>.md and save to tasks/<module-id>/plan.md and tasks/<module-id>/todo.md. With no module id, ask which module to plan.

If tasks/plan.md or tasks/todo.md already exists with unchecked tasks for different work, stop and ask before writing — never silently overwrite an incomplete plan.
