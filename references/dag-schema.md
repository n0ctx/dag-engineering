# DAG schema

`<project>/.dag/dag.json` is controller-owned execution state. Use UTF-8 JSON schema version `2`; schema 1 has no compatibility or migration path. Large reports live under `.dag/artifacts/` and the DAG stores only project-relative references.

## Top level

The required root fields are `schema_version`, `dag_id`, `objective`, `source_refs`, `assumptions`, `global_acceptance`, `planning_status`, `approval_ref`, `planning_session`, `decomposition_review`, `max_parallel`, `created_at`, `updated_at`, `convergence`, and `nodes`.

`planning_status` is `draft`, `awaiting_approval`, `approved`, `replan_required`, or `complete`. `convergence.status` is `pending`, `failed`, or `passed`; a failed result retains its `gaps` until explicit gap nodes and a later convergence review close them. `approval_ref` is the durable user approval or decision reference and may be null before approval. Timestamps are UTC RFC 3339 values ending in `Z`.

## Execution tiering

New DAGs must include `execution_tiering`; legacy DAGs without it remain readable. Its `version` is `1`, and `nodes` maps every node ID exactly once to an object with `executor_class` and `estimated_active_minutes`. `micro` is 2–5 minutes and may cover at most 3 tracked scope files; `tier2` is 5–15 minutes and may cover at most 8. `senior` and `controller` are 1–60 minutes and are reserved for explicit design decisions and graph-wide convergence. Minutes mean active agent time, not test wall time. The runtime validates profiles, includes them in review fingerprints, and refuses to create a new DAG without them.

## Node contract

Each node contains `id`, `title`, `objective`, `execution_plan`, `work_type`, `scope`, `read_first`, `depends_on`, `conflicts_with`, `inputs`, `outputs`, `acceptance`, `verification`, `status`, `attempts`, and `handoff`.

`execution_plan` is a required non-empty string array; every step is non-empty and the validator limits it to six steps. It appears in the node brief, plan view, review request, and plan fingerprint. It describes execution within one node and is not a cross-node interface, so changing only this field does not trigger downstream interface review.

`work_type` is `implementation`, `investigation`, `integration`, or `documentation`. Node status is `pending`, `running`, `done`, `failed`, or `blocked`. Contracts for `pending`, `failed`, and `blocked` nodes may be revised. `running` and `done` contracts are frozen. Dependencies reference existing nodes, conflicts are symmetric, and the graph must be acyclic. Parallel scopes may not overlap unless ordered or explicitly conflicted.

Paths in `scope.files`, `scope.forbidden`, and `read_first` are safe project-relative paths or globs. `scope.files`, `acceptance`, `verification`, and `outputs` are non-empty. Every verification covers an acceptance criterion. Read context and scope limits are enforced before dispatch.

## Attempts, handoff, and evidence

An attempt records `number`, `started_at`, `finished_at`, `outcome`, `worktree`, `branch`, `base_ref`, `head_ref`, `worker_head`, `fix_commit`, `reviews`, `handoff_ref`, `verification_ref`, and `failure_reason`. It closes with outcome `done`, `failed`, or `blocked`.

Handoff artifacts are JSON objects with the node, a worker completion status, commit, changed files, verification, and context used. A done node requires a final approved review (or a narrow recorded resolution), a master verification artifact, matching Git head, and complete evidence. Artifact paths must exist, be non-empty, and remain within the project. Scope files must equal the sorted Git diff paths; out-of-scope changes fail closed.

## Plan review artifacts

Normal plan review uses exactly:

```text
.dag/artifacts/<review>/review-request.json
.dag/artifacts/<review>/review.json
```

The runtime request contains `dag_id`, mode, current `plan_fingerprint`, required lanes, affected nodes, and the plan contracts. A full request always requires `requirements`, `graph`, and `execution`. A repair request uses only the runtime-computed `required_lanes` and affected nodes.

`review.json` contains `dag_id`, `plan_fingerprint`, `mode`, and a `lanes` object. Each required lane occurs exactly once and has `checks_performed` (a non-empty string array), `findings` (an array), and, for repair, `resolutions`. Finding lane ownership comes from the object key, not reviewer input. Finding IDs, severity (`critical`, `important`, `minor`), kind, details, and node references are validated. Repair must resolve every prior `open_findings` entry exactly once; `NOT_ADDRESSED` findings remain open.

The fingerprint covers objective, global acceptance, assumptions, source references, and all node plan fields including `execution_plan`; execution state is excluded. Fingerprint mismatch, missing lanes, malformed findings, invalid graph/path/scope checks, or incomplete evidence fail closed.

Independent review and all findings remain durable. A user’s persisted `approval_ref` may explicitly override subjective critical or important findings during planning approval or scoped revision. Planning approval also stores the override as `decomposition_review.decision_ref`; scoped revision stores it in the revision entry. It cannot override deterministic validation or preflight gates, and approval is not a way to skip review.

## Control-plane lifecycle

`.dag/` exists only at the main worktree root. The runtime seals `dag.json` beside it and obtains an exclusive lock for updates. `update-task --init` is the only creation path. `--amend` replaces an unapproved plan and clears its review; `--revise` reorganizes an approved plan while preserving execution records and requiring scoped review when interfaces or dependent contracts change. Running nodes cannot be amended, and running nodes cannot be retired. Archived DAGs are records, not control files.

`status --node <id>` emits a dispatch brief, `status --plan-view` emits every node contract without execution records, and `status --review-request` emits the current review request. No worker reads or writes the control file directly.

## Non-obvious invariants

- `source_refs` identify durable source material: project paths, stable URLs, issue IDs, or commit references. Chat-only input is first persisted as a source artifact. `assumptions` are explicit proposed choices, not facts; approval accepts them and unresolved choices belong in an investigation/decision node.
- The runtime seals `dag.json` in `.dag/.dag.json.seal` after every write and verifies it before every read. `--reseal --reason` is the deliberate repair path and records the reason in `reseal`.
- `planning_session` is stamped on first approval and preserved through amendments/revisions. A planning session cannot execute its own DAG.
- Amend/revise preserve attempts, handoffs, and statuses for work already run. Running contracts cannot change; done contracts cannot change; revise requires scoped review when changed interfaces affect dependants.
- Node review outcomes bind the reviewed handoff and immutable Git head. Reviewer fix commits (`fix_commit`) must be independent descendants of `worker_head`; a done node also requires a matching master verification artifact and scope/evidence checks.
- Failed convergence gaps are imported only through a reviewed gap-node artifact whose `dag_id` and `addresses_gaps` exactly match the recorded gaps. Import appends pending nodes, records the artifact in `source_refs`, clears approval, and returns planning to `awaiting_approval`.
