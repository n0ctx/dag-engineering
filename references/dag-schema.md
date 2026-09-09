# DAG schema

`<project>/.dag/dag.json` is controller-owned execution state. Use UTF-8 JSON and schema version `1`. Keep large reports under `.dag/artifacts/` and store only short references in the DAG.

Nothing reads the control file directly except the runtime. `status --node <id>` emits one node's dispatch brief and `status --plan-view` emits every node's contract without the execution record; those are what a worker and a plan reviewer receive.

## Top level

```json
{
  "schema_version": 1,
  "dag_id": "auth-migration",
  "objective": "Observable project-level outcome",
  "source_refs": ["docs/prd.md", "issue:123"],
  "assumptions": ["Proposed choice not established by the source; approval accepts it"],
  "global_acceptance": [
    {"id": "G1", "criterion": "Observable end-to-end behavior"}
  ],
  "planning_status": "awaiting_approval",
  "approval_ref": null,
  "planning_session": null,
  "decomposition_review": null,
  "max_parallel": 2,
  "created_at": "2026-09-08T12:00:00Z",
  "updated_at": "2026-09-08T12:00:00Z",
  "convergence": {
    "status": "pending",
    "verification_ref": null,
    "checked_at": null,
    "gaps": []
  },
  "nodes": []
}
```

Required top-level fields are shown above. `approval_ref` may be null until approved. Timestamps use UTC RFC 3339 with a trailing `Z`.

Every `source_refs` entry must identify durable state: a project path, stable URL/issue reference, or commit. When the only source is chat, planning first writes `.dag/sources/<dag-id>.md`; a date label is not a source reference.

`assumptions` is an array of concise proposed choices that are not facts in the sources but are needed to make node contracts executable. It may be empty. Every non-empty entry appears in the approval summary; approving the DAG accepts those choices. Unresolved choices that should not be accepted yet belong in a bounded investigation/decision node instead.

`planning_status` is one of:

- `draft`: planning is not ready for validation or review;
- `awaiting_approval`: valid proposal waiting for user approval;
- `approved`: execution is authorized;
- `replan_required`: the accepted graph no longer covers reality;
- `complete`: all nodes are done and convergence passed.

`convergence.status` is `pending`, `failed`, or `passed`. It is not inferred from an empty ready set. A failed result and its gap list remain in the DAG while explicit gap nodes run; a later full review replaces it with `passed`.

## Node

```json
{
  "id": "implement-auth-consumer",
  "title": "Implement the authentication consumer",
  "objective": "One independently reviewable outcome",
  "work_type": "implementation",
  "scope": {
    "files": ["src/auth/**", "tests/auth/**"],
    "forbidden": ["src/billing/**"]
  },
  "read_first": ["docs/auth-contract.md", "src/auth/client.py"],
  "depends_on": ["define-auth-contract"],
  "conflicts_with": [],
  "inputs": ["Auth contract exported by define-auth-contract"],
  "outputs": ["Consumer implementation", "behavioral regression tests"],
  "acceptance": [
    {"id": "AC1", "criterion": "Invalid credentials are rejected without creating a session"}
  ],
  "verification": [
    {
      "id": "V1",
      "run": "pytest tests/auth/test_client.py -q",
      "expect": "exit 0 and the invalid-credentials case passes",
      "covers": ["AC1"]
    }
  ],
  "status": "pending",
  "attempts": [],
  "handoff": {
    "status": null,
    "ref": null,
    "verification_refs": [],
    "commit": null,
    "summary": null,
    "failure_reason": null,
    "updated_at": null
  }
}
```

`work_type` is `implementation`, `investigation`, `integration`, or `documentation`. It informs judgment but never makes dispatch automatic.

`status` is `pending`, `running`, `done`, `failed`, or `blocked`. `ready` is never stored.

## Control plane location and lifecycle

