# dag-engineering

An Agent Skill for large, multi-step code engineering: decompose the work into a DAG of subtasks and execute them with subagents in a dedicated Git worktree.

## Scope

Use this Skill for a PRD, roadmap, issue, large refactor, or other work that benefits from decomposition into dependent subtasks.

It is intentionally not for small single-session changes, quick fixes, isolated file edits, or ordinary non-code work.

## Workflow

1. Open one worktree and branch for the effort.
2. Decompose the work into nodes with explicit dependencies, conflicts, scope, steps, acceptance, and verification.
3. Save the plan as Markdown under `.dag/` in the main worktree.
4. Dispatch ready nodes to subagents, in parallel when they are independent.
5. Each subagent commits its node; the main agent reviews the commit diff and fixes problems itself in a follow-up commit.
6. One subagent reviews and fixes the whole worktree diff.
7. The main agent performs final acceptance and reports.

See [SKILL.md](SKILL.md) for details.

## License

MIT. See [LICENSE](LICENSE).
