# Node contract

The node contract is binding; the worker and reviewer report against it and cannot rewrite it. The worker must not open `.dag/dag.json`.

## Package

合同是事实前置的：来源材料已经确定的结论必须直接写在现有合同字段中，使 worker 无需重新推导。`read_first` 只提供合同无法直接表达的最小证据入口；除它以外优先再读不超过 5 个文件，每个文件对应一个具名主张或验收问题。超出预算时写入 handoff `concerns`，不要因此停工。`scope.files` 只规定可写路径；`scope.forbidden` 是写保护清单，不要求主动阅读。

Run `status --node <id>` and forward its original node brief once. Do not reconstruct or paraphrase its fields. Add only:

```text
worktree/path
absolute handoff artifact path
binding project constraints
```

Do not send the whole DAG, chat history, unrelated upstream reports, worker reasoning, or a reviewer prompt. The package must state this reading boundary:

- Read every `read_first` file before searching or editing.
- `scope.files` is the write boundary, not a search budget.
- Beyond `read_first`, prefer files directly named by those files or required by a named claim. A longer read list is a warning in `concerns`, not a stop.
- Do not survey the repository or invent missing context. If the contract is insufficient, report `NEEDS_CONTEXT` naming the missing file, artifact, or decision.

`execution_plan` is a non-empty string array supplied by the controller: each string is one concrete route step for a low-cost worker. An equivalent local implementation is allowed, but the worker must stop and report `NEEDS_CONTEXT` before changing `scope`, interfaces, outputs, architecture, or acceptance. Such a change requires controller-led revise/replan; it is never an implicit worker decision.

## Worker protocol

若合同足够完成工作、但读取预算耗尽后仍有不影响交付的未决问题，使用 `DONE_WITH_CONCERNS` 交付，并在 handoff 记录未决主张、已核对证据和残余风险；不要把这种情况默认升级为 `NEEDS_CONTEXT`。`NEEDS_CONTEXT` 仅用于问题阻塞合同，或必须改变合同、scope、接口或架构的情况。

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
