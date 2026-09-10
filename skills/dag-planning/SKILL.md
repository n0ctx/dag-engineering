---
name: dag-planning
description: Internal module compiling approved or proposed engineering requirements into a persistent DAG; load only through dag-engineering.
---

# DAG Planning

Produce `<project>/.dag/dag.json`. Do not implement its nodes.

## Hard stop for a newly proposed DAG

If `.dag/dag.json` did not exist at the start of the user turn and no durable plan was approved before that turn, the turn is planning-only. “Start,” “do it,” and “execute” cannot approve a DAG the user has not seen.

When writing or validating an `awaiting_approval` DAG:

1. show a compact summary;
2. ask for approval;
3. end the response.

Do not load execution, record approval, dispatch workers, or edit files outside `.dag/` in that turn. The user's wish to complete a project is not approval of a particular decomposition.

## Load only what planning needs

Before drafting, read:

- `${SKILL_ROOT}/references/dag-schema.md`
- `${SKILL_ROOT}/references/decomposition.md`

Read `${SKILL_ROOT}/references/scheduling.md` only when dependency, conflict, cost, or worktree choices are non-trivial. Do not load `node-contract.md` for ordinary planning.

## Evidence rule

Planning facts come from the user's words, durable source documents, or the repository. Everything else is an open choice.

- Carry objectives, requirements, constraints, non-goals, compatibility requirements, and approval state only when a source establishes them. Naming cases in scope does not exclude unnamed cases.
- A framework, protocol, interface shape, payload, status code, storage, session/token model, or security mechanism is not a fact unless the source or repository establishes it. Otherwise record it as an assumption awaiting approval or as the output of a bounded investigation.
- Keep this distinction visible in `.dag/sources/`, the DAG, and node contracts.

## 0. Normalize the source

Classify the input as a PRD, roadmap, spec, issue, checklist, existing plan, natural-language request, or partial implementation. Preserve durable paths, URLs, issue identifiers, and commit references in `source_refs`; do not paste source documents into the DAG.

If requirements exist only in chat, write a concise faithful snapshot to `.dag/sources/<dag-id>.md`, label it `user-request: <date>`, and read it back before relying on it. Extract the objective, observable global acceptance, explicit non-goals, constraints, compatibility requirements, known Git state, and decisions downstream work may rely on. A source record is a faithful account, not a design document.

For natural-language, unapproved, or mutually ambiguous sources, start with `planning_status: awaiting_approval`. A new DAG starts only as `draft` or `awaiting_approval`; approval inferred during this turn does not qualify as prior approval.

## 1. Directed reconnaissance

Read only enough code and documentation to identify owners, interfaces, nearby implementation, tests, and integration boundaries. Prefer targeted search and at most one bounded exploration. Record evidence and real paths; if ownership remains unknown, keep planning open or create a bounded investigation node.

## 2. Clarify load-bearing ambiguity

After reconnaissance, ask only questions whose answers could change the objective, scope, acceptance, compatibility, interface ownership, migration safety, or graph. Do not repeat choices settled by sources or the repository.

Persist the normalized request first and verify the file exists. If read-back fails, report the persistence failure and ask no clarification questions.

Facts are the controller's work; decisions belong to the user. Ask one frontier round of at most three questions. Each question must have bounded options, a recommendation, its reason, and its main tradeoff:

```text
Q1. <decision and bounded options>
Recommended: <one option>, because <reason>; tradeoff: <main cost>.
```

Do not ask downstream technology questions while product intent, delivery boundary, scale, or acceptance is open. If one answer changes another question's options, ask the upstream question first. After each answer, recompute the frontier.

Persist questions, answers, source facts, and resulting decisions in `.dag/sources/<dag-id>-clarifications.md` and add it to `source_refs`. Keep proposed assumptions separate from user decisions. Exit only when each remaining unknown is an explicit assumption awaiting approval or a bounded investigation output. Clarification establishes planning inputs; it does not approve the DAG or authorize implementation.

## 3. Decompose

Apply `${SKILL_ROOT}/references/decomposition.md`. Every node in the new schema has a non-empty string-array `execution_plan`, normally 1–6 ordered steps. Each string uses `project-relative landing (to a symbol or section when known)—concrete action; covers AC*/V*`. Do not repeat the node's objective, scope, inputs, outputs, or acceptance, and do not copy verification commands. If the landing cannot be identified, make that uncertainty a bounded investigation node instead of inventing a path. Do not duplicate the remaining node-sizing, dependency, conflict, or contract rules here.

Every newly created DAG also declares `execution_tiering.version: 1` and one profile per node. Use `micro` only for a 2–5 minute mechanical leaf, `tier2` only for a 5–15 minute decision-free leaf, and reserve `senior` or `controller` for an explicit design decision or graph-wide convergence. A `tier2` node must have one behavior, one bounded ownership surface, complete `read_first` context, and a defined `NEEDS_CONTEXT` stop; split it otherwise. Runtime rejects missing profiles, invalid estimates, and `micro`/`tier2` scopes wider than their limits before review or approval.

