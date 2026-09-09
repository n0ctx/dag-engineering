# Node contract

The controller packages one node for one fresh worker. The node contract is binding; the worker and reviewer report against it and cannot amend it. The worker must not open `.dag/dag.json`.

## Package

Run `status --node <id>` and forward its original node brief once. Do not reconstruct or paraphrase its fields. Add only:

```text
worktree/path
absolute handoff artifact path
binding project constraints
```

Do not send the whole DAG, chat history, unrelated upstream reports, worker reasoning, or a reviewer prompt. The package must state this reading boundary:

- Read every `read_first` file before searching or editing.
- `scope.files` is the write boundary, not a search budget.
- Beyond `read_first`, read only files directly named by those files or required caller changes.
- Do not survey the repository or invent missing context. If the contract is insufficient, report `NEEDS_CONTEXT` naming the missing file, artifact, or decision.

`execution_plan` is a non-empty string array supplied by the controller: each string is one concrete route step for a low-cost worker. An equivalent local implementation is allowed, but the worker must stop and report `NEEDS_CONTEXT` before changing `scope`, interfaces, outputs, architecture, or acceptance. Such a change requires controller-led revise/replan; it is never an implicit worker decision.

## Worker protocol

Use the fixed prefix supplied by the controller: read `read_first`, follow `execution_plan`, write only paths matched by `scope.files`, report conflicts as `NEEDS_CONTEXT`, run verification, commit, and write handoff. Do the work directly; do not dispatch subagents or a reviewer.

Stop when a forbidden file must change, reality contradicts the contract, an upstream output is absent, acceptance cannot be met, or a new architecture choice is required. Use exactly one status: `DONE`, `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, or `BLOCKED`. For the latter two, state the exact blocker, evidence, attempted bounded check, and required context or decision.

Every attempt writes the compact handoff at the supplied path, including stopped attempts:

```json
{
  "node": "node-id",
  "status": "DONE",
  "files_changed": ["project/relative/path"],
  "outputs": ["artifact or interface produced"],
  "decisions": ["durable decisions within the contract"],
  "verification": [{"command": "focused command", "result": "exit code and concise result", "artifact_ref": "path"}],
  "context_used": ["paths read beyond read_first"],
  "concerns": [],
  "unresolved": [],
  "downstream_notes": [],
  "commit": "full Git commit SHA"
}
```

Keep chronology, searches, failed commands, and reasoning out of the handoff; link durable evidence instead. After the worker reports, the controller runs the scope gate and sends the strict package to an independent reviewer.
