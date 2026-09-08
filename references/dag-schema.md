# DAG schema

`<project>/.dag/dag.json` is controller-owned execution state. Use UTF-8 JSON and schema version `1`. Keep large reports under `.dag/artifacts/` and store only short references in the DAG.

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
  "estimated_cost": "medium",
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

`work_type` is `implementation`, `investigation`, `integration`, or `documentation`. `estimated_cost` is `low`, `medium`, or `high`; it informs judgment but never makes dispatch automatic.

`status` is `pending`, `running`, `done`, `failed`, or `blocked`. `ready` is never stored.

## Path and contract rules

- `scope.files`, `scope.forbidden`, and `read_first` contain project-relative paths or glob patterns. Absolute paths and `..` traversal are invalid.
- `scope.files`, `acceptance`, `verification`, and `outputs` are non-empty.
- Node IDs and acceptance/verification IDs are unique within their owner.
- Every verification entry covers at least one acceptance ID, and every acceptance ID has coverage.
- `depends_on` and `conflicts_with` reference existing nodes and cannot reference self.
- `conflicts_with` is symmetric. The validator rejects one-sided conflicts.
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
  "reviews": [],
  "handoff_ref": null,
  "verification_ref": null,
  "failure_reason": null
}
```

It closes that attempt on `done`, `failed`, or `blocked`. Each `reviews` entry records `round`, derived `outcome`, artifact `ref`, reviewed `handoff_ref`, immutable `head_ref`, and `reviewed_at`. The runtime permits at most three entries per attempt. Do not store worker transcripts here.

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
