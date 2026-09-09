# Decomposition

## Start with observable outcomes

Trace backward from the original objective and global acceptance. Identify the artifacts, interfaces, behaviors, migrations, and integration checks that must exist, then group work into nodes that can be accepted separately without losing the end-to-end path.

Do not decompose while an unresolved choice could change the objective, scope, acceptance, interface ownership, dependency edges, or conflict edges. Return to planning clarification instead of hiding the choice in a worker contract.

## Node sizing

The following rule is adapted from Superpowers `writing-plans` with `task` changed to `node`:

> A node is the smallest unit that carries its own test cycle and is worth a
> fresh reviewer's gate. When drawing node boundaries: fold setup,
> configuration, scaffolding, and documentation steps into the node whose
> deliverable needs them; split only where a reviewer could meaningfully reject
> one node while approving its neighbor. Each node ends with an independently
> testable deliverable.

For this DAG, a good node also has:

- one clear objective and hard scope;
- explicit inputs and outputs;
- enough context for a fresh worker;
- acceptance based on observable behavior or artifacts;
- normally one logical commit;
- no unresolved architecture choice hidden inside implementation.

Combine mechanically identical, same-shaped edits on one review surface. Split outcomes that could receive different verdicts, interfaces that must be accepted before consumers start, or failures that should not invalidate neighboring work. Do not create bookkeeping nodes for one import, symbol rename, test run, or commit, or subsystem-sized nodes that require a new architecture while being implemented.

## Dependency pass

Ask for each pair:

1. Does one consume an interface, decision, schema, artifact, or verified behavior produced by the other?
2. Would starting the consumer first make it guess?
3. Would upstream failure make downstream work invalid rather than merely harder to merge?

If yes, add `depends_on` before considering file overlap. Different files do not remove a semantic dependency.

## Conflict pass

Ask separately:

1. Could both nodes be designed correctly at the same time?
2. Would concurrent edits contend on a file, generated artifact, migration sequence, lockfile, shared worktree state, or external mutable resource?
3. Can isolation remove the contention without creating a later semantic choice?

Use symmetric `conflicts_with` when concurrent activity is unsafe but neither node consumes the other. Same-file work usually conflicts; different files can still depend semantically.

## Independently executable contracts

Use exact project-relative paths, interface names, durable upstream handoff references, and real verification commands. A technology or interface choice that sources do not establish belongs in `assumptions` or investigation output, never in a node contract as settled fact.

Avoid:

- “implement as appropriate,” “handle edge cases,” or “tests above”;
- references to current chat or implicit prior reasoning;
- acceptance that only restates implementation activity;
- behavioral acceptance verified only by syntax, import, file existence, or keyword search;
- a node whose first action must be broad architecture discovery;
- fake dependencies added only to force a preferred schedule.

When uncertainty is itself work, create a bounded `investigation` node with a decision artifact, explicit questions, a read budget or stop condition, and downstream nodes dependent on its accepted output.

## Decomposition review

Split the fresh review into three narrow lanes dispatched in one batch and in parallel:

- `requirements`: objective, global acceptance, source references, assumptions, and coverage;
- `graph`: dependency, conflict, and resource shape;
- `execution`: scope, context, acceptance, and verification executability.

Each lane receives only its relevant contract and repository evidence, never the planner's reasoning. For one plan fingerprint and review request, dispatch each lane once. The controller waits for all requested lanes, then atomically records one complete manifest. A repair review includes only affected lanes and does not add a pre-approval full-plan review.

The validator already proves graph structure: no cycles, symmetric conflicts, undeclared same-file collisions between concurrent nodes, and acceptance coverage by verification. Review what scripts cannot decide:

- **missing dependency**: a node needs another node's output, interface, decision, or verified behavior but has no edge;
- **fabricated dependency**: an edge encodes scheduling preference rather than consumption;
- **granularity**: the node needs broad architecture discovery first or has no independent review value;
- **acceptance**: criteria restate implementation activity or behavioral criteria have only structural verification;
- **coverage**: a source requirement, global acceptance criterion, or integration step has no delivering node;
- **context**: `read_first` omits a file required by acceptance or names a directory instead of files;
- **scope realism**: `scope.files` does not match where behavior lives;
- **assumption laundering**: a decision was parked in `assumptions` instead of clarified.

Every finding names the offending nodes and the failure, not a style preference. Each lane writes a bound artifact whose `lane` matches the manifest and whose findings have stable IDs:

```json
{
  "dag_id": "auth-migration",
  "plan_fingerprint": "<current plan fingerprint>",
  "mode": "full",
  "lane": "graph",
  "findings": [
    {
      "id": "graph-001",
      "severity": "important",
      "kind": "missing_dependency",
      "nodes": ["auth-client"],
      "detail": "auth-client consumes the token shape that define-auth-contract produces, but declares no dependency",
      "suggestion": "add auth-client.depends_on = [define-auth-contract]"
    }
  ],
  "checks_performed": ["dependency pass", "conflict pass", "node sizing"]
}
```

The runtime derives the verdict; a reviewer does not pass its own approval flag. `critical` and `important` findings block approval, while `minor` findings are recorded.

## Scoped review revision

When the controller reorganizes a live plan, use scoped review rather than full decomposition review. It answers only whether the change breaks downstream work.

Give it the changed neighborhood, never the whole DAG or repository:

- nodes dropped, added, or retouched, before and after;
- the execution fact that caused the change;
- unfinished contracts that transitively depend on changed nodes;
- nothing else.

The validator reruns structural checks. Scoped review judges the prose interface: whether downstream nodes still receive what their `inputs` and `outputs` promise. Findings use the same severities; `critical` and `important` block the revision.

## Coverage convergence preview

Before approval, map every source requirement and global acceptance criterion to at least one node and verification path. Check cross-node integration explicitly; a populated task list is not evidence of coverage.
