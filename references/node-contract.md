# Node contract and agent protocols

The controller packages one node for a fresh worker. The node contract is binding; worker and reviewer reports cannot amend it.

## Worker package

Provide:

```text
node id and title
objective
scope.files and scope.forbidden
read_first (the complete declared context, not a starting point)
inputs
precise upstream outputs and handoff refs
expected outputs
acceptance
verification
worktree/path
report artifact path (absolute; a relative one lands in the worker's own worktree)
project constraints that bind this node
```

Do not provide full chat history, the whole DAG, every upstream report, or the implementer's future reviewer prompt.

State the search boundary in the same package, because `scope.files` bounds what the worker may write and nothing bounds what it may read:

> Read every file in `read_first` first; it is the whole context this node was planned with. Beyond it, search only inside `scope.files` and the paths those files name directly, such as an import or a caller you must change. Do not survey the repository, read unrelated modules, read Git history, or read documentation the contract does not name. If you cannot proceed on this context, report `NEEDS_CONTEXT` naming the file you need — do not go find it yourself.

## Worker protocol

The worker:

1. Reads all of `read_first` before searching or editing, and treats it as complete rather than as a starting point.
2. Raises a scope or contract problem before guessing, and returns `NEEDS_CONTEXT` naming the missing file rather than exploring outside the boundary to compensate for a thin contract.
3. Implements only the declared objective and scope.
4. Runs focused verification and records reproducible evidence.
5. Self-reviews the actual diff, but does not approve the node.
6. Commits the node when the assigned workflow uses commits.
7. Writes a short structured handoff artifact and returns only its status and reference.

The next block is copied exactly from Superpowers' current implementer prompt because recursive delegation would violate the same controller/reviewer separation here:

## You Do Not Dispatch Subagents

Do all of this task's work yourself. Never spawn a subagent to
implement part of the task, and above all never spawn a reviewer to
check your work. Self-review (below) means reading your own diff.
Review is the controller's job: after you report, it dispatches a
fresh reviewer against your diff. A reviewer you spawn duplicates
that review at full cost, and its approval counts for nothing in
the process. If you catch yourself thinking "an independent review
would strengthen my report" — that review is already scheduled.
Report instead.

The worker must stop instead of expanding scope when a required file is outside `scope.files`, a forbidden file must change, code reality contradicts the contract, an upstream output is absent, acceptance cannot be met, or a new architecture choice is required.

Use exactly one status:

- `DONE`: work and declared verification completed without known concern;
- `DONE_WITH_CONCERNS`: requested work completed, but correctness or integration doubt remains;
- `NEEDS_CONTEXT`: a specific missing fact, artifact, or decision is required;
- `BLOCKED`: the worker cannot complete the node with the current contract or capability.

For `NEEDS_CONTEXT` or `BLOCKED`, report facts, the exact blocker, what was tried, and the decision/context needed. Do not keep exploring without a bounded hypothesis. Stopping is still a report: write the handoff artifact at the supplied path with that status, `commit` and `files_changed` empty, and the blocker in `unresolved`. A stop returned only as chat prose leaves the controller nothing durable to act on, and the node cannot be resolved from it.

## Worker handoff

Write a compact artifact at the exact path the controller supplied, normally `<control-plane-root>/.dag/artifacts/<node-id>/handoff.json`:

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

`context_used` is required and may be empty. It is a self-report, so it gates nothing; it exists so that a node which had to read forty files to start is visible as a defective contract rather than as an invisible cost.

Keep implementation chronology, searches, failed commands, and reasoning out of the handoff. Link durable reports instead.

## Fresh reviewer package

Provide only the original node contract, actual diff or immutable diff artifact, base/head refs, structured handoff, relevant test evidence, and any paths `check-scope` reported as also claimed by an unfinished node. The reviewer is read-only and does not receive the implementer's reasoning transcript.

Bound the reviewer's reading too: judge the diff against the contract, and open a file outside the diff and `read_first` only to settle a specific claim the diff makes. Reviewing is not a repository survey.

The next block is copied exactly from Superpowers' current task reviewer prompt because its anti-anchoring rule is identical here:

## Do Not Trust the Report

Treat the implementer's report as unverified claims about the code. It
may be incomplete, inaccurate, or optimistic. Verify the claims against
the diff. Design rationales in the report are claims too: "left it per
YAGNI," "kept it simple deliberately," or any other justification is the
implementer grading their own work. Judge the code on its merits — a
stated rationale never downgrades a finding's severity.

The reviewer checks:

- each original acceptance criterion: verified, missing, or not verifiable from the package;
- missing, extra, or misunderstood behavior;
- edits outside `scope.files` or inside `scope.forbidden`;
- behavior regressions and missing caller/integration changes;
- test validity, including assertions that do not prove the criterion;
- code quality, security, error handling, and unnecessary abstraction.

A design choice or defect in the node contract goes to the controller as `spec: CANNOT_VERIFY`; the reviewer does not silently choose or edit the contract.

Output:

```json
{
  "node": "node-id",
  "head_ref": "full Git commit SHA",
  "round": 1,
  "spec": "APPROVED | NEEDS_FIXES | CANNOT_VERIFY",
  "quality": "APPROVED | NEEDS_FIXES",
  "blocking_findings": [
    {"severity": "critical", "path": "src/auth/client.py", "detail": "acceptance/scope impact and file:line evidence"}
  ],
  "controller_decisions": [],
  "minor_findings": [],
  "out_of_scope_observations": [],
  "checks_performed": []
}
```

Save it as a project-relative JSON artifact. The runtime derives the review outcome from these verdicts and findings; neither implementer nor controller passes an unverified `approved` flag.

### Where a blocking problem goes

Every blocking finding carries the project-relative `path` it is about, and that path decides its bucket. The runtime enforces the split both ways and rejects a misfiled finding.

- **`blocking_findings`** — the path is inside `scope.files`. The worker can fix it, so it drives the fix loop.
- **`controller_decisions`** — the path is outside `scope.files` or inside `scope.forbidden`, and this diff made it a problem: a changed signature breaks a caller the worker may not touch, a migration needs a companion change elsewhere. The worker fixing it would trip the scope gate, so it is the controller's call.
- **`out_of_scope_observations`** — noticed outside the scope but not caused by this diff and not blocking. Reported, never actioned here.

Do not move a fixable defect into `controller_decisions` to skip a fix round, and do not put an out-of-scope breakage into `blocking_findings`: the worker would have to leave its scope to satisfy it, and the scope gate will refuse the result.

A review with escalations and no blocking findings comes back as `escalated`, which does not complete the node. The controller gives those paths an owner — a gap node or a replan — and then re-reviews the same head, where the escalation no longer stands because someone now owns it.

## Scoped re-review

Re-review receives the original contract, prior blocking finding IDs, the fix-only diff, and appended verification evidence. It verdicts every prior finding `ADDRESSED` or `NOT_ADDRESSED`, then identifies only new breakage introduced by the fix. It does not restart broad review or change acceptance.

After three unsuccessful fix/re-review rounds, return the unresolved finding IDs to the controller. The controller chooses more context, a stronger model, re-slicing, replanning, or `blocked`.