## 4. Draft and validate

Write `.dag/draft.json`, then let the runtime install it:

```bash
"${SKILL_ROOT}/scripts/update-task" --init .dag/draft.json
"${SKILL_ROOT}/scripts/status"
```

`--init` validates and installs `.dag/dag.json`, consumes the draft, and refuses to replace an existing control file. Never write `.dag/dag.json` directly.

If an unfinished DAG already exists, report its state and ask whether to resume or drop it. Dropping it requires the user's persisted instruction:

```bash
"${SKILL_ROOT}/scripts/update-task" --abandon --reason "<why>" --reference "<path under .dag/sources/>"
```

A new request superseding an existing plan is not, by itself, a reason to abandon it. Fix validator errors before presenting the plan. Preview coverage by mapping each source requirement and global acceptance criterion to nodes and verification paths, including integration work. Audit every technology or concrete interface, payload, status code, storage, or security choice against source evidence, an `assumptions` entry, or an investigation output.

## 5. Review the decomposition

Before showing the plan, create one `review-request.json` containing the plan fingerprint, the complete review input, and the three lanes (`requirements`, `graph`, `execution`). Dispatch one independent reviewer **with file-editing capability** to examine all three lanes in one pass and return one `review.json`; do not create `plan.json`, per-lane files, or a manifest. The reviewer works fix-first, like an execution reviewer: copy the plan from the request package into `.dag/draft.json`, repair every defect it can fix directly in that draft, and record each one as a bug; everything that needs the user or controller goes into `unsure`. The single result uses a `lanes` object: each required lane appears exactly once as `{checks_performed: [...], bugs: [...], unsure: [...]}`. A full review has all three lanes; a repair has only the lanes named by the runtime request and retains `resolutions` confirming each prior open item. The request includes only the contracts, relevant source references, and repository evidence needed for those lanes. A repair request contains only affected nodes and lanes, plus the evidence needed to reassess them and their downstream interfaces.

Bind the dispatched reviewer to the request package. It judges the plan against the supplied contracts, source references, and cited repository evidence, and works the required lanes linearly rather than re-running reconnaissance. Its repository access is limited to targeted existence checks on paths and symbols the plan cites; a citation it cannot confirm is a finding, not an invitation to locate the right one. Any claim it cannot settle from the package becomes an `unsure` entry naming the missing evidence. Every repository check it performs goes into the lane's `checks_performed`.

```bash
"${SKILL_ROOT}/scripts/status" --review-request > .dag/artifacts/decomposition-review/review-request.json
```

The reviewer must complete and disclose the full requested review. A review is independent advice, not a decision-maker above the user: `approval_ref` may record the user's clear, durable decision to approve or continue after seeing an open `unsure` item. That decision cannot bypass runtime-enforced schema, acyclicity, paths, scope, Git/SHA, evidence-binding, or authorization-boundary checks. Record the durable decision and rationale where the control plane requires it; do not suppress, rewrite, or omit findings.

Record the verdict and the reviewer's own fixes in one step; pass the draft only when the review reports bugs:

```bash
"${SKILL_ROOT}/scripts/update-task" --decomposition-review ".dag/artifacts/decomposition-review/review.json"
"${SKILL_ROOT}/scripts/update-task" --decomposition-review ".dag/artifacts/decomposition-review/review.json" \
  --draft ".dag/draft.json"
```

The runtime validates the review against the current fingerprint, applies the draft under the amendment protections (the execution record, attempted and done nodes, and the deliverable fields cannot change), and derives the verdict: any `unsure` entry means `needs_changes`; otherwise `approved`. The reviewer's own fixes need no further repair round. Remaining `unsure` items are resolved by amending the plan, not by abandoning and recreating it; an amendment clears the verdict and the next repair review includes only affected nodes and lanes.

```bash
"${SKILL_ROOT}/scripts/update-task" --amend ".dag/draft.json"
```

## 6. Approval gate

For a newly generated or vague DAG, show objective, nodes, dependency edges, conflicts, expected parallel frontier, global acceptance, and every `assumptions` entry. Then stop for explicit approval. Default `max_parallel` to 1 unless safe isolation and meaningful wall-clock benefit are established.

Record approval only after persisting the exact approval and accepted DAG version under `.dag/sources/`:

```bash
"${SKILL_ROOT}/scripts/update-task" --planning-status approved --reference "<durable approval reference>"
```

Recording approval ends the planning turn. Ask the user to start a new session for `/dag-engineering continue`; execution may not run in the session that planned the DAG, and compacting that session is not a substitute.
