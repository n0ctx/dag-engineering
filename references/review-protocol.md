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

> Review only the supplied package; do not survey the repository. Work the checklist below in order. Beyond the files in the diff and the node's `read_first`, read at most five additional files, each to settle one specific named claim, and re-run at most one cheap verification command, only when a specific evidence claim looks doubtful. A claim that cannot be settled within this budget becomes an `unsure` entry naming the exact missing context — never a reason to keep exploring. Every in-scope defect you find, fix on the spot in one commit on the node branch and record it under `bugs`; the runtime rejects `bugs` without a `fix_commit`. Anything you cannot safely repair or should not decide alone goes under `unsure`.

Checklist, in order:

1. Scope: the changed-path list against `scope.files` and `scope.forbidden`. Both are in the package; this needs no repository reading.
2. Acceptance: each criterion against the diff and the supplied evidence, one by one. A design rationale in the handoff is not evidence.
3. Test validity: the changed tests actually exercise the claimed behavior. Read only test files that appear in the diff.
4. Caller impact: only when the diff changes a caller-visible signature, return shape, or exception — run one targeted search for callers and read only the specific call sites at issue.
5. Security, error handling, and unnecessary abstraction: judge from the diff itself, not from a codebase survey.

Re-running the worker's full verification suite belongs to the controller's independent gate, not to review. For a `micro` or `tier2` node, stop after item 3 and escalate any remaining doubt as `unsure` instead of auditing design. Where the harness allows choosing the reviewer model, a cheaper model suffices for these checklist-only reviews.

## Findings and decisions

Disclose every finding completely before deciding the verdict. A user's explicit, durable decision outranks subjective review preferences and tradeoffs, but it must use the supported resolution or revise/replan transition; it does not fabricate an approved node review. Runtime-enforced schema, acyclicity, path/scope, Git-SHA, evidence-binding, and external-authorization checks remain mandatory. Any contract or acceptance change requires controller-led revise/replan and cannot be covered silently.

Every finding lands in exactly one field:

- `bugs`: an in-scope defect the reviewer has already repaired in its fix commit; `detail` names the file, what was wrong, and how it was fixed;
- `unsure`: everything the reviewer cannot or should not settle alone, with a `reason`:
  - `contract`: the contract or acceptance text admits two plausible readings — the controller settles it with `resolve-contract`;
  - `scope`: the defect sits outside `scope.files` or inside `scope.forbidden`, where a fix would trip the scope gate — the controller assigns an owner with `resolve-escalation`;
  - `unsafe`: repairing it would exceed the review package or risk behavior the reviewer cannot re-verify — the controller decides.

In-scope repairs never go back to the worker. The reviewer repairs every bug in the same session: at most one independent fix commit, entirely within scope. There is no report-only fix round — the runtime rejects a review whose `bugs` lack a `fix_commit`, and a `fix_commit` without `bugs`. The reviewer cannot approve that fix; the controller performs the final acceptance check. An escalation does not complete the node.

## Review artifact

Save a project-relative JSON artifact and include the final reviewed head:

```json
{
  "node": "node-id",
  "round": 1,
  "head_ref": "full final Git commit SHA",
  "worker_head": "full worker Git commit SHA",
  "fix_commit": null,
  "bugs": [],
  "unsure": []
}
```

- a bug entry is `{"id": "b1", "detail": "what was wrong and how it was fixed", "path": "src/foo.ts"}`; `path` must be inside `scope.files`;
- an unsure entry is `{"id": "u1", "detail": "...", "path": "src/foo.ts", "reason": "contract|scope|unsafe"}`;
- omit `worker_head` (or set it null) when the reviewer lands no fix commit; it is required when `fix_commit` is present;
- `fix_commit` must equal `head_ref` and be an independent, non-empty, in-scope descendant of `worker_head`.

The runtime derives the outcome from `unsure`: any `reason: "contract"` entry gives `cannot_verify`; any other `unsure` entry gives `escalated`; otherwise `approved`. Neither worker nor controller supplies an unverified approval flag. After review, the controller independently verifies a real acceptance criterion and records the runtime transition.
