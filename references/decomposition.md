# Decomposition

## Start with observable outcomes

Trace backward from the original objective and global acceptance. Identify the artifacts, interfaces, behaviors, migrations, and integration checks that must exist, then group work into nodes that can be accepted separately without losing the end-to-end path.

Do not decompose while an unresolved choice could change the objective, scope, acceptance, interface ownership, dependency edges, or conflict edges. Return to planning clarification instead of hiding the choice in a worker contract.

## Node sizing

The following rule is adapted from Superpowers `writing-plans` with `task` changed to `node`:

> A node is the smallest unit that carries its own test cycle and is worth a
> fresh reviewer's gate. When drawing node boundaries: fold setup,
> configuration, scaffolding, and documentation steps into the node whose
> deliverable needs them; split only where a reviewer could meaningfully reject
> one node while approving its neighbor. Each node ends with an independently
> testable deliverable.

For this DAG, a good node also has:

- one clear objective and hard scope;
- explicit inputs and outputs;
- enough context for a fresh worker;
- acceptance based on observable behavior or artifacts;
- normally one logical commit;
- no unresolved architecture choice hidden inside implementation.

Combine mechanically identical, same-shaped edits on one review surface. Split outcomes that could receive different verdicts, interfaces that must be accepted before consumers start, or failures that should not invalidate neighboring work. Do not create bookkeeping nodes for one import, symbol rename, test run, or commit, or subsystem-sized nodes that require a new architecture while being implemented.

## Dependency pass

Ask for each pair:

1. Does one consume an interface, decision, schema, artifact, or verified behavior produced by the other?
2. Would starting the consumer first make it guess?
3. Would upstream failure make downstream work invalid rather than merely harder to merge?

If yes, add `depends_on` before considering file overlap. Different files do not remove a semantic dependency.

## Conflict pass

Ask separately:

1. Could both nodes be designed correctly at the same time?
2. Would concurrent edits contend on a file, generated artifact, migration sequence, lockfile, shared worktree state, or external mutable resource?
3. Can isolation remove the contention without creating a later semantic choice?

Use symmetric `conflicts_with` when concurrent activity is unsafe but neither node consumes the other. Same-file work usually conflicts; different files can still depend semantically.

## Independently executable contracts

合同应直接携带可执行事实：来源已经确定的值、条件、边界和可观察结果，写入现有 `objective`、`inputs`、`outputs`、`acceptance` 或 `execution_plan`。只有 worker 必须重新打开来源才能判断的内容，才进入 `read_first`；每个条目都要对应一个具名主张。`scope.files` 是写边界，`scope.forbidden` 是写保护区，不是阅读清单。

Use exact project-relative paths, interface names, durable upstream handoff references, and real verification commands. A technology or interface choice that sources do not establish belongs in `assumptions` or investigation output, never in a node contract as settled fact.

Avoid:

- “implement as appropriate,” “handle edge cases,” or “tests above”;
- references to current chat or implicit prior reasoning;
- acceptance that only restates implementation activity;
- behavioral acceptance verified only by syntax, import, file existence, or keyword search;
- a node whose first action must be broad architecture discovery;
- fake dependencies added only to force a preferred schedule.

When uncertainty is itself work, create a bounded `investigation` node with a decision artifact, explicit questions, a read budget or stop condition, and downstream nodes dependent on its accepted output.

Every node carries a non-empty string-array `execution_plan` for a fresh implementer, normally a few ordered steps. Each string uses `project-relative landing (to a symbol or section when known)—concrete action; covers AC*/V*`. The plan is an execution map, not a second contract: do not restate `objective`, `scope`, `inputs`, `outputs`, or `acceptance`, and do not copy verification commands. When the landing is unknown, use a bounded `investigation` node with a decision artifact rather than a guessed location.

```json
"execution_plan": [
  "src/auth/client.py:AuthClient.login—Reject invalid credentials before session creation; covers AC1/V1"
]
```

## Decomposition review

Use one independent reviewer and one review package covering the whole plan in one pass: requirements (objective, global acceptance, source references, assumptions, coverage), graph (dependency, conflict, and resource shape), and execution (scope, context, acceptance, and verification executability).

The controller writes one `review-request.json` containing the exact plan fingerprint and every node contract without execution records. The reviewer is dispatched with file-editing capability and works fix-first: it copies the plan from the request into a draft, repairs every defect it can fix directly in the draft, and records each as a bug. The reviewer returns one flat `review.json`:

```json
{
  "dag_id": "auth-migration",
  "plan_fingerprint": "<current plan fingerprint>",
  "checks_performed": ["coverage pass", "dependency pass", "conflict pass", "node sizing", "scope pass", "context pass"],
  "draft_ref": ".dag/draft.json",
  "bugs": [
    {
      "id": "graph-001",
      "kind": "missing_dependency",
      "nodes": ["auth-client"],
      "detail": "auth-client consumes the token shape that define-auth-contract produces, but declares no dependency",
      "fix": "add auth-client.depends_on = [define-auth-contract] in the draft"
    }
  ],
  "unsure": []
}
```

There are no `plan.json`, per-lane result files, manifests, or review modes. The reviewer is an independent source of findings, not a decision-maker above the user. Full review and disclosure remain mandatory. A user who has seen an open `unsure` item may make a clear, durable decision to approve or continue, recorded through `approval_ref`; that decision cannot bypass runtime-enforced schema, acyclicity, paths, scope, Git/SHA, evidence-binding, or authorization-boundary checks. The runtime, not the reviewer, derives the verdict: any `unsure` entry means `needs_changes`, otherwise `approved`.

The validator already proves graph structure: no cycles, symmetric conflicts, undeclared same-file collisions between concurrent nodes, and acceptance coverage by verification. Review what scripts cannot decide:

- **missing dependency**: a node needs another node's output, interface, decision, or verified behavior but has no edge;
- **fabricated dependency**: an edge encodes scheduling preference rather than consumption;
- **granularity**: the node needs broad architecture discovery first or has no independent review value;
- **acceptance**: criteria restate implementation activity or behavioral criteria have only structural verification;
- **coverage**: a source requirement, global acceptance criterion, or integration step has no delivering node;
- **context**: `read_first` omits a file required by acceptance or names a directory instead of files;
- **scope realism**: `scope.files` does not match where behavior lives;
- **assumption laundering**: a decision was parked in `assumptions` instead of clarified.

Every finding names the offending nodes and the failure, not a style preference. Bugs carry a `fix` describing what the reviewer's draft changes; unsure entries carry a `question` stating the decision only the user or controller may make. The single `review.json` binds its entries to the request fingerprint and names the reviewer's draft in `draft_ref` when it reports bugs.

The runtime derives the verdict; a reviewer does not pass its own approval flag. Any open `unsure` entry blocks approval until the controller settles it with a further `--revise` or the user records a durable override; bugs are already fixed in the draft and never block on their own. A material `--revise` at any later stage clears the review, and the revised plan is reviewed and approved as a whole. A narrow revision of execution maps, `read_first`, verification commands, or tighter `scope` on an approved plan keeps that approval.

## Coverage convergence preview

Before approval, map every source requirement and global acceptance criterion to at least one node and verification path. Check cross-node integration explicitly; a populated task list is not evidence of coverage.
