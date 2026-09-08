---
name: dag-execution
description: Internal module for resuming and controlling implementation from an existing .dag/dag.json; load only through dag-engineering.
---

# DAG Execution

Use the existing DAG as the control plane. Do not reconstruct the project from old chat history.

## Load only what execution needs

Always read:

- `${CLAUDE_SKILL_DIR}/references/scheduling.md`
- `${CLAUDE_SKILL_DIR}/references/node-contract.md`
- `${CLAUDE_SKILL_DIR}/references/verification.md`

Read `${CLAUDE_SKILL_DIR}/references/review-protocol.md` when a node comes back for review, not at session start. Read `${CLAUDE_SKILL_DIR}/references/dag-schema.md` only to diagnose or repair invalid state. Read `${CLAUDE_SKILL_DIR}/references/decomposition.md` only when re-slicing or replanning is required.

## Cold start and recovery

At the start of every session:

```bash
"${CLAUDE_SKILL_DIR}/scripts/validate-dag"
"${CLAUDE_SKILL_DIR}/scripts/status"
git status --short --branch
git log --oneline --decorate -12
```

Also read the source references and project documents named by the DAG only as needed for current decisions. Trust Git, project artifacts, and the DAG over chat recollection.

If a node is still `running`, reconcile it before new dispatch:

- read the open attempt's recorded worktree, branch, base/head, review rounds, structured handoff, and diff;
- if completed work is recoverable, finish its review and master gate;
- if the attempt cannot be proven complete, mark it `failed` with a stale-attempt reason, then decide whether to add context, change model, re-slice, or retry;
- never replay work merely because the old session disappeared.

Do not start nodes while `planning_status` is not `approved`. If any command reports that the control file was modified outside the runtime, stop and tell the user: something wrote state that no gate approved, and the correct response is to inspect it, not to re-seal past it.

## Select; do not mechanically drain

Obtain eligible candidates:

```bash
"${CLAUDE_SKILL_DIR}/scripts/ready-tasks" --json
```

The result means only: pending, all dependencies done, and no active conflict. Choose among it using actual wall-clock benefit, repeated exploration cost, hidden shared state, worktree availability, and integration risk.

- One worker is normal.
- Use two when independence is clear and concurrent work saves meaningful time.
- Use three only for highly independent, stable scopes with obvious benefit.
- Never exceed the DAG's `max_parallel` or the hard ceiling of three.
- Open-ended investigations normally run one at a time until boundaries become known.

## Dispatch a node

Mark a node running before dispatch:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" <node-id> running \
  --worktree "<absolute or project-relative worktree>" \
  --branch "<expected branch>" --base-ref "<expected HEAD>"
```

Use a fresh worker with no inherited chat history. Give it only:

- the exact node contract;
- precise upstream handoff references and outputs;
- necessary project constraints and paths;
- its worktree/path, and an absolute report artifact path under the control plane reported by `status`;
- the worker protocol from `${CLAUDE_SKILL_DIR}/references/node-contract.md`, including its search boundary.

State that boundary explicitly in the prompt rather than assuming it: `read_first` is the whole context, searching is limited to `scope.files` and the paths those files name, and a thin contract is reported as `NEEDS_CONTEXT` naming the missing file instead of being filled in by exploration. `scope.files` stops a worker from writing too widely; only this sentence stops it from reading too widely.

Do not send the whole DAG, full PRD, prior worker reasoning, or full session history. The worker implements and reports; it does not define acceptance, approve the node, edit the DAG, integrate itself into the control branch, or dispatch its own reviewer.

## Worktree policy

For one node or strict serial work, use the current clean working tree unless existing project rules require isolation. Do not create a worktree for ceremony.

`.dag/` exists only at the main worktree root. A relative `.dag/artifacts/...` path handed to a worker would resolve inside that worker's own worktree, where the runtime will not find it, so give absolute artifact paths whenever the worker is not in the main worktree.

For concurrent implementation nodes, each node gets an isolated worktree and branch from the correct accepted base. Verify the location is ignored when project-local, reproduce required environment setup, run a clean baseline check, and keep scope explicit. Workers commit only their node. They do not merge themselves into the control branch.

## Review and repair

Check scope first, before spending a reviewer on the diff:

```bash
"${CLAUDE_SKILL_DIR}/scripts/check-scope" <node-id> --head-ref "<reported-sha>" \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json"
```

If it reports a violation, do not dispatch a reviewer. Return the paths to the worker or fail the attempt. The review gate would reject the same diff later anyway, after a full reviewer run has already been paid for.

Now read `${CLAUDE_SKILL_DIR}/references/review-protocol.md` and create a bounded review package: original node contract, actual diff or immutable diff path, commit range, structured handoff, relevant evidence, and any paths `check-scope` reported as also claimed by an unfinished node. Dispatch a fresh read-only reviewer using that protocol. Do not include the implementer's reasoning transcript.

The reviewer checks both acceptance/scope compliance and code quality. It cannot invent acceptance criteria or choose a new architecture. Every finding it raises carries the path it is about, and that path decides whether it is a `blocking_finding` the worker can fix or a `controller_decision` outside the node's scope. Save its JSON verdict using the schema in `${CLAUDE_SKILL_DIR}/references/review-protocol.md`, then record the round:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" <node-id> review \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json" \
  --review-ref ".dag/artifacts/<node-id>/review-<round>.json" \
  --head-ref "<reviewed-sha>"
```

