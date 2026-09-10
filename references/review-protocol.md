# Review protocol

Review is a fresh, independent check of the node contract and actual Git change. The reviewer does not receive the implementer's reasoning transcript and must not self-certify a repair.

## Strict reviewer package

Provide only these inputs:

- the original node contract;
- immutable `base_ref..worker_head` diff and changed paths;
- the structured handoff;
- relevant verification evidence;
- output paths;
- the node worktree path and expected branch, so the reviewer can commit its fix there.

Do not attach a controller summary, restated contract, whole DAG, unrelated reports, chat history, or worker reasoning. Review against the original contract.

## Bounded review procedure

The controller starts the reviewer prompt with this fixed preamble:

> Review only the supplied package; do not survey the repository. Work the checklist below in order. Beyond the files in the diff and the node's `read_first`, read at most five additional files, each to settle one specific named claim, and log every such read in `checks_performed`. Re-run at most one cheap verification command, only when a specific evidence claim looks doubtful. A claim that cannot be settled within this budget becomes a recorded finding naming the exact missing context — never a reason to keep exploring. Repair every in-scope defect you find in one commit on the node branch; the runtime rejects a review that reports in-scope defects without fixing them. A defect you cannot safely repair belongs in `controller_decisions`, not in `blocking_findings`.

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

- `blocking_findings`: an in-scope defect the reviewer repairs in its own fix commit;
- `controller_decisions`: an out-of-scope, forbidden-scope, contract, unsafe-repair, or changed-caller problem the controller must resolve;
- `out_of_scope_observations`: unrelated observations that are not caused by this diff and do not block the node.

In-scope repairs never go back to the worker. The reviewer repairs every in-scope blocking finding in the same session: at most one independent fix commit, entirely within scope, with one `ADDRESSED` resolution per blocking finding. There is no report-only fix round — the runtime rejects a review whose blocking findings lack a fix commit. The reviewer cannot approve that fix; the controller performs the final acceptance check. An escalation does not complete the node.

## Review artifact

Save a project-relative JSON artifact and include the final reviewed head:

```json
{
  "node": "node-id",
  "round": 1,
  "head_ref": "full final Git commit SHA",
  "worker_head": "full worker Git commit SHA",
  "reviewer_fix_head": null,
  "spec": "APPROVED | CANNOT_VERIFY",
  "blocking_findings": [],
  "finding_resolutions": [],
  "controller_decisions": [],
  "minor_findings": [],
  "out_of_scope_observations": [],
  "checks_performed": []
}
```

The runtime derives the outcome from verdicts and findings; neither worker nor controller supplies an unverified approval flag. After review, the controller independently verifies a real acceptance criterion and records the runtime transition.
