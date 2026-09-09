# Review protocol

Review is a fresh, independent check of the node contract and actual Git change. The reviewer does not receive the implementer's reasoning transcript and must not self-certify a repair.

## Strict reviewer package

Provide only these inputs:

- the original node contract;
- immutable `base_ref..worker_head` diff and changed paths;
- the structured handoff;
- relevant verification evidence;
- output paths.

Do not attach a controller summary, restated contract, whole DAG, unrelated reports, chat history, or worker reasoning. Review against the original contract. Read outside the diff only `read_first` files or a specific caller needed to settle a claim. If a changed signature, return shape, exception, or other caller-visible contract is involved, search its callers before judging it.

Verify claims against the diff and reproducible evidence. Check the acceptance criteria, caller/integration impact, test validity, security, error handling, and unnecessary abstraction. A design rationale in the handoff is not evidence.

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