- `.dag/` lives at the main worktree root and nowhere else. Commands resolve it from there, so they work from any subdirectory, and reject a `.dag/dag.json` found elsewhere. Outside Git, or under `--separate-git-dir` where Git cannot report a trustworthy worktree root, resolution falls back to the current directory.
- `.dag/dag.json` is sealed. Its checksum lives beside it in `.dag/.dag.json.seal`, written on every runtime write and verified on every read, so a control file edited with a file tool fails closed instead of carrying state no gate approved. A deliberate repair, such as resolving a merge conflict, is re-sealed with `update-task --reseal --reason "<why>"`, which records the repair in `reseal` for the audit trail. Archived copies are records, not control files, and carry no seal.
- An unapproved plan is corrected with `update-task --amend <draft>`, which keeps the same `dag_id`, clears `decomposition_review`, and refuses once the plan is approved or any node has attempts.
- `update-task --revise` also accepts a patch in place of a whole plan: `{"dag_id": "<same>", "update_nodes": [...], "drop_nodes": [...]}`. Fields absent from an `update_nodes` entry keep their current value, an entry whose `id` is unknown adds a node, and the merged plan is validated in full, so a revision costs what the change costs rather than what the plan weighs. `--amend` takes a whole plan only, since the user approves what it produces.
- An approved plan is reorganised with `update-task --revise <draft> --reason "<why>"`, which keeps it `approved` so execution is not interrupted. It may re-slice, add, drop, and re-wire unstarted work, and correct a `failed` or `blocked` node's contract; the contract of a `done` or `running` node is frozen, and every node that ran keeps its `status`, `attempts`, and `handoff`. It refuses any change to `objective`, `global_acceptance`, `assumptions`, or `source_refs`, because those are what the project promises. Dropping a node or changing one's `outputs` or `depends_on` moves an interface downstream nodes were written against, so the runtime names the affected dependants and requires a scoped review passed with `--revision-review`.
- A plan that is not `approved` is replaced with `update-task --amend <draft>`, which is how review findings get fixed. It preserves what already ran: a node with attempts keeps its `status`, `attempts`, and `handoff` and cannot be dropped, and a `done` node keeps its contract too, because its diff was accepted against it. Everything unstarted may be added, dropped, and re-sliced. Amending clears `decomposition_review`, leaves the plan unapproved, and is refused while a node is running. Each amendment appends to `replans` and the draft it names is kept, since that file is the record of what was proposed, so give each amendment its own path. An approved plan reaches this path through `update-task --planning-status replan_required --reference "<why>"`. Abandoning is for dropping an effort, not for editing one.
- One `.dag/dag.json` exists at a time. `update-task --init <draft>` is the only sanctioned way to create it and refuses to replace an existing one.
- `update-task --archive` retires a `complete` DAG; `update-task --abandon --reason "<why>" --reference "<durable path>"` retires an unfinished one, recording both the reason and the user's persisted instruction, because dropping unfinished work is never the controller's own decision. Both write `.dag/archive/<dag-id>-<timestamp>.json`, add `archived_at`, remove `.dag/dag.json`, and refuse while a node is running. Archived files are records only and are never reloaded.

## Path and contract rules

