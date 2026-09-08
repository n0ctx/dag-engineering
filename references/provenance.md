# Provenance and third-party notices

This skill contains small verbatim prompt excerpts from one MIT-licensed source. Other studied sources influenced concepts only or were adapted independently.

## Verbatim source

| Source | Commit | File | Copied section | Local destination | Mode | Reason |
|---|---|---|---|---|---|---|
| `obra/superpowers` | `b36e0829c6d0140e93cfef2ca599b1b07d4a7797` | `skills/writing-plans/SKILL.md` | `Task Right-Sizing`, lines 38–43 | `references/decomposition.md` | exact-copy | The review-boundary semantics are identical and the wording is evaluation-informed. |
| `obra/superpowers` | `b36e0829c6d0140e93cfef2ca599b1b07d4a7797` | `skills/subagent-driven-development/implementer-prompt.md` | `You Do Not Dispatch Subagents`, lines 50–60 | `references/node-contract.md` | exact-copy | The controller/reviewer separation is identical. |
| `obra/superpowers` | `b36e0829c6d0140e93cfef2ca599b1b07d4a7797` | `skills/subagent-driven-development/task-reviewer-prompt.md` | `Do Not Trust the Report`, lines 64–71 | `references/review-protocol.md` | exact-copy | The anti-anchoring rule is identical. |

Everything else in this skill is adapted or independently implemented. In particular, Task Master was concept-only because its license adds the Commons Clause; no Task Master code or prompt was copied.

## Superpowers license

MIT License

Copyright (c) 2025 Jesse Vincent

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

## Studied sources not copied verbatim

| Source | Commit | License | Use |
|---|---|---|---|
| `mattpocock/skills` | `3cca18b368ae95cdbdebbff572ccafa662551015` | MIT | Adapted the decision-tree frontier and facts-versus-user-decisions split from `skills/productivity/grilling/SKILL.md`; kept clarification bounded, persistent, and DAG-specific rather than copying the stateless interview. |
| `obra/superpowers` | `b36e0829c6d0140e93cfef2ca599b1b07d4a7797` | MIT | Adapted load-bearing clarification, option comparison, and approval separation from `skills/brainstorming/SKILL.md`; no additional wording copied verbatim. |
| `github/spec-kit` | `4a7341a93d944d6efe153b71da4a1adb9c2b578c` | MIT | Adapted executable task structure, semantic ordering, same-file conflict, and final gap/convergence review. |
| `open-gsd/gsd-core` | `0ebc3cf27925a1ad8a183782a360fb1478ba7d2d` | MIT | Adapted dependency graph validation, blocker propagation, wave/frontier thinking, and fail-closed orchestration. |
| `gsd-build/get-shit-done` | `bdcaab2c752d9a33a1a1ca9acf3a3c81fb991815` | MIT | Adapted fresh executor, compact state/context persistence, and controller-routes-not-executes discipline. |
| `Yeachan-Heo/oh-my-claudecode` | `4820f5641828cb980b7eb488a3c187f3d01459c3` | MIT | Adapted role separation, dependency lifecycle, and independent verification. |
| `ZinkLu/Orca-Orchestration` | `0d6c334f5cf448e2ff431d877bca646cbf68d043` | MIT | Concept-only PRD-to-spec-to-DAG shape, self-contained node I/O, and bounded failure handling. |
| `eyaltoledano/claude-task-master` | `c0c98d367c55296bfe69e65680625b6db437af02` | MIT plus Commons Clause | Concept-only status, dependency, test strategy, and next-task selection; no copied material. |
| `bmad-code-org/BMAD-METHOD` | `abe4eb1bce919c9d22cd18b3519353d5824c4b75` | MIT plus trademark notice | Concept-only independently executable stories and persistent evidence; framework and branding rejected. |
| `OneWave-AI/claude-skills` | `82859c0ebaff803889be6ca2efa0834ba8787773` | MIT | Concept-only schema validation; generic workflow DSL rejected. |
