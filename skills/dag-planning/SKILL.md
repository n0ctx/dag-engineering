---
name: dag-planning
description: Internal module for compiling approved or proposed engineering requirements into a persistent DAG; load only through dag-engineering.
---

# DAG Planning

Produce `<project>/.dag/dag.json`. Do not implement its nodes.

## Hard stop for a newly proposed DAG

If `.dag/dag.json` did not exist at the start of this user turn and there was no durable plan approved before the turn, this entire turn is planning-only. The same message that says “start,” “do it,” “execute,” or similar cannot approve a DAG that did not yet exist for the user to inspect.

After writing and validating an `awaiting_approval` DAG:

1. show the compact summary;
2. ask for approval;
3. end the response immediately.

Do not load the execution module, run an approval transition, dispatch any worker, or edit files outside `.dag/` in that turn. There is no implied or self-issued approval. Seeing that the user wants the project completed is not evidence that they approved this particular decomposition.

## Load only what planning needs

Before drafting the DAG, read:

- `${CLAUDE_SKILL_DIR}/references/dag-schema.md`
- `${CLAUDE_SKILL_DIR}/references/decomposition.md`
- `${CLAUDE_SKILL_DIR}/references/node-contract.md`

Read `${CLAUDE_SKILL_DIR}/references/scheduling.md` only when dependency, conflict, cost, or worktree choices are non-trivial.

## Evidence rule

A planning fact points to the user's words, a durable source document, or the repository. Everything else is an open choice.

- Objective, requirements, constraints, and non-goals carry only what the source establishes. Naming the cases in scope does not put the unnamed ones out of scope.
- A framework, protocol, interface shape, payload, status code, storage, session/token model, or security mechanism the source and repository do not establish is never a fact, and never a default. A filename such as `app.py`, a placeholder module, or `/route` notation is not evidence of Flask, FastAPI, a queue, a database, or an auth scheme.
- When such a choice is needed to make the plan executable it becomes a top-level `assumptions` entry shown at approval, or a bounded investigation node's output, never a silent default.

This rule binds `.dag/sources/` records, the DAG, and node contracts alike.

## 0. Normalize the source

Classify the input as one or more of: PRD, roadmap, spec, issue, checklist, existing plan, natural-language request, or partial implementation. Preserve durable paths, URLs, issue identifiers, and commit references in `source_refs`. Do not paste full source documents into the DAG. If requirements exist only in chat, write a concise, faithful snapshot to `.dag/sources/<dag-id>.md` and reference that path; a label such as `user-request: <date>` is not recoverable source state.

Extract:

- the global objective and observable global acceptance criteria;
- explicit non-goals, constraints, compatibility requirements, and approval state;
- known partial implementation and current Git state;
- decisions that downstream work may rely on.

Every record written under `.dag/sources/` is a faithful account, not a design document. Apply the evidence rule and keep stated facts visibly separate from open choices. Copy a non-goal only where the source states the exclusion: listing the channels, platforms, or cases in scope leaves the unnamed ones open, not excluded.

If the source is a natural-language request or an unapproved/mutually ambiguous document, set `planning_status` to `awaiting_approval`. A DAG is always created as `draft` or `awaiting_approval`; the runtime rejects any other starting state. When a clearly approved written plan predates this turn, it still becomes a reviewed DAG first, and its durable approval reference is then recorded through the approval transition. An approval created or inferred during the current planning turn does not qualify.

## 1. Directed reconnaissance

Read only enough code and project documentation to identify owners, interfaces, nearby implementation, tests, and integration boundaries. Prefer targeted search and known architecture documents. Use one bounded exploration when necessary; do not launch several open-ended explorers to fill concurrency slots.

Record real paths in `scope.files` and `read_first`. Reconnaissance produces evidence, not certainty: where ownership or an interface boundary stays unknown, keep planning open or create a bounded investigation/decision node.

## 2. Clarify load-bearing ambiguity

After reconnaissance, scan for unresolved choices that would materially change the objective, non-goals, acceptance, compatibility, architecture or interface ownership, migration safety, or the DAG's dependencies and conflicts. Run an interview only when such choices exist or the user explicitly asks to brainstorm or stress-test the request. Skip it when durable sources and repository facts already settle the plan, and never repeat decisions an approved plan already records.

