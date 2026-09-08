---
name: dag-engineering
description: Use when planning or executing a long-lived, multi-step engineering effort from a PRD, roadmap, issue, checklist, existing plan, or persistent .dag/dag.json.
---

# DAG Engineering

Compile large engineering work into a persistent DAG, then use that DAG as the control plane across workers and sessions.

## Invariants

- Git is the code truth, project documentation is the knowledge truth, `.dag/dag.json` is the execution-state truth, and a session is disposable computation.
- The controller owns planning, scheduling, state transitions, review coordination, independent verification, integration, and final convergence. It is not the default implementation worker.
- One project has one control plane: `.dag/` at the main worktree root, holding one `.dag/dag.json` at a time. A finished effort is archived before the next one starts.
- Workers and reviewers never edit `.dag/dag.json`. Only the controller changes it, using this skill's runtime scripts after initial creation.
- A worker's or reviewer's claim is evidence, not acceptance. The controller runs at least one real check tied directly to each node's acceptance criteria before marking it done.
- `depends_on` expresses semantic ordering. `conflicts_with` expresses unsafe concurrent use of files or resources. Never substitute one for the other.
- Eligible work is not mandatory parallel work. Three is the hard concurrency ceiling; one is normal.

## Route

Treat `${CLAUDE_SKILL_DIR}` as this skill's root. Quote it in every command.

Use planning when the user asks to turn a PRD, roadmap, issue, checklist, existing plan, large refactor, or multi-step request into a plan or DAG, or when no `.dag/dag.json` exists. Explicitly read:

```text
${CLAUDE_SKILL_DIR}/skills/dag-planning/SKILL.md
```

When no DAG existed at the start of the turn, that turn is planning-only unless the request points to a durable plan that was already approved before this turn. “Start,” “do it,” “execute,” or equivalent wording cannot approve a DAG the user has not seen. If planning creates `awaiting_approval`, present it and end the turn: do not load execution, approve it, dispatch, or edit project implementation files.

Recording a user's approval of a pending DAG is a planning action: load the planning module, record it, and end the turn there. Approval never authorizes execution inside the session that planned the DAG.

Use execution when `.dag/dag.json` exists and the user explicitly asks to run, execute, continue, resume, or finish the remaining work. Explicitly read:

```text
${CLAUDE_SKILL_DIR}/skills/dag-execution/SKILL.md
```

If a DAG exists but the request does not clearly ask to execute it, inspect or explain its state without dispatching workers. Invoking this skill never by itself authorizes broad parallel execution.

The nested `SKILL.md` files are modules loaded by this router. Do not rely on nested Skill discovery or invent separate `/dag-planning` or `/dag-execution` commands.
