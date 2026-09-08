# Node contract and agent protocols

The controller packages one node for a fresh worker. The node contract is binding; worker and reviewer reports cannot amend it.

## Worker package

Provide:

```text
node id and title
objective
scope.files and scope.forbidden
read_first
inputs
precise upstream outputs and handoff refs
expected outputs
acceptance
verification
worktree/path
report artifact path
project constraints that bind this node
```

Do not provide full chat history, the whole DAG, every upstream report, or the implementer's future reviewer prompt.

## Worker protocol

The worker:

1. Reads the contract and listed context before editing.
2. Raises a scope or contract problem before guessing.
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

For `NEEDS_CONTEXT` or `BLOCKED`, report facts, the exact blocker, what was tried, and the decision/context needed. Do not keep exploring without a bounded hypothesis.

## Worker handoff

Write a compact artifact, preferably `.dag/artifacts/<node-id>/handoff.json`:

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
  "concerns": [],
  "unresolved": [],
  "downstream_notes": [],
  "commit": "full Git commit SHA"
}
```

Keep implementation chronology, searches, failed commands, and reasoning out of the handoff. Link durable reports instead.

## Fresh reviewer package

Provide only the original node contract, actual diff or immutable diff artifact, base/head refs, structured handoff, and relevant test evidence. The reviewer is read-only and does not receive the implementer's reasoning transcript.

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

Every blocking finding names severity, acceptance/scope impact, and file:line evidence. A design choice or defect in the node contract goes to the controller; the reviewer does not silently choose or edit the contract. Out-of-scope observations are reported but not fixed.

Output:

```json
{
  "node": "node-id",
  "head_ref": "full Git commit SHA",
  "round": 1,
  "spec": "APPROVED | NEEDS_FIXES | CANNOT_VERIFY",
  "quality": "APPROVED | NEEDS_FIXES",
  "blocking_findings": [],
  "minor_findings": [],
  "out_of_scope_observations": [],
  "checks_performed": []
}
```

Save it as a project-relative JSON artifact. The runtime derives the review outcome from these verdicts and findings; neither implementer nor controller passes an unverified `approved` flag.

## Scoped re-review

Re-review receives the original contract, prior blocking finding IDs, the fix-only diff, and appended verification evidence. It verdicts every prior finding `ADDRESSED` or `NOT_ADDRESSED`, then identifies only new breakage introduced by the fix. It does not restart broad review or change acceptance.

After three unsuccessful fix/re-review rounds, return the unresolved finding IDs to the controller. The controller chooses more context, a stronger model, re-slicing, replanning, or `blocked`.
