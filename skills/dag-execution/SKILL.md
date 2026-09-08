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

Read `${CLAUDE_SKILL_DIR}/references/dag-schema.md` only to diagnose or repair invalid state. Read `${CLAUDE_SKILL_DIR}/references/decomposition.md` only when re-slicing or replanning is required.

## Cold start and recovery

At the start of every session:

```bash
"${CLAUDE_SKILL_DIR}/scripts/validate-dag" .dag/dag.json
"${CLAUDE_SKILL_DIR}/scripts/status" .dag/dag.json
git status --short --branch
git log --oneline --decorate -12
```

Also read the source references and project documents named by the DAG only as needed for current decisions. Trust Git, project artifacts, and the DAG over chat recollection.

If a node is still `running`, reconcile it before new dispatch:

- read the open attempt's recorded worktree, branch, base/head, review rounds, structured handoff, and diff;
- if completed work is recoverable, finish its review and master gate;
- if the attempt cannot be proven complete, mark it `failed` with a stale-attempt reason, then decide whether to add context, change model, re-slice, or retry;
- never replay work merely because the old session disappeared.

Do not start nodes while `planning_status` is not `approved`.

## Select; do not mechanically drain

Obtain eligible candidates:

```bash
"${CLAUDE_SKILL_DIR}/scripts/ready-tasks" .dag/dag.json --json
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
"${CLAUDE_SKILL_DIR}/scripts/update-task" .dag/dag.json <node-id> running \
  --worktree "<absolute or project-relative worktree>" \
  --branch "<expected branch>" --base-ref "<expected HEAD>"
```

Use a fresh worker with no inherited chat history. Give it only:

- the exact node contract;
- precise upstream handoff references and outputs;
- necessary project constraints and paths;
- its worktree/path and report artifact path;
- the worker protocol from `${CLAUDE_SKILL_DIR}/references/node-contract.md`.

Do not send the whole DAG, full PRD, prior worker reasoning, or full session history. The worker implements and reports; it does not define acceptance, approve the node, edit the DAG, integrate itself into the control branch, or dispatch its own reviewer.

## Worktree policy

For one node or strict serial work, use the current clean working tree unless existing project rules require isolation. Do not create a worktree for ceremony.

For concurrent implementation nodes, each node gets an isolated worktree and branch from the correct accepted base. Verify the location is ignored when project-local, reproduce required environment setup, run a clean baseline check, and keep scope explicit. Workers commit only their node. They do not merge themselves into the control branch.

## Review and repair

After implementation, create a bounded review package: original node contract, actual diff or immutable diff path, commit range, structured handoff, and relevant evidence. Dispatch a fresh read-only reviewer using the reviewer protocol. Do not include the implementer's reasoning transcript.

The reviewer checks both acceptance/scope compliance and code quality. It cannot invent acceptance criteria or choose a new architecture. Save its JSON verdict using the schema in `${CLAUDE_SKILL_DIR}/references/node-contract.md`, then record the round:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" .dag/dag.json <node-id> review \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json" \
  --review-ref ".dag/artifacts/<node-id>/review-<round>.json" \
  --head-ref "<reviewed-sha>"
```

The runtime derives `approved`, `needs_fixes`, or `cannot_verify` from the artifact and binds it to the reviewed head. Deterministic defects go back to the original implementer, followed by a scoped fresh re-review.

Run at most three review/fix rounds. `update-task` rejects a fourth round. After round three with open Critical/Important or spec findings, stop that loop and choose explicitly: add context, use a stronger model, re-slice, re-plan, or mark blocked. Never waive a load-bearing finding merely because the loop reached its cap.

## Master verification gate

Apply `${CLAUDE_SKILL_DIR}/references/verification.md`. The controller personally verifies scope from Git, inspects reproducible evidence, confirms the reviewer used the actual contract and diff, and runs at least one real criterion tied directly to this node's acceptance.

Only after all four checks pass may the controller record completion:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" .dag/dag.json <node-id> done \
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
"${CLAUDE_SKILL_DIR}/scripts/update-task" .dag/dag.json --convergence failed \
  --reference ".dag/artifacts/convergence-review.md" --gap "<short gap>"
```

Then write a compact gap-plan JSON artifact as specified in `${CLAUDE_SKILL_DIR}/references/dag-schema.md` and import it through:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" .dag/dag.json \
  --add-gap-nodes ".dag/artifacts/gap-plan.json"
```

This preserves the failed convergence record, adds the artifact as a source, and returns planning to `awaiting_approval`. Do not edit nodes directly or hide the gap inside a completed node.

If convergence passes, record it and complete the DAG:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" .dag/dag.json --convergence passed \
  --reference ".dag/artifacts/convergence-review.md"
"${CLAUDE_SKILL_DIR}/scripts/update-task" .dag/dag.json --planning-status complete \
  --reference ".dag/artifacts/convergence-review.md"
```

Announce project completion only when every node is done, convergence passed, and `planning_status` is `complete`.
