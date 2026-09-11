# dag-engineering

An Agent Skill for planning and controlling large-scale, multi-session code engineering through a persistent directed acyclic graph (DAG).

## Scope

Use this Skill when a request involves a PRD, roadmap, issue, checklist, existing plan, large refactor, or other work that needs durable decomposition and execution across sessions.

It is intentionally not for small single-session changes, quick fixes, isolated file edits, or ordinary non-code work.

## Core model

The controller owns planning, scheduling, state transitions, review coordination, independent verification, integration, and final convergence. Workers execute bounded node contracts; they do not directly rewrite the control plane.

The target repository's `.dag/` directory is the control plane. `.dag/dag.json` stores the current plan and execution state. The runtime validates paths, scope, dependencies, conflicts, evidence, Git references, and state transitions, seals the control file after writes, and warns on process-cost issues that do not break those facts.

The main repository remains the source of code truth. Project documentation remains the source of knowledge truth. The DAG remains the source of execution-state truth.

## Lifecycle

### Planning

Planning normalizes the source, records durable references, separates facts from assumptions, identifies ownership and boundaries, and decomposes the work into node contracts. A new or materially changed plan enters the approval gate after one fix-first decomposition review and controller close-out. Planning stops until the user explicitly approves the plan. A narrow revision of execution maps, `read_first`, verification commands, or tighter scope on an approved plan keeps that approval.

### Execution

Execution dispatches only approved ready nodes, records bounded attempts and handoffs, runs one fix-first review round, and performs independent master verification. A node is not complete until its scope, commit or artifact, verification evidence, and handoff satisfy the runtime contract.

### Replanning and convergence

When implementation reveals that a contract or graph is wrong, revise the plan through the runtime. A material change restarts the review and approval loop; a narrow revision keeps approval. Integrate completed nodes in dependency order, run whole-plan checks, record gaps as explicit nodes, and mark the DAG complete only after every node is done and convergence has passed.

## Runtime entry points

The repository includes shell-neutral executable entry points for:

- `check-scope` — validate a node's scope and current repository boundary
- `preflight-plan` — run planning preflight checks
- `ready-tasks` — list nodes eligible for dispatch
- `status` — inspect plan, node, or review-request state
- `update-task` — create, revise, approve, execute, complete, converge, or archive state through the runtime
- `validate-dag` — validate or update the DAG state
- `self-test` — run isolated runtime tests

Use the entry-point help and the relevant reference files for exact arguments. Do not write `.dag/dag.json` directly.

## Repository layout

```text
dag-engineering/
├── SKILL.md
├── README.md
├── references/
│   ├── dag-schema.md
│   ├── decomposition.md
│   ├── node-contract.md
│   ├── provenance.md
│   ├── review-protocol.md
│   ├── scheduling.md
│   └── verification.md
├── scripts/
│   ├── check-scope
│   ├── preflight-plan
│   ├── ready-tasks
│   ├── self-test
│   ├── status
│   ├── update-task
│   └── validate-dag
└── skills/
    ├── dag-execution/SKILL.md
    └── dag-planning/SKILL.md
```

Read [SKILL.md](SKILL.md) first. It routes planning and execution to the appropriate nested module. Use the references for schema, decomposition, node contracts, provenance, review, scheduling, and verification details.

## License

MIT. See [LICENSE](LICENSE).
