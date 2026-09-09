# Node contract worker protocol

The controller packages one node for a fresh worker. The node contract is binding; a worker or reviewer reports against it and cannot amend it. Load `references/review-protocol.md` only when the controller reviews the result.

## Worker package

`status --node <id>` emits the node contract plus each upstream node's outputs, handoff reference, and commit. The controller builds the package; the worker must not open the control file. Add:

```text
worktree/path
absolute handoff artifact path
project constraints binding this node
```

Do not provide the whole DAG, chat history, unrelated upstream reports, or a future reviewer prompt. State the reading boundary in the same package:

- Read every file in `read_first` first; that is the planned context.
- `scope.files` is the write boundary, not a search budget.
- Beyond `read_first`, read only files those files name directly (such as an import or caller that must change).
- Do not open `.dag/dag.json`, survey the repository, read unrelated modules or history, or invent missing context.
- If the contract is insufficient, report `NEEDS_CONTEXT` naming the file, artifact, or decision; do not expand the search to compensate.

## Worker protocol

1. Read all `read_first` files before searching or editing.
2. Raise scope or contract problems before guessing; implement only the declared objective and scope.
3. Run focused verification, record reproducible evidence, and self-review the actual diff.
4. Commit the node when the assigned workflow requires commits.
5. Write the compact handoff artifact at the supplied path.

## You do not dispatch subagents

Do the node's work yourself. Never spawn a subagent, especially a reviewer: the controller owns the fresh review gate, and a worker-spawned reviewer duplicates cost and cannot approve the node.

Stop instead of expanding scope when a required file is outside `scope.files`, a forbidden file must change, reality contradicts the contract, an upstream output is absent, acceptance cannot be met, or a new architecture choice is required.

Use exactly one status:

- `DONE`: declared verification completed without known concern;
- `DONE_WITH_CONCERNS`: requested work completed but an integration or correctness doubt remains;
- `NEEDS_CONTEXT`: a specific missing fact, artifact, or decision is required;
- `BLOCKED`: the node cannot be completed under its contract or capability.

For `NEEDS_CONTEXT` or `BLOCKED`, report facts, the exact blocker, what was tried, and the required decision/context. Do not continue open-ended exploration. Even a stopped attempt writes a handoff with its status, empty `files_changed` when appropriate, and the blocker in `unresolved`; chat-only stopping leaves no durable state.

## Worker handoff

Write a compact artifact at the exact path supplied by the controller, normally `<control-plane-root>/.dag/artifacts/<node-id>/handoff.json`:

```json
{
  "node": "node-id",
  "status": "DONE",
  "files_changed": ["project/relative/path"],
  "outputs": ["artifact or interface produced"],
  "decisions": ["only durable decisions made within contract"],
  "verification": [
    {"command": "focused command", "result": "exit code and concise result", "artifact_ref": "path"}
  ],
  "context_used": ["every path read that read_first did not name"],
  "concerns": [],
  "unresolved": [],
  "downstream_notes": [],
  "commit": "full Git commit SHA"
}
```

Keep chronology, searches, failed commands, and reasoning out of the handoff; link durable reports instead. After the worker reports, the controller checks scope, then reviews the diff using `${SKILL_ROOT}/references/review-protocol.md`.
