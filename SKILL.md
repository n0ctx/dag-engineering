---
name: dag-engineering
description: Use when planning or executing a long-lived, multi-step engineering effort from a PRD, roadmap, issue, checklist, existing plan, or persistent .dag/dag.json.
---

# DAG Engineering

Compile large engineering work into a persistent DAG, then use it as the control plane across sessions.

## Invariants

- An explicit user decision outranks this skill's recommendations and subjective review findings. Preserve the finding, record the decision durably, and use the supported state or contract transition. It does not bypass runtime-enforced schema, acyclicity, path/scope, Git-SHA, evidence-binding, or external-authorization checks.

- Git is the code truth, project documentation is the knowledge truth, `.dag/dag.json` is the execution-state truth, and a session is disposable computation.
- The controller owns planning, scheduling, state transitions, review coordination, independent verification, integration, and final convergence. A node's execution is bounded: one worker, one fix-first reviewer round, and — whenever anything is still open after that — one controller close-out in which the controller itself judges, repairs, commits, and records through the same gates. A node never bounces between worker and reviewer for repeated rework. Planning is bounded the same way: one full decomposition review, at most one repair review after the amending pass, then remaining open items go to the user at the approval gate rather than into another review round.
- One project has one control plane: `.dag/` at the main worktree root, holding one `.dag/dag.json` at a time. A finished effort is archived before the next one starts.
- `.dag/dag.json` is only ever written by this skill's runtime scripts, including its creation, and carries an integrity seal that makes any other write a hard failure. Editing it with a file tool bypasses every gate and is never the shortcut it looks like; workers and reviewers do not touch it at all.

## Route

Define `SKILL_ROOT` as the directory containing this top-level `SKILL.md`. This is a documentation placeholder resolved by the execution agent from that top-level path; the host does not automatically inject an environment variable with this name. Nested modules inherit this top-level root and must not treat their own directory as the root.

Treat `${SKILL_ROOT}` as this skill's root. Quote it in every command.

Use planning when the user asks to turn a PRD, roadmap, issue, checklist, existing plan, large refactor, or multi-step request into a plan or DAG, or when no `.dag/dag.json` exists. Read:

```text
${SKILL_ROOT}/skills/dag-planning/SKILL.md
```

When no DAG existed at the start of the turn, that turn is planning-only unless the request points to a durable plan approved before this turn. “Start,” “do it,” and “execute” cannot approve a DAG the user has not seen. If planning creates `awaiting_approval`, present it and end the turn: do not load execution, approve it, dispatch, or edit project files.

Recording approval of a pending DAG is a planning action: load planning, record it, and end the turn there. Approval never authorizes execution in the planning session.

Use execution when `.dag/dag.json` exists and the user explicitly asks to run, execute, continue, resume, or finish the remaining work. Explicitly read:

```text
${SKILL_ROOT}/skills/dag-execution/SKILL.md
```

If a DAG exists but the request does not clearly ask to execute it, inspect or explain its state without dispatching workers. Invoking this skill never by itself authorizes broad parallel execution.

The nested `SKILL.md` files are modules loaded by this router. Do not rely on nested Skill discovery or invent separate `/dag-planning` or `/dag-execution` commands.
