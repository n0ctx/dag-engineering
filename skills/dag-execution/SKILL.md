---
name: dag-execution
description: Internal module resuming and controlling implementation from an existing .dag/dag.json; load only through dag-engineering.
---

# DAG Execution

Use the existing DAG as the control plane. Do not reconstruct the project from chat history.

## Load only what execution needs

Always read:

- `${SKILL_ROOT}/references/node-contract.md`
- `${SKILL_ROOT}/references/verification.md`

Read `${SKILL_ROOT}/references/scheduling.md` only when there are multiple ready candidates, concurrency, conflicts, worktrees, or a scheduling choice. Read `${SKILL_ROOT}/references/review-protocol.md` when a node returns for review. Read `${SKILL_ROOT}/references/dag-schema.md` only to diagnose or repair invalid state, and `${SKILL_ROOT}/references/decomposition.md` only to re-slice or replan.

## Cold start and recovery

At the start of every session:

```bash
"${SKILL_ROOT}/scripts/validate-dag"
"${SKILL_ROOT}/scripts/status"
git status --short --branch
git log --oneline --decorate -12
```

Read only the DAG's source references needed for current decisions. Trust Git, project artifacts, and the DAG over chat recollection.

If a node is `running`, reconcile its recorded worktree, branch, base/head, review rounds, handoff, and diff before dispatching anything. Recoverable work goes through the review gate. If completion cannot be proven, settle the attempt:

```bash
"${SKILL_ROOT}/scripts/update-task" <node-id> failed --reason "<what was and was not established>"
```

Never replay work only because a session disappeared, and never leave a vanished worker's node `running`. A worker with no handoff artifact has not reported. Do not start nodes unless `planning_status` is `approved`; if a command reports an external control-file write, stop and inspect rather than reseal past the gate.

## Select work

Obtain eligible candidates:

```bash
"${SKILL_ROOT}/scripts/ready-tasks" --json
```

The result means pending, dependencies done, and no active conflict. If there is one candidate, serial execution is the default. If there are several, load scheduling and choose using independence, wall-clock benefit, shared state, worktree cost, and integration risk; never dispatch merely to fill slots.

## Dispatch

Mark the node running before dispatch:

```bash
"${SKILL_ROOT}/scripts/update-task" <node-id> running \\
  --worktree "<absolute or project-relative worktree>" \\
  --branch "<expected branch>" --base-ref "<expected HEAD>"
"${SKILL_ROOT}/scripts/status" --node <node-id>
```

Send a fresh worker only the emitted node package, upstream output and handoff references, required project constraints, worktree, and an absolute handoff path. Apply `node-contract.md`'s reading boundary and do not provide the whole DAG, chat history, or unrelated reports. Serial work uses the current clean worktree when allowed; concurrent work uses isolated worktrees and branches. The worker commits only its node and never edits the DAG or merges.

## Review gate

Run the scope check before paying for review:

```bash
"${SKILL_ROOT}/scripts/check-scope" <node-id> --head-ref "<reported-sha>" \\
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json"
```

A scope violation is returned to the worker or failed; do not dispatch a reviewer. Otherwise load `review-protocol.md` and give a fresh reviewer the original contract, immutable `base_ref..worker_head` diff, handoff, and relevant evidence. The reviewer may make one independent fix commit for an in-scope defect; outside-scope, unsafe-repair, and contract findings go to the controller.

Record the verdict only through runtime:

```bash
"${SKILL_ROOT}/scripts/update-task" <node-id> review \\
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json" \\
  --review-ref ".dag/artifacts/<node-id>/review-<round>.json" \\
  --head-ref "<reviewed-sha>"
```

The runtime derives `approved`, `needs_fixes`, `cannot_verify`, or `escalated` and binds the head. There is no worker/reviewer ping-pong: resolve same-head escalations or contract decisions in the controller.

## Master verification gate

Apply `${SKILL_ROOT}/references/verification.md`. The controller checks scope, reproducible evidence, review against the actual contract and diff, and at least one real acceptance criterion. Only then record completion:

```bash
"${SKILL_ROOT}/scripts/update-task" <node-id> done \\
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json" \\
  --verification-ref ".dag/artifacts/<node-id>/master-verification.json" \\
  --head-ref "<reviewed-sha>" --summary "<short result>"
```

If an attempt fails or is blocked, record the reason and durable evidence. Never put a transcript in `dag.json`; do not fix implementation code inline while acting as the independent acceptance check.

## Re-slice or replan

When execution shows that the graph is wrong, reorganize the plan instead of widening a node or silently carrying a dead one:

```bash
"${SKILL_ROOT}/scripts/update-task" --revise ".dag/revision.json" --reason "<one line>"
```

Use a patch naming only moved, added, dropped, or re-briefed nodes. Nodes that ran retain status, attempts, and handoff; `done` and `running` contracts are frozen. Dropping a node or changing `outputs` or `depends_on` changes a downstream interface and requires the scoped review described in `${SKILL_ROOT}/references/decomposition.md`.

Changing `objective`, `global_acceptance`, `assumptions`, or `source_refs` changes the project promise: record `replan_required`, amend, review, and obtain approval again. Correct a `failed` or `blocked` contract in place. Never rewrite a completed node's record; use a new node to undo accepted work.

```bash
"${SKILL_ROOT}/scripts/update-task" --planning-status replan_required --reference "<durable path recording why>"
"${SKILL_ROOT}/scripts/update-task" --amend ".dag/amendment.json"
```

## Integrate and converge

Integrate only nodes that passed worker, reviewer, and master gates, in semantic dependency order. Preserve approximately one logical commit per node. The controller may resolve mechanical conflicts; behavior or interface choices require replan. Run integration checks after related nodes land. After a parallel node passes acceptance and is successfully integrated, clean up its temporary worktree and temporary local branch; do not clean either when it is not integrated, has uncommitted changes, or safety cannot be confirmed.

All nodes being `done` is necessary, not sufficient. Re-read the objective, source references, global acceptance, and integrated behavior; perform a fresh whole-plan convergence review from `status --plan-view`. If a gap exists, record it and import explicit gap nodes:

```bash
"${SKILL_ROOT}/scripts/update-task" --convergence failed \\
  --reference ".dag/artifacts/convergence-review.md" --gap "<short gap>"
"${SKILL_ROOT}/scripts/update-task" --add-gap-nodes ".dag/artifacts/gap-plan.json"
```

A passing convergence permits completion and archival:

```bash
"${SKILL_ROOT}/scripts/update-task" --convergence passed \\
  --reference ".dag/artifacts/convergence-review.md"
"${SKILL_ROOT}/scripts/update-task" --planning-status complete \\
  --reference ".dag/artifacts/convergence-review.md"
"${SKILL_ROOT}/scripts/update-task" --archive
```

Announce completion only when every node is done, convergence passed, and `planning_status` is `complete`.