- `scope.files`, `scope.forbidden`, and `read_first` contain project-relative paths or glob patterns. Absolute paths and `..` traversal are invalid.
- `read_first` is the node's complete declared context, not a reading suggestion, and `scope.files` bounds only what a worker may write. Approval refuses a node declaring more than 120,000 characters of `read_first`, or a `scope.files` expanding past 200 files or 20,000 lines of tracked content, because nothing at execution time can cap what a worker reads. `validate-dag` reports the same measurements as warnings, and `ready-tasks` prints them per candidate. Starting a node is refused while any `read_first` path is missing from the worktree: by then its dependencies are done, so an absent input sends the worker looking for its own.
- A review outcome is `approved`, `needs_fixes`, `cannot_verify`, or `escalated`. A review binds `worker_head` to the immutable worker delivery and may bind `reviewer_fix_head` to one independent reviewer repair commit. The controller accepts the final head only after checking the fix-only range and an independent acceptance check; the reviewer never accepts its own fix. `escalated` means a blocking problem is outside `scope.files`, unsafe to repair, or contract-bound; same-SHA escalation/contract resolution never re-reviews unchanged code. Every review finding carries a stable ID and the runtime rejects one filed on the wrong side of the scope boundary.
- `scope.files`, `acceptance`, `verification`, and `outputs` are non-empty.
- Node IDs and acceptance/verification IDs are unique within their owner.
- Every verification entry covers at least one acceptance ID, and every acceptance ID has coverage.
- `depends_on` and `conflicts_with` reference existing nodes and cannot reference self.
- `conflicts_with` is symmetric. The validator rejects one-sided conflicts.
- `decomposition_review` records one complete manifest of the three narrow lanes (or the affected subset for repair), fingerprinted over objective, global acceptance, assumptions, source refs, and each node contract. Each requested lane occurs exactly once; execution state is excluded. `planning_status` cannot become `approved` without a matching approved manifest. Repair approval does not trigger another full review; only final convergence is whole-plan.
- `planning_session` is stamped by the runtime at the first approval and is never rewritten, including after a replan. Node execution is refused from that same session, so planning context cannot leak into execution. It stays null when no session identity is available, and the check then passes.
- Two nodes that can run concurrently, meaning neither depends on the other, cannot have overlapping `scope.files` without a `conflicts_with` edge. The validator rejects the undeclared collision.
- A done or running node cannot depend on a non-done node.
- A done node has an existing structured handoff, a final approved review, a master verification artifact, and a Git head that all identify the same node revision.
- A complete DAG has all nodes done and passed convergence with a verification reference.

## Attempts

The runtime appends an attempt when a node enters `running`:

```json
{
  "number": 1,
  "started_at": "2026-09-08T12:10:00Z",
  "finished_at": null,
  "outcome": null,
  "worktree": "/absolute/path/to/worktree",
  "branch": "dag/implement-auth-consumer",
  "base_ref": "full-commit-sha",
  "head_ref": null,
  "worker_head": null,
  "reviewer_fix_head": null,
  "reviews": [],
  "handoff_ref": null,
  "verification_ref": null,
  "failure_reason": null
}
```

It closes that attempt on `done`, `failed`, or `blocked`. Each `reviews` entry records `round`, derived `outcome`, artifact `ref`, reviewed `handoff_ref`, immutable final `head_ref`, `worker_head`, optional `reviewer_fix_head`, and `reviewed_at`. Active attempts also expose `open_escalations`/`escalation_resolution` and `open_contract_findings`/`contract_resolution`; older DAGs may omit these and remain readable. A same-head escalation or contract decision is resolved without re-review. Do not store worker transcripts here.

`update-task` obtains an exclusive sibling lock before reading and replacing `dag.json`, so two controller updates cannot silently overwrite each other. The lock file is runtime state, not part of the DAG schema.

## Durable evidence

Artifact references used for review, completion, and convergence are project-relative paths to existing non-empty files. Review and completion artifacts are JSON objects. The runtime checks their node ID and Git head rather than accepting a non-empty string as proof.

The master verification object contains at least:

```json
{
  "node": "implement-auth-consumer",
  "head_ref": "full-commit-sha",
  "checked_at": "2026-09-08T12:30:00Z",
  "acceptance_ids": ["AC1"],
  "checks": [{"command": "pytest tests/auth/test_client.py -q", "exit_code": 0, "result": "1 passed"}],
  "scope_files": ["src/auth/client.py", "tests/auth/test_client.py"]
}
```

`scope_files` must exactly equal the sorted Git diff path list produced by the runtime. Uncommitted changes outside `.dag/`, a path outside `scope.files`, or a path matching `scope.forbidden` prevents review and completion.

## Gap-node import

After failed convergence, planning writes a project-relative JSON artifact with the current `dag_id`, an `addresses_gaps` array equal to the recorded gaps, and one or more complete pending node objects. `update-task --add-gap-nodes <ref>` validates and appends them, adds the artifact to `source_refs`, clears the old approval reference, and moves planning to `awaiting_approval`. Direct edits to add gap nodes bypass the supported control-plane operation.
