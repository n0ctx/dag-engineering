# Review protocol

Load this when reviewing a reported node, not when dispatching one. It defines the fresh reviewer's package, verdict schema, and the boundary between what the worker can fix and what only the controller can. The node contract it judges against is in `${CLAUDE_SKILL_DIR}/references/node-contract.md`.

## Fresh reviewer package

Provide only the original node contract, actual diff or immutable diff artifact, base/head refs, structured handoff, relevant test evidence, and any paths `check-scope` reported as also claimed by an unfinished node. The reviewer is read-only and does not receive the implementer's reasoning transcript.

Bound the reviewer's reading: judge the diff against the contract, and open a file outside the diff and `read_first` only to settle a specific claim the diff makes. Reviewing is not a repository survey.

Searching for callers is one of those claims, not a survey. When the diff changes a signature, return shape, raised exception, or any other caller-visible contract, find the callers before judging it — `grep` for the changed name across the repository. A caller the diff broke is a finding whether or not it sits inside `scope.files`; skipping the search is how a node passes review and breaks the build.

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

A review with escalations and no blocking findings comes back as `escalated`, which does not complete the node,
whatever the `spec` and `quality` verdicts say: with nothing named inside `scope.files` there is no fix for the
worker to make. A `NEEDS_FIXES` verdict that names no finding at all is rejected outright. The controller gives those paths an owner — a gap node or a replan — and then re-reviews the same head, where the escalation no longer stands because someone now owns it.

## Scoped re-review

Re-review receives the original contract, prior blocking finding IDs, the fix-only diff, and appended verification evidence. It verdicts every prior finding `ADDRESSED` or `NOT_ADDRESSED`, then identifies only new breakage introduced by the fix. It does not restart broad review or change acceptance.

After three unsuccessful fix/re-review rounds, return the unresolved finding IDs to the controller. The controller chooses more context, a stronger model, re-slicing, replanning, or `blocked`.
