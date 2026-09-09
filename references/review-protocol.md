# Review protocol

Load this when reviewing a reported node. It defines the bounded reviewer package, the reviewer-fix handoff, and the boundary between what the reviewer may fix and what only the controller can decide. The node contract it judges against is in `${SKILL_ROOT}/references/node-contract.md`.

## Fresh reviewer package

Provide only the original node contract, immutable `base_ref..worker_head` worker diff, `base_ref`, `worker_head`, structured handoff, relevant test evidence, and any paths `check-scope` reported as also claimed by an unfinished node. The reviewer does not receive the implementer's reasoning transcript. If a node-scope defect is safe to repair, it makes one independent reviewer fix commit and reports its `reviewer_fix_head`; otherwise it escalates to the controller.

Bound the reviewer's reading: judge the diff against the contract, and open a file outside the diff and `read_first` only to settle a specific claim the diff makes. Reviewing is not a repository survey.

Searching for callers is one of those claims, not a survey. When the diff changes a signature, return shape, raised exception, or any other caller-visible contract, find the callers before judging it — `grep` for the changed name across the repository. A caller the diff broke is a finding whether or not it sits inside `scope.files`; skipping the search is how a node passes review and breaks the build.

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
  "round": 1,
  "head_ref": "full final Git commit SHA",
  "worker_head": "full worker Git commit SHA",
  "reviewer_fix_head": null,
  "spec": "APPROVED | NEEDS_FIXES | CANNOT_VERIFY",
  "quality": "APPROVED | NEEDS_FIXES",
  "blocking_findings": [
    {"id": "scope-1", "severity": "critical", "path": "src/auth/client.py", "detail": "acceptance/scope impact and file:line evidence"}
  ],
  "finding_resolutions": [],
  "controller_decisions": [],
  "minor_findings": [],
  "out_of_scope_observations": [],
  "checks_performed": []
}
```

`head_ref` is the final reviewed head and is required. With a reviewer repair,
`worker_head` is the worker commit and `reviewer_fix_head` is a non-null,
different descendant containing only node-scope changes; the handoff still
describes `worker_head`. Put the original in-scope blocking findings in
`blocking_findings` and provide exactly one `ADDRESSED` entry for each in
`finding_resolutions`. A new review must settle these in the same artifact;
it does not first record `needs_fixes`. The legacy `resolutions` field is read
only for already-recorded historical `needs_fixes` attempts and must not be
used by new reviews.

Save it as a project-relative JSON artifact. The runtime derives the review outcome from these verdicts and findings; neither implementer nor controller passes an unverified `approved` flag.

### Where a blocking problem goes

Every blocking finding carries the project-relative `path` it is about, and that path decides its bucket. The runtime enforces the split both ways and rejects a misfiled finding.

- **`blocking_findings`** — the path is inside `scope.files`. The reviewer may fix it in the reviewer-fix commit; it is never sent back to the worker for another review round.
- **`controller_decisions`** — the path is outside `scope.files` or inside `scope.forbidden`, and this diff made it a problem: a changed signature breaks a caller the worker may not touch, a migration needs a companion change elsewhere. The worker fixing it would trip the scope gate, so it is the controller's call.
- **`out_of_scope_observations`** — noticed outside the scope but not caused by this diff and not blocking. Reported, never actioned here.

Do not escalate a safe in-scope defect to skip the reviewer fix, and do not put an out-of-scope breakage into `blocking_findings`: the reviewer or worker would have to leave its scope to satisfy it, and the scope gate will refuse the result.

A review with escalations comes back as `escalated` and does not complete the node. The controller gives each path a legal owner with `resolve-escalation`, without re-reviewing the unchanged worker head. A `NEEDS_FIXES` verdict must name stable findings and either a reviewer fix head or an explicit escalation.

## Reviewer fix acceptance

The controller validates both immutable ranges: `base_ref..worker_head` for the worker delivery and, when present, `worker_head..reviewer_fix_head` for the reviewer fix. The fix head must be a new descendant commit, remain entirely within node scope, and include resolutions for every blocking finding. The controller runs at least one independent acceptance check against the final head. The reviewer cannot approve its own fix; final acceptance is a controller decision.