Before formulating or sending the first question, write the normalized request to a durable path in `source_refs`, then read that file back and confirm it exists. When the request came from chat, its faithful snapshot belongs under `.dag/sources/`. If the read-back fails or no source file exists, the response may only report that persistence failed; it must not contain clarification questions. Never claim to be following the clarification workflow while leaving the first round recoverable only from chat.

Facts are the controller's work. Inspect the environment directly, or use one bounded exploration only when that has clear value; do not ask the user to look up repository state. Decisions belong to the user. For unresolved decisions:

1. Model which decisions depend on others.
2. Ask one round containing only the current frontier: the load-bearing decisions whose prerequisites are settled. Ask no more than three questions in a round; choose those that collapse the most downstream ambiguity.
3. For every question, name bounded options and immediately state one recommended option with its reason and main tradeoff. A round is malformed if any question lacks its own recommendation; do not merely ask what the user prefers.
4. After the answer, recompute the frontier. If the frontier is broad, narrow the objective before asking dozens of questions.

Product intent, delivery boundary, constraints, scale, and acceptance are upstream of every implementation choice that depends on them. Put implementation choices in a later frontier.

Use a compact, answerable shape for each item:

```text
Q1. <decision and bounded options>
Recommended: <one option>, because <reason>; tradeoff: <main cost>.
```

Before sending a round, verify three things. No question names a language, framework, provider, protocol, queue, database, or architecture style while product intent, delivery boundary, scale, or acceptance is still open in this same round. For each remaining pair, ask whether either answer would change the other's option set; if so, drop the downstream question and keep the upstream one. The question count equals the recommendation count, and every recommendation carries both a reason and a tradeoff.

A choice may become an assumption only after a round actually asked it and the user could not decide; never pre-record a technology or design the interview has not reached. Then either expose the proposed choice in top-level `assumptions` for DAG approval or create a bounded `investigation` node whose accepted output unblocks the affected nodes. Do not keep rephrasing a question that discussion cannot settle.

Persist questions, answers, source facts, and resulting decisions in `.dag/sources/<dag-id>-clarifications.md` as rounds complete, and add it to `source_refs`. While product intent, delivery boundary, scale, or acceptance is open, neither the round nor this record proposes a stack: no recommended language, framework, queue, database, or email provider, and no node breakdown that presumes one. Keep proposed assumptions and still-open questions visibly separate from user decisions; chat history is never the only record of a completed round.

Exit clarification only when no unresolved choice can still change the DAG's objective, scope, acceptance, interfaces, dependency edges, or conflict edges. Every remaining unknown must be either an explicit assumption awaiting approval or the output of a bounded investigation node. Clarification establishes planning inputs; it does not approve the DAG or authorize implementation.

## 3. Decompose

Apply `${CLAUDE_SKILL_DIR}/references/decomposition.md`.

Each node has one goal, one hard scope, independently reviewable outputs, and acceptance criteria that a fresh worker can satisfy without chat history. It normally corresponds to one logical commit and one fresh review gate. Fold setup, configuration, documentation, and tests into the node whose deliverable needs them.

Do not create tiny bookkeeping nodes such as changing one import, renaming one symbol, running a test, or committing. Do not create subsystem-sized nodes that need new architecture while being implemented.

## 4. Analyze two graphs

Derive semantic dependencies first:

- Add `B.depends_on += A` only when B cannot be correctly started until A's output or decision exists.
- Different files do not remove a semantic dependency.

Then derive resource conflicts:

- Add symmetric `conflicts_with` entries when nodes could start independently but cannot safely be active together.
- Same-file work normally conflicts, but a conflict is not a fabricated dependency.
- Include shared generated artifacts, migrations, schemas, lockfiles, branch-wide rewrites, and external mutable resources when relevant.

Use `work_type` to expose investigation or integration work to the scheduler. Do not encode waves as persistent state; ready sets are derived.

## 5. Write fresh-context node contracts

For every node, provide objective, hard scope, `read_first`, inputs, exact upstream outputs, expected outputs, acceptance criteria, executable or inspectable verification, stop conditions through the shared worker protocol, and an initially empty handoff.

`read_first` names the files, not the directories that contain them, and it must be sufficient on its own: it is the whole context the worker gets, and a worker told to read `src/auth/` will read the repository instead. Nothing at execution time can cap that reading, so a node that does not fit in one context here cannot be executed at all: approval refuses more than 120,000 characters of `read_first`, or a `scope.files` expanding past 200 files or 20,000 lines. `validate-dag` reports both before you reach approval, and a node over either limit is split, not argued down.

