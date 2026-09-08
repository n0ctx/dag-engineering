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

## Coverage and convergence preview

Before approval, map each source requirement and each global acceptance criterion to at least one node and verification path. Check cross-node integration explicitly. A populated task list is not evidence of full requirement coverage.
