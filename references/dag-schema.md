# DAG schema

`<project>/.dag/dag.json` is controller-owned execution state. Use UTF-8 JSON schema version `2`; schema 1 has no compatibility or migration path. Large reports live under `.dag/artifacts/` and the DAG stores only project-relative references.

## Top level

The required root fields are `schema_version`, `dag_id`, `objective`, `source_refs`, `assumptions`, `global_acceptance`, `planning_status`, `approval_ref`, `planning_session`, `decomposition_review`, `max_parallel`, `created_at`, `updated_at`, `convergence`, and `nodes`.

`planning_status` is `draft`, `awaiting_approval`, `approved`, `replan_required`, or `complete`. `convergence.status` is `pending`, `failed`, or `passed`; a failed result retains its `gaps` until explicit gap nodes and a later convergence review close them. `approval_ref` is the durable user approval or decision reference and may be null before approval. Timestamps are UTC RFC 3339 values ending in `Z`.

## Node contract

Each node contains `id`, `title`, `objective`, `execution_plan`, `work_type`, `scope`, `read_first`, `depends_on`, `conflicts_with`, `inputs`, `outputs`, `acceptance`, `verification`, `status`, `attempts`, and `handoff`.

`execution_plan` is a required non-empty string array; every step is non-empty. It appears in the node brief, plan view, review request, and plan fingerprint. It describes execution within one node and is not a cross-node interface, so changing only this field does not trigger downstream interface review.

`work_type` is `implementation`, `investigation`, `integration`, or `documentation`. Node status is `pending`, `running`, `done`, `failed`, or `blocked`. Contracts for `pending`, `failed`, and `blocked` nodes may be revised. `running` and `done` contracts are frozen. Dependencies reference existing nodes, conflicts are symmetric, and the graph must be acyclic. Parallel scopes may not overlap unless ordered or explicitly conflicted.

Paths in `scope.files`, `scope.forbidden`, and `read_first` are safe project-relative paths or globs. `scope.files`, `acceptance`, `verification`, and `outputs` are non-empty. Every verification covers an acceptance criterion. Read context and scope size are measured and reported before dispatch.

## Attempts, handoff, and evidence

An attempt records `number`, `started_at`, `finished_at`, `outcome`, `worktree`, `branch`, `base_ref`, `head_ref`, `worker_head`, `fix_commit`, `reviews`, `handoff_ref`, `verification_ref`, and `failure_reason`. It closes with outcome `done`, `failed`, or `blocked`.

Handoff artifacts are JSON objects with the node, a worker completion status, commit, changed files, verification, and context used. A done node requires a final approved review (or a narrow recorded resolution), a master verification artifact, matching Git head, and complete evidence. Artifact paths must exist, be non-empty, and remain within the project. Scope files must equal the sorted Git diff paths; out-of-scope changes fail closed.

## Plan review artifacts

Normal plan review uses exactly:

```text
.dag/artifacts/<review>/review-request.json
.dag/artifacts/<review>/review.json
```

The runtime request contains `dag_id`, the current `plan_fingerprint`, the plan fields (`objective`, `source_refs`, `assumptions`, `global_acceptance`, `max_parallel`), and every node contract without execution records. One request shape serves new plans and revised plans alike.

`review.json` is flat: `dag_id`, `plan_fingerprint`, `checks_performed` (a non-empty string array), `bugs`, `unsure`, and `draft_ref`. A bug entry names an ID, kind, detail, node references, and `fix` (what the reviewer draft changes); an unsure entry names an ID, kind, detail, node references, and `question` (the decision only the user or controller may make). Finding IDs must be unique and node references must name existing nodes. Bugs and the draft are two halves of one claim: a review with bugs must name its draft plan in `draft_ref` and be recorded with `--draft`, and a draft without bugs is refused. Recording applies the draft under the revision protections — the execution record, attempted and done nodes, and the deliverable fields cannot change — and no further review round follows the reviewer's own fixes. The verdict is `needs_changes` while any `unsure` entry remains, otherwise `approved`.

The fingerprint covers objective, global acceptance, assumptions, source references, and all node plan fields including `execution_plan`; execution state is excluded. Fingerprint mismatch, malformed findings, invalid graph/path/scope checks, or incomplete evidence fail closed.

Independent review and all findings remain durable. A user’s persisted `approval_ref` may explicitly override open `unsure` items during planning approval; the override is stored as `decomposition_review.decision_ref`. It cannot override deterministic validation or preflight gates, and approval is not a way to skip review.

## Control-plane lifecycle

`.dag/` exists only at the main worktree root. The runtime seals `dag.json` beside it and obtains an exclusive lock for updates. `update-task --init` is the only creation path. `--revise` is the only plan-change path: it accepts a whole draft or a patch at any planning status except `complete`, preserves execution records, freezes the contracts of running and done nodes, and returns the plan to `awaiting_approval` with its review cleared for a fresh review round. Failed convergence is repaired only with `--add-gap-nodes`. Running nodes cannot be retired. Archived DAGs are records, not control files.

`status --node <id>` emits a dispatch brief, `status --plan-view` emits every node contract without execution records, and `status --review-request` emits the single-round review request. No worker reads or writes the control file directly.

## Non-obvious invariants

- `source_refs` identify durable source material: project paths, stable URLs, issue IDs, or commit references. Chat-only input is first persisted as a source artifact. `assumptions` are explicit proposed choices, not facts; approval accepts them and unresolved choices belong in an investigation/decision node.
- The runtime seals `dag.json` in `.dag/.dag.json.seal` after every write and verifies it before every read. `--reseal --reason` is the deliberate repair path and records the reason in `reseal`.
- `planning_session` is stamped on first approval and preserved through revisions. A planning session cannot execute its own DAG.
- Revisions preserve attempts, handoffs, and statuses for work already run. Running contracts cannot change; done contracts cannot change.
- Node review outcomes bind the reviewed handoff and immutable Git head. Reviewer fix commits (`fix_commit`) must be independent descendants of `worker_head`; a done node also requires a matching master verification artifact and scope/evidence checks.
- Failed convergence gaps are imported only through a reviewed gap-node artifact whose `dag_id` and `addresses_gaps` exactly match the recorded gaps. Import appends pending nodes, records the artifact in `source_refs`, clears approval, and returns planning to `awaiting_approval`.
