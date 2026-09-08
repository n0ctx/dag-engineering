# Scheduling and integration

## Candidate derivation

`ready-tasks` returns eligible candidates, not a dispatch order. A node is eligible when it is pending, every semantic dependency is done, and no conflicting node is running.

The controller then decides whether dispatch has positive value. Consider:

- critical-path delay and real wall-clock benefit;
- repeated reconnaissance or duplicated context cost;
- implicit shared state not captured by file paths;
- `work_type` and `estimated_cost`;
- stable scope and acceptance;
- worktree setup and later integration cost.

`max_parallel` is a ceiling. The global ceiling is three. One is normal; two require clear independence; three require stable, highly independent work with obvious benefit. Several open-ended investigations are a reason to serialize, not a reason to fill slots.

## Dependency and failure behavior

`depends_on` is semantic. A failed or blocked upstream node keeps dependents ineligible. Do not bypass it by manually marking downstream work running.

When a node fails or blocks:

1. Stop dispatching descendants.
2. Preserve the attempt and evidence reference.
3. Classify the response: add missing context, change capability, retry with a changed brief, re-slice, re-plan, or explicitly block.
4. Never repeat an identical prompt indefinitely.

Independent ready nodes may continue only when the failure cannot invalidate their contracts or integration base.

## Conflict behavior

`conflicts_with` is symmetric resource exclusion. It does not imply output consumption and does not choose an order. If two eligible nodes conflict, the controller selects one using critical-path and cost judgment; the other stays pending.

Treat shared migrations, schemas, generated indexes, lockfiles, bulk formatters, mutable test fixtures, and external environments as possible conflicts even when declared file globs differ.

## Worktrees

Use the current working tree for one node or strict serial execution when it is clean and project rules permit. Use one worktree and branch per concurrently active implementation node.

Before dispatch into a worktree:

- identify repository root, common Git directory, base commit, and branch;
- prefer an existing project worktree convention; for a project-local directory, verify Git ignores it;
- reproduce required environment files or links without copying secrets into Git;
- run a project-appropriate clean baseline check;
- tell the worker its exact directory, scope, and absolute artifact paths.

A linked worktree never contains its own `.dag/`. The control plane stays at the main worktree root, and the runtime refuses to start a node in a worktree that carries a second one.

A worker commits its node but does not merge, rebase the control branch, edit another node's worktree, or update the DAG.

## Integration

Integrate only nodes that passed worker, reviewer, and master gates. Follow semantic dependency and interface order, not completion time. Preserve one node approximately as one logical commit.

The controller may resolve conflicts that are purely mechanical and do not change behavior or contracts. If resolution requires choosing behavior, changing an interface, or writing new domain logic, send it back to the responsible worker or re-plan.

After integrating a related group, run a targeted integration check before releasing new downstream nodes whose contract depends on the integrated behavior.
