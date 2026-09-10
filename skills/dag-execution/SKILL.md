---
name: dag-execution
description: Internal module resuming and controlling implementation from an existing .dag/dag.json; load only through dag-engineering.
---

# DAG Execution

Use the existing DAG as the control plane. Do not reconstruct the project from chat history.

## Load and recover

Always read `${SKILL_ROOT}/references/node-contract.md` and `${SKILL_ROOT}/references/verification.md`. Read scheduling only when choosing among multiple ready nodes; read review protocol when a node returns; read schema or decomposition only for the corresponding diagnosis or replan.

At session start, run:

```bash
"${SKILL_ROOT}/scripts/validate-dag"
"${SKILL_ROOT}/scripts/status"
git status --short --branch
git log --oneline --decorate -12
```

Trust Git, project artifacts, and the DAG over chat recollection. Reconcile every `running` node's worktree, branch, base/head, review rounds, handoff, and diff before dispatch. A vanished worker without a handoff is not complete; settle an unprovable attempt with `update-task <node-id> failed --reason "..."`. Do not start work unless `planning_status` is `approved`.

## Select and dispatch

派发前追加以下读取边界：合同已经直接写出的结论就是 worker 的工作事实；除 `read_first` 外最多再读 5 个文件，每个文件只为落实一个具名主张或验收问题，不得做仓库调查。`scope.files` 是写边界；`scope.forbidden` 只是其他节点的写保护区，不是阅读任务，也不要为了熟悉上下文主动打开。若在预算内仍有不影响合同完成的未决问题，完成工作并在 handoff 以 `DONE_WITH_CONCERNS` 登记主张、证据和残余风险；只有问题阻塞合同或要求改变合同、scope、接口或架构时才返回 `NEEDS_CONTEXT`。

Use:

```bash
"${SKILL_ROOT}/scripts/ready-tasks" --json
```

When one node is ready, run it serially. When several are ready, load scheduling and choose based on independence, conflicts, worktree cost, wall-clock benefit, and integration risk.

Mark the node running before dispatch:

```bash
"${SKILL_ROOT}/scripts/update-task" <node-id> running \
  --worktree "<absolute-or-project-relative-worktree>" \
  --branch "<expected-branch>" --base-ref "<expected-HEAD>"
"${SKILL_ROOT}/scripts/status" --node <node-id>
```

Invoke `status --node` once and forward its original node brief once, unchanged. Add only the worktree, absolute handoff path, and binding project constraints. Do not restate node fields or send the whole DAG, unrelated upstream material, worker reasoning, or a future reviewer prompt.

Use this fixed short prefix before the raw brief:

> Read `read_first` first. Follow `execution_plan`; write only paths matched by `scope.files`. If reality conflicts with the contract, return `NEEDS_CONTEXT`. Run verification, commit the node, and write the handoff.

The worker may use an equivalent local implementation. Changing `scope`, interfaces, outputs, architecture, or acceptance is a contract change: stop and report it; the controller must revise or replan, never silently widen the node.

## Review gate

Run the scope gate before dispatching review:

```bash
"${SKILL_ROOT}/scripts/check-scope" <node-id> --head-ref "<reported-sha>" \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json"
```

Do not dispatch review after a scope violation. Otherwise provide the strict package in `references/review-protocol.md`. Record the verdict only through the runtime:

```bash
"${SKILL_ROOT}/scripts/update-task" <node-id> review \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json" \
  --review-ref ".dag/artifacts/<node-id>/review-<round>.json" \
  --head-ref "<reviewed-sha>"
```

The runtime derives the review outcome from the artifact and binds the head. Dispatch the reviewer with edit and commit capability in the node's worktree: it repairs every in-scope defect on the spot in one independent fix commit within the same session and cannot approve its own fix. The review artifact records repaired defects as `bugs` (which require the reviewer's `fix_commit`) and everything the reviewer cannot settle alone as `unsure` with a reason. There is no report-only round that returns in-scope defects to the worker. Contract, scope, unsafe-repair, and caller decisions return to the controller as `unsure` entries.

## Controller close-out

The reviewer round is the last dispatch. Whatever is still open after it — a remaining defect, a gate rejecting an artifact over a fixable discrepancy, an `unsure` item the controller can settle from evidence — the controller closes itself: make the final judgment, repair in the node's worktree, commit, and record the closing review, handoff, and verification artifacts through the same runtime gates. Never send the node back for another worker or reviewer round; the whole loop is worker, reviewer, and at most this one close-out. The runtime binds commits and artifacts, not authorship — the close-out faces the same gates, and every recorded check must actually have run.

## Master verification

Apply `references/verification.md`. The controller independently checks scope, reproducible evidence, the actual contract and diff, and at least one real acceptance criterion. Record completion only after creating the handoff and master-verification artifacts:

```bash
"${SKILL_ROOT}/scripts/update-task" <node-id> done \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json" \
  --verification-ref ".dag/artifacts/<node-id>/master-verification.json" \
  --head-ref "<reviewed-sha>" --summary "<short result>"
```

Do not fix implementation code inline while acting as the independent acceptance check.

`--head-ref` binds the review and the completion to a real commit. When the node produced no commit — an investigation, a decision recorded in an artifact — omit it from both the review and the done command, and the artifacts carry null refs and empty file lists instead of a commit binding.

## Replan and converge

If execution shows that the graph or contract is wrong, change the plan with the same loop planning uses; do not widen a node silently. `--revise` replaces the plan from a draft or patch, preserves every execution record, and returns it to `awaiting_approval` for one full review round and the approval gate, exactly like a new plan. Mark a live plan for replanning when the deliverable itself is in doubt:

```bash
"${SKILL_ROOT}/scripts/update-task" --revise ".dag/revision.json" --reason "<one line>"
"${SKILL_ROOT}/scripts/update-task" --planning-status replan_required \
  --reference "<durable path recording why>"
"${SKILL_ROOT}/scripts/update-task" --revise ".dag/replan.json" --reason "<one line>"
```

Integrate only nodes that passed worker, review, and master gates, in semantic dependency order. After related nodes land, run integration checks and a whole-plan convergence review. If a gap exists, record it and add explicit gap nodes:

```bash
"${SKILL_ROOT}/scripts/update-task" --convergence failed \
  --reference ".dag/artifacts/convergence-review.md" --gap "<short gap>"
"${SKILL_ROOT}/scripts/update-task" --add-gap-nodes ".dag/artifacts/gap-plan.json"
```

Only a passed convergence permits completion and archival:

```bash
"${SKILL_ROOT}/scripts/update-task" --convergence passed \
  --reference ".dag/artifacts/convergence-review.md"
"${SKILL_ROOT}/scripts/update-task" --planning-status complete \
  --reference ".dag/artifacts/convergence-review.md"
"${SKILL_ROOT}/scripts/update-task" --archive
```

Announce completion only when every node is `done`, convergence is `passed`, and `planning_status` is `complete`.