The runtime derives `approved`, `needs_fixes`, `cannot_verify`, or `escalated` from the artifact and binds it to the reviewed head. Deterministic defects go back to the original implementer, followed by a scoped fresh re-review.

`escalated` means the reviewer found a blocking problem this diff caused outside `scope.files`, typically a caller the worker was not allowed to touch. Do not send it back as a fix round: the worker would have to leave its scope, and the scope gate will refuse the result. Give those paths an owner instead — `--add-gap-nodes`, or a replan if the contract itself was wrong — then re-review the same head, which passes once the problem belongs to someone.

Run at most three review/fix rounds. `update-task` rejects a fourth round. After round three with open Critical/Important or spec findings, stop that loop and choose explicitly: add context, use a stronger model, re-slice, re-plan, or mark blocked. Never waive a load-bearing finding merely because the loop reached its cap.

## Master verification gate

Apply `${CLAUDE_SKILL_DIR}/references/verification.md`. The controller personally verifies scope from Git, inspects reproducible evidence, confirms the reviewer used the actual contract and diff, and runs at least one real criterion tied directly to this node's acceptance.

Only after all four checks pass may the controller record completion:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" <node-id> done \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json" \
  --verification-ref ".dag/artifacts/<node-id>/master-verification.json" \
  --head-ref "<reviewed-sha>" --summary "<short result>"
```

The runtime refuses completion unless the worktree is clean outside `.dag/`, the base-to-head diff stays inside node scope, referenced artifacts exist and parse, the final review is approved for that head, and the master artifact records a passing check against declared acceptance.

For a failed or blocked attempt, record a short reason and durable evidence reference. Never put a transcript in `dag.json`.

## Integrate accepted nodes

Land parallel work only after each node passes its own gate, in dependency and integration order. Preserve roughly one logical commit per node. The controller may resolve purely mechanical conflicts; semantic conflicts go back to the responsible worker or cause replanning. Run integration checks after landing related nodes.

## Final convergence

All nodes being `done` is necessary but not sufficient. Re-read the original objective, source references, global acceptance, and integrated behavior. Perform a fresh whole-DAG convergence review for omitted requirements, inconsistent interfaces, and integration gaps.

If a gap exists, record failed convergence and its durable report:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --convergence failed \
  --reference ".dag/artifacts/convergence-review.md" --gap "<short gap>"
```

Then write a compact gap-plan JSON artifact as specified in `${CLAUDE_SKILL_DIR}/references/dag-schema.md` and import it through:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" \
  --add-gap-nodes ".dag/artifacts/gap-plan.json"
```

This preserves the failed convergence record, adds the artifact as a source, and returns planning to `awaiting_approval`. Do not edit nodes directly or hide the gap inside a completed node.

If convergence passes, record it and complete the DAG:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --convergence passed \
  --reference ".dag/artifacts/convergence-review.md"
"${CLAUDE_SKILL_DIR}/scripts/update-task" --planning-status complete \
  --reference ".dag/artifacts/convergence-review.md"
```

Announce project completion only when every node is done, convergence passed, and `planning_status` is `complete`.

Then retire the control plane so the next effort starts clean:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --archive
```

This moves the DAG and its recorded evidence into `.dag/archive/<dag-id>-<timestamp>.json` and removes `.dag/dag.json`. Archiving is refused while any node is running.