`scope.files` says where the node writes. It is not a reading budget and must not be widened to give a worker room to look around; a file the node must read is named in `read_first`.

Acceptance belongs to the plan, not the worker or reviewer. Criteria must describe observable behavior or artifacts, not implementation activity. Every acceptance ID must be covered by at least one verification entry. A name search, file-existence check, import, or syntax check may support structural acceptance, but it cannot by itself verify behavioral acceptance.

## 6. Validate deterministically

Write the draft to `.dag/draft.json`, then let the runtime install it:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --init .dag/draft.json
"${CLAUDE_SKILL_DIR}/scripts/status"
```

`--init` validates the draft, installs it as `.dag/dag.json`, and consumes the draft. Never write `.dag/dag.json` directly: the runtime is what guarantees a new plan cannot overwrite an existing one.

If `--init` reports that a control file already exists, stop. A completed DAG is retired with `update-task --archive`, which you may do yourself. An unfinished one is different: report what it is and what remains, then ask whether to resume it or drop it, and end the turn. Abandoning half-finished work is the user's call, so the runtime also requires their persisted instruction:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --abandon --reason "<why>" --reference "<path under .dag/sources/>"
```

Deciding for yourself that a new request supersedes an existing plan is never a reason to abandon it.

Fix every validation error before presenting the plan. The model's own review does not replace the validator.

Check convergence before approval: map every original requirement and global acceptance criterion to one or more nodes, and inspect for missing integration work. Do not claim coverage merely because all planned nodes have titles.

Audit the DAG against the evidence rule before presenting it. Walk every technology name and concrete interface, payload, status code, storage, or security choice in `.dag/sources/`, the objective, node outputs, and acceptance, and resolve each one to its evidence, an `assumptions` entry, or an investigation output.

## 7. Review the decomposition

A plan that validates can still be a bad plan. Before showing it to the user, dispatch one fresh reviewer using the protocol in `${CLAUDE_SKILL_DIR}/references/decomposition.md`. Give it the plan, its source references, and the repository; never your own reasoning about why you split it this way. Write the plan out rather than transcribing it:

```bash
"${CLAUDE_SKILL_DIR}/scripts/status" --plan-view > .dag/artifacts/decomposition-review/plan.json
```

That is every node's contract without the record of running them, which is what this review judges. Rebuilding the same view by hand costs the whole plan read and written again for each round. Save the reviewer's JSON verdict under `.dag/artifacts/`, then record it:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --decomposition-review ".dag/artifacts/decomposition-review.json"
```

The runtime derives the verdict from the findings and binds it to this exact plan. Resolve every `critical` and `important` finding by writing a corrected draft and amending the plan in place:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --amend ".dag/draft.json"
```

Amending clears the previous verdict, so dispatch a fresh review afterwards. Give each amendment its own draft path; the file is kept as the record of what that round proposed. Fixing findings is an amendment, never an abandon-and-recreate: abandoning is for dropping an effort the user no longer wants, and using it to edit a plan destroys the record of what was reviewed. Do not argue a blocking finding away in chat: either change the plan or record why the reviewer was wrong and get a fresh verdict.

## 8. Approval gate

For a newly generated DAG from a short, natural-language, vague, or unapproved source, show a compact summary containing objective, nodes, dependency edges, conflicts, expected parallel frontier, global acceptance, and every entry in `assumptions`. Then stop and wait for explicit approval. Default `max_parallel` to 1 unless reconnaissance establishes safe isolation and meaningful wall-clock benefit; never set it to 3 merely because three is allowed.

Validation success is not product approval. After explicit approval, the controller records it through:

```bash
"${CLAUDE_SKILL_DIR}/scripts/update-task" --planning-status approved --reference "<durable approval reference>"
```

For an approval given in chat, first persist the exact approval and the DAG version it accepts under `.dag/sources/`, then pass that project-relative path. The runtime rejects a missing local approval artifact.

Recording approval ends the planning turn. Do not route to execution from here: report that the DAG is approved and ask the user to start a new session and run `/dag-engineering continue`. Execution belongs to a session that never held the planning context, and the runtime refuses to start a node from the session stamped in `planning_session`. Compacting this session is not a substitute, because it keeps the same session and carries planning residue forward.
