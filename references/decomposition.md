# Decomposition

## Start from observable outcomes

Trace backward from the original objective and global acceptance. Identify the artifacts, interfaces, behaviors, migrations, and integration checks that must exist. Then group work into nodes that can be accepted separately without losing the end-to-end path.

Do not decompose while an unresolved choice could still change the objective, scope, acceptance, interface ownership, dependency edges, or conflict edges. Return to the planning module's clarification stage instead of hiding that choice inside a worker contract.

## Node sizing

The following wording is copied exactly from Superpowers `writing-plans` because it defines the same review boundary:

> A task is the smallest unit that carries its own test cycle and is worth a
> fresh reviewer's gate. When drawing task boundaries: fold setup,
> configuration, scaffolding, and documentation steps into the task whose
> deliverable needs them; split only where a reviewer could meaningfully
> reject one task while approving its neighbor. Each task ends with an
> independently testable deliverable.

For this DAG, a good node also has:

- one clear objective and hard scope;
- explicit inputs and outputs;
- enough context for a fresh worker;
- acceptance based on observable behavior or artifacts;
- normally one logical commit;
- no unresolved architecture choice hidden inside implementation.

Combine mechanically identical, same-shaped edits when they share one review surface. Split when different outcomes could legitimately receive different verdicts, when an interface must be accepted before consumers start, or when failure should not invalidate neighboring work.

## Dependency pass

Ask for each pair of nodes:

1. Does one consume an interface, decision, schema, artifact, or verified behavior produced by the other?
2. Would starting the consumer first require it to guess?
3. Would failure upstream make downstream work invalid rather than merely hard to merge?

If yes, add `depends_on`. Do this before considering file overlap.

## Conflict pass

Ask separately:

1. Could both nodes be designed correctly at the same time?
2. Would concurrent edits contend on the same file, generated artifact, migration sequence, lockfile, shared worktree state, or external mutable resource?
3. Can isolation remove the contention without creating a later semantic choice?

Use symmetric `conflicts_with` when concurrent activity is unsafe but neither node semantically consumes the other. Same-file work usually conflicts. Different files can still depend semantically.

## Independently executable contracts

Use exact project-relative paths, interface names, durable upstream handoff references, and real verification commands. A technology or interface choice the source and repository do not establish belongs in `assumptions` or an investigation output, never inside a node contract as settled fact. Avoid:

- “implement as appropriate,” “handle edge cases,” or “tests for the above”;
- references to current chat or implicit prior reasoning;
- acceptance that only restates the implementation step;
- behavioral acceptance “verified” only by syntax, import, file existence, or keyword search;
- a node whose first action must be broad architecture discovery;
- fake dependencies added only to force a preferred schedule.

When uncertainty is itself the work, create a bounded `investigation` node with a decision artifact, explicit questions, a read budget or stop condition, and downstream nodes that depend on its accepted output.

## Decomposition review

A fresh reviewer that took no part in drafting judges the plan before it reaches the user. It receives the DAG, its `source_refs`, the objective and global acceptance, and the repository — never the planner's reasoning.

The validator already proves the graph is well formed: no cycles, symmetric conflicts, no undeclared same-file collision between concurrent nodes, every acceptance criterion covered by a verification entry. This review covers what no script can decide:

- **missing dependency**: a node cannot correctly start without another node's output, interface, decision, or verified behavior, yet no edge says so;
- **fabricated dependency**: an edge encoding a scheduling preference rather than real consumption;
- **granularity**: a node whose first act must be broad architecture discovery, or one so small it carries no independent review value;
- **acceptance**: criteria restating implementation activity, or behavioral criteria whose only verification is structural;
- **coverage**: a source requirement, global acceptance criterion, or integration step that no node delivers;
- **context**: `read_first` omits a file the node's acceptance plainly requires, or names a directory where it should name files, so a fresh worker has to survey the repository to find its own inputs. The runtime already refuses a node whose declared context cannot fit in one worker, so this finding is for the case the size check cannot see: a contract that is small because it is incomplete;
- **scope realism**: `scope.files` that do not match where the behavior actually lives;
- **assumption laundering**: a decision parked in `assumptions` that should have been asked during clarification.

Every finding names the offending nodes and what would go wrong, not a style preference:

```json
{
  "dag_id": "auth-migration",
  "findings": [
    {
      "severity": "important",
      "kind": "missing_dependency",
      "nodes": ["auth-client"],
      "detail": "auth-client consumes the token shape that define-auth-contract produces, but declares no dependency, so a fresh worker would have to invent it",
      "suggestion": "add auth-client.depends_on = [define-auth-contract]"
    }
  ],
  "checks_performed": ["dependency pass", "coverage against global acceptance", "node sizing"]
}
```

The runtime derives the verdict from the findings, so no reviewer passes its own approval flag. `critical` and `important` block approval; `minor` is recorded and does not.

## Scoped review of a revision

When the controller reorganises a live plan, a full decomposition review is the wrong instrument: the plan was already reviewed, and only part of it moved. The scoped review answers one question — does the change break something downstream of it?

It receives the change and its neighbourhood, never the whole DAG and never the repository:

- what moved: the nodes dropped, added, or retouched, with the before and after of each;
- why: the execution facts that motivated it, such as the failing handoff or the review verdict;
- the contracts of the unfinished nodes that transitively depend on the changed ones;
- nothing else. It does not re-judge nodes the change did not touch.

The validator has already re-run everything structural — cycles, unknown edges, conflict symmetry, undeclared scope collisions, acceptance coverage — so this review is only for what those cannot read: whether a downstream node still gets what it was written to consume. `inputs` and `outputs` are prose, so no script can compare them; that judgement is the entire reason this review exists.

Findings use the same shape and severities as a full decomposition review, and may name nodes the revision dropped. `critical` and `important` block the revision.

## Coverage and convergence preview

Before approval, map each source requirement and each global acceptance criterion to at least one node and verification path. Check cross-node integration explicitly. A populated task list is not evidence of full requirement coverage.
