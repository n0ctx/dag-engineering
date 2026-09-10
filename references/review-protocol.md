# Review protocol

Review is a fresh, independent check of the node contract and actual Git change. The reviewer does not receive the implementer's reasoning transcript and must not self-certify a repair.

## Strict reviewer package

Provide only these inputs:

- the original node contract;
- immutable `base_ref..worker_head` diff and changed paths;
- the structured handoff;
- relevant verification evidence;
- output paths.

Do not attach a controller summary, restated contract, whole DAG, unrelated reports, chat history, or worker reasoning. Review against the original contract.

## Bounded review procedure

The controller starts the reviewer prompt with this fixed preamble:

> Review only the supplied package; do not survey the repository. Work the checklist below in order. Beyond the files in the diff and the node's `read_first`, read at most five additional files, each to settle one specific named claim, and log every such read in `checks_performed`. Re-run at most one cheap verification command, only when a specific evidence claim looks doubtful. A claim that cannot be settled within this budget becomes a recorded finding naming the exact missing context — never a reason to keep exploring.

Checklist, in order:

1. Scope: the changed-path list against `scope.files` and `scope.forbidden`. Both are in the package; this needs no repository reading.
2. Acceptance: each criterion against the diff and the supplied evidence, one by one. A design rationale in the handoff is not evidence.
3. Test validity: the changed tests actually exercise the claimed behavior. Read only test files that appear in the diff.
4. Caller impact: only when the diff changes a caller-visible signature, return shape, or exception — run one targeted search for callers and read only the specific call sites at issue.
5. Security, error handling, and unnecessary abstraction: judge from the diff itself, not from a codebase survey.

Re-running the worker's full verification suite belongs to the controller's independent gate, not to review. For a `micro` or `tier2` node, stop after item 3 and escalate any remaining doubt as a finding instead of auditing design. Where the harness allows choosing the reviewer model, a cheaper model suffices for these checklist-only reviews.

## Findings and decisions

Disclose every finding completely before deciding the verdict. A user's explicit, durable decision outranks subjective review preferences and tradeoffs, but it must use the supported resolution or revise/replan transition; it does not fabricate an approved node review. Runtime-enforced schema, acyclicity, path/scope, Git-SHA, evidence-binding, and external-authorization checks remain mandatory. Any contract or acceptance change requires controller-led revise/replan and cannot be covered silently.

Put findings in the path-appropriate field:

- `blocking_findings`: an in-scope defect the reviewer may safely repair;
- `controller_decisions`: an out-of-scope, forbidden-scope, contract, unsafe-repair, or changed-caller problem the controller must resolve;
- `out_of_scope_observations`: unrelated observations that are not caused by this diff and do not block the node.

Do not send an in-scope repair back for another worker round. The reviewer may make at most one independent fix commit, entirely within scope, and must provide one `ADDRESSED` resolution for each original in-scope blocking finding. The reviewer cannot approve that fix; the controller performs the final acceptance check. An escalation does not complete the node.

## Review artifact

Save a project-relative JSON artifact and include the final reviewed head:

```json
{
  "node": "node-id",
  "round": 1,
  "head_ref": "full final Git commit SHA",
  "worker_head": "full worker Git commit SHA",
  "reviewer_fix_head": null,
  "spec": "APPROVED | NEEDS_FIXES | CANNOT_VERIFY",
  "quality": "APPROVED | NEEDS_FIXES",
  "blocking_findings": [],
  "finding_resolutions": [],
  "controller_decisions": [],
  "minor_findings": [],
  "out_of_scope_observations": [],
  "checks_performed": []
}
```

The runtime derives the outcome from verdicts and findings; neither worker nor controller supplies an unverified approval flag. After review, the controller independently verifies a real acceptance criterion and records the runtime transition.
