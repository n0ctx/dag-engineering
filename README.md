# dag-engineering

An Agent Skill for large, multi-step code engineering: decompose the work into a DAG of subtasks and execute them in parallel with subagents, each node in its own Git worktree, continuously integrated by the main agent.

## Scope

Use this Skill for a PRD, roadmap, issue, large refactor, or other work that benefits from decomposition into dependent subtasks.

It is intentionally not for small single-session changes, quick fixes, isolated file edits, or ordinary non-code work.

## Workflow

1. Open one integration worktree and branch (`dag/<plan-id>`) for the effort; only the main agent writes to it.
2. Investigate the code: the main agent locates the relevant code and, when there are several independent questions, dispatches read-only subagents to answer them in parallel. Fix the cross-node design (interfaces, reuse, ownership) before splitting.
3. Decompose the work into nodes with explicit dependencies, real conflicts, scope, steps, acceptance, and verification; before dispatch, check the plan for gaps that would leave the goal unmet and for wasted work (an independent read-only review when it has more than five nodes). Ordinary overlap in the same file is left to Git merges rather than serialized.
4. Save the plan as Markdown under `.dag/` in the main worktree; it records node status (`pending` / `running` / `submitted` / `done` / `blocked`), worktree, branch, and commit.
5. Dispatch every ready node at once, each in its own worktree and branch (`dag-node/<plan-id>/<node-id>`) created from the latest accepted integration commit. No fixed concurrency limit and no batch waiting.
6. As each node is submitted, the main agent merges it into the integration branch with `--no-commit`, verifies, then commits and immediately dispatches newly unblocked nodes. Failures are aborted without touching accepted history and routed by ownership: node defects go back to the node's subagent, plan errors are fixed locally. Integrated node worktrees are removed safely.
7. One read-only subagent reviews the full integration diff; fixes are routed by ownership.
8. The main agent performs final acceptance and reports. Merging into the target branch and removing the integration worktree happen only on the user's request.

Interrupted work resumes from the plan, reconciled against the actual Git state.

See [SKILL.md](SKILL.md) for details.

## License

MIT. See [LICENSE](LICENSE).
