# Verification gates

A node reaches `done` only after worker evidence, fresh review, and controller verification all exist. No agent approves its own work.

## 1. Scope gate

Run it against real Git data as soon as the worker reports, before a reviewer is dispatched:

```bash
"${SKILL_ROOT}/scripts/check-scope" <node-id> --head-ref <sha> \
  --handoff-ref ".dag/artifacts/<node-id>/handoff.json"
```

It compares every changed path with `scope.files` and `scope.forbidden`, refuses an unclean worktree, and writes nothing. Generated or untracked files count. A useful out-of-scope edit is still an out-of-scope edit: reject or explicitly re-plan it rather than silently widening the node.

Read its other two reports as evidence rather than verdicts. Paths also claimed by an unfinished node are legal — that is what `conflicts_with` is for — but they are where a worker doing a neighbour's work shows up. Context read beyond the contract is self-reported and a long list indicts the node contract, not the worker.

## 2. Evidence gate

Check that each claimed command, lint, typecheck, build, CLI action, or manual observation has a reproducible command and result tied to the reviewed commit/worktree. Missing, truncated, stale, or pre-fix evidence is not a pass.

Worker evidence can satisfy breadth, but it does not replace the independent controller check.

## 3. Review gate

Confirm a fresh reviewer evaluated:

- the original node contract and acceptance, not criteria it invented;
- the actual diff or immutable review package;
- scope compliance and test validity;
- the immutable worker range and, when present, the reviewer-fix range rather than an earlier commit.

Findings inside scope must be repaired in the reviewer's independent fix commit within the same review; there is no report-only round that defers them to the worker. The reviewer cannot approve its own fix; the controller checks the fix-only range and final acceptance. Spec, scope, or unsafe-repair findings are escalations or contract decisions for the controller. Minor findings may be recorded without another review when they do not undermine acceptance.

An `escalated` outcome is not a fix round. It names paths outside the node's scope that this diff broke, so returning them to the worker only produces a diff the scope gate will refuse. Give every path an owner with `resolve-escalation`; the unchanged head is not re-reviewed. Resolve an unchanged contract question with `resolve-contract`.

## 4. Independent controller gate

The controller personally runs at least one real criterion directly tied to a node acceptance ID. Reading the worker or reviewer report is not a check. Prefer the smallest command or observable probe that would fail if the claimed behavior were absent.

Record command, immutable Git head, timestamp, covered acceptance IDs, exit status, concise output, and the exact Git diff path list in a JSON artifact matching `${SKILL_ROOT}/references/dag-schema.md`. Pass that artifact to `update-task --verification-ref`.

If the check fails, return the concrete finding to the implementer. The controller does not fix it inline by default, because implementing the fix would compromise its independent acceptance role.

## Integration gate

After landing mutually related nodes, test the integrated interface and any global behavior affected by their combination. Per-node green checks do not prove merge order, schema compatibility, generated outputs, or end-to-end flow.

## Final convergence gate

When every node is done, independently compare the integrated project with:

- the original objective;
- every `source_ref` and global acceptance criterion;
- accepted non-goals and compatibility constraints;
- end-to-end behavior and integration evidence.

Classify each global criterion as verified, missing, or partly verified. Check for requirements that never mapped to a node. A missing or partial requirement creates an explicit gap node or `replan_required`; it never becomes an implicit postscript to a done node.

Only passed convergence plus all nodes done permits the DAG to become `complete`.
