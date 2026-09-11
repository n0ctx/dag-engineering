---
name: dag-engineering
description: "Use only for large-scale, multi-session code engineering: turning a PRD, roadmap, or big refactor into a DAG, or continuing an effort tracked by .dag/dag.json. NOT for small or single-session tasks, quick fixes, single-file changes, non-code work, or any request a couple of tool calls can finish — handle those directly without this skill."
---

# DAG Engineering

Compile large engineering work into a persistent DAG, then use it as the control plane across sessions.

## Invariants

- An explicit user decision outranks this skill's recommendations and subjective review findings. Preserve the finding, record the decision durably, and use the supported state or contract transition. It does not bypass runtime-enforced schema, acyclicity, path/scope, Git-SHA, evidence-binding, or external-authorization checks.

- Git is the code truth, project documentation is the knowledge truth, `.dag/dag.json` is the execution-state truth, and a session is disposable computation.
- The controller owns planning, scheduling, state transitions, review coordination, independent verification, integration, and final convergence. A node's normal loop is one worker, one fix-first reviewer round, and — whenever anything is still open after that — one controller close-out in which the controller itself judges, repairs, commits, and records through the same gates. Do not bounce in-scope defects back to the worker. If the contract is wrong or the worker returned `NEEDS_CONTEXT`, revise or re-dispatch and record the extra round. Planning runs the same loop at any stage: draft the plan, install or change it (`--init` or `--revise`), one fix-first decomposition review, one controller close-out of what the review left open, and the user's approval gate. A material change re-enters that loop; a narrow revision of execution maps, `read_first`, verification commands, or tighter scope on an approved plan keeps approval and warns.
- One project has one control plane: `.dag/` at the main worktree root, holding one `.dag/dag.json` at a time. A finished effort is archived before the next one starts.
- `.dag/dag.json` is only ever written by this skill's runtime scripts, including its creation, and carries an integrity seal that makes any other write a hard failure. Editing it with a file tool bypasses every gate and is never the shortcut it looks like; workers and reviewers do not touch it at all.

## Route

Define `SKILL_ROOT` as the directory containing this top-level `SKILL.md`. This is a documentation placeholder resolved by the execution agent from that top-level path; the host does not automatically inject an environment variable with this name. Nested modules inherit this top-level root and must not treat their own directory as the root.

Treat `${SKILL_ROOT}` as this skill's root. Quote it in every command.

Use planning when the user asks to turn a PRD, roadmap, issue, checklist, existing plan, large refactor, or multi-step request into a plan or DAG, or when no `.dag/dag.json` exists. Read:

```text
${SKILL_ROOT}/skills/dag-planning/SKILL.md
```

When no DAG existed at the start of the turn, that turn is planning-only unless the request points to a durable plan approved before this turn. “Start,” “do it,” and “execute” cannot approve a DAG the user has not seen. If planning creates `awaiting_approval` and the user has not approved this decomposition, present it and stop: do not load execution, record approval, dispatch, or edit project files.

Recording approval is a planning action. Once `approval_ref` is recorded, a user who also asked to execute may continue into execution in the same session; the runtime warns that independent execution is weaker. Do not wait for a new session.

Use execution when `.dag/dag.json` exists, `planning_status` is `approved`, and the user explicitly asks to run, execute, continue, resume, or finish the remaining work. Explicitly read:

```text
${SKILL_ROOT}/skills/dag-execution/SKILL.md
```

If a DAG exists but the request does not clearly ask to execute it, inspect or explain its state without dispatching workers. Invoking this skill never by itself authorizes broad parallel execution.

The nested `SKILL.md` files are modules loaded by this router. Do not rely on nested Skill discovery or invent separate `/dag-planning` or `/dag-execution` commands.
