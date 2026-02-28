# Evaluation: GSD vs Superpowers vs spec-driven-dev

## How AI coding frameworks work

Both GSD and Superpowers use the same underlying mechanism: Claude Code's `Task()` tool to spawn subagents. Each subagent gets a fresh 200k token context window. The frameworks are markdown prompts that instruct Claude how to orchestrate these subagents.

There is no process isolation, no Docker containers, no separate CLI invocations. It's prompt engineering with structure.

## GSD (Get Shit Done)

**What it is**: ~33k lines of markdown + ~17k lines of JavaScript. A full project lifecycle manager: requirements gathering, roadmap creation, phase planning, parallel execution, verification, gap closure.

**Strengths**:

- Wave-based parallel execution (multiple agents implementing different plans simultaneously)
- State tracking across sessions (`.planning/` directory with structured state)
- Gap closure loop (verification fails → targeted replan → re-execute → re-verify)
- Deviation rules (4 levels of response to unexpected situations during execution)
- JavaScript tooling (`gsd-tools.cjs`) for deterministic state operations (phase management, milestone tracking)

**Weaknesses**:

- `.planning/` directory must be gitignored in team settings (concurrent work on different features causes conflicts) — not reviewable in PRs, not shareable
- Verifier does static analysis only (grep, file existence) — never runs tests, linter, or type-checker
- Subagents ignore nested CLAUDE.md files (only reads root CLAUDE.md)
- Heavy overhead to reformat an existing plan into GSD's format
- Complex for single-phase work (31 commands, 11 specialized agents)

**In practice** (tested on a real project with a team-compatible, plan-first workflow):

GSD is optimized for its own lifecycle: requirements → roadmap → phases. In a team setting, `.planning/` must be gitignored — concurrent work on different features by different developers causes conflicts on the shared state files. This means the planning layer is local-only and not shareable. To work on unrelated features across sessions, I had to write a shell script that resets `.planning/` with fake project/requirement files, bypassing the project-management layer entirely. The actual path was: write a plan in git → reset `.planning/` → inject the plan via `/gsd:plan-phase 1 --prd @path/to/plan.md` → wait for GSD to reformat it into its own structure → `/gsd:execute-phase 1`.

The reformatting step is not trivial — GSD breaks the plan into its own task format, generates wave assignments, and produces execution plans. This takes significant time, which partially offsets the parallelization gains during execution. The execution itself (wave-based parallel agents) works well, but the cost of getting there matters.

## Superpowers

> **Note**: Not tested yet — the analysis below is based on code reading. Will be updated after real usage.

**What it is**: ~11k lines of markdown across 14 skills. A skill-based workflow framework with strict quality controls. Cross-platform (Claude Code, Cursor, Codex, OpenCode).

**Strengths**:

- TDD "Iron Law" — no production code without a failing test
- Two-stage review per task (spec compliance + code quality)
- Evidence-based verification (must run command, must read output, must prove claims)
- Git worktree isolation (each feature gets its own filesystem copy)
- Plans stored in `docs/plans/` — committable, reviewable
- Subagent-driven development: dispatches a fresh agent per task with parallel support (`dispatching-parallel-agents` skill)
- Real test suite: unit tests for skill content, multi-turn invocation tests, and full E2E tests with real projects
- Cross-platform: works with Claude Code, Cursor, Codex, and OpenCode via platform-specific plugin manifests

**Weaknesses**:

- Opinionated about the planning phase (`/write-plan` guides creation with its own structure) — you can bring an existing plan to `/execute-plan`, but the workflow steers you toward its format
- "Iron Laws" are prompt text, not code — well-tested but not enforced by tooling
- Complexity has grown significantly (generated plans in `docs/plans/` run 8k-27k lines each)
- Subagents may still miss nested CLAUDE.md files depending on skill configuration

## spec-driven-dev

**What it is**: ~770 lines of markdown across 2 commands + 7 skills. A full plan-first workflow: discussion → plan → review → step-by-step execution with drift detection.

**Strengths**:

- Full cycle: `/write-plan` (discussion → reviewed plan) and `/implement-plan` (step-by-step execution)
- Plans are your own markdown files — version-controlled, PR-reviewable, team-compatible
- Each pass gets fresh context (no attention pollution between concerns)
- Drift detection per implementation step (fresh agent verifies output vs plan)
- Most skills work standalone (`/plan-review`) and all compose via the orchestrators
- Dynamically discovers project verification commands (not hardcoded to any stack)
- Dynamically discovers project coding standards (not hardcoded to any checklist)
- Explicitly reads nested CLAUDE.md files
- No hidden state, no special directories

**Weaknesses**:

- No parallelism (sequential passes)
- No state tracking across sessions
- Claude Code only (no cross-platform support)
- Prompt-based, not enforced — same reliability characteristics as the others

## The real comparison

|                                  | GSD                                       | Superpowers                                                                 | spec-driven-dev                      |
| -------------------------------- | ----------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------ |
| Plans in git (PR-reviewable)     | No (.planning/ must be gitignored in teams) | Yes (docs/plans/)                                                           | Yes (your choice)                    |
| Bring your own plan              | Partial (--prd flag)                      | Partial (execute-plan accepts plans, but workflow steers toward its format) | Yes (that's the point)               |
| Parallel execution               | Yes (wave-based)                          | Yes (per-task subagents + dispatching skill)                                | No                                   |
| Test execution during impl       | In subagent (not visible)                 | Evidence-based (must prove)                                                 | In subagent (explicit instruction)   |
| Drift detection per step         | No                                        | No                                                                          | Yes (fresh agent per step)           |
| Post-implementation verification | Static analysis (grep)                    | Evidence-based                                                              | Standards review + final review      |
| TDD                              | Opt-in (type: tdd)                        | Mandatory (Iron Law)                                                        | Conditional (logic=TDD, glue=no)     |
| Nested CLAUDE.md awareness       | No                                        | Partial                                                                     | Yes (explicit in prompts)            |
| Dynamic stack discovery          | No (project-specific)                     | Partial (some skills adapt)                                                 | Yes (verification + standards)       |
| Team-compatible                  | No (local state, conflicts in teams)      | Mostly (docs/plans/)                                                        | Yes (no special setup)               |
| Cross-platform                   | Claude Code, OpenCode, Gemini CLI, Codex  | Claude Code, Cursor, Codex, OpenCode                                        | Claude Code only                     |
| Complexity                       | High (~50k lines, md + JS)                | Medium-High (~11k lines)                                                   | Low (~770 lines)                     |
| Best for                         | Multi-phase projects with long lifecycles | Solo dev wanting strict quality + mature tooling                            | Team dev with existing plan workflow |

## When to use what

**Use GSD** if you're building a multi-phase project solo and want automated state tracking, parallel execution, and gap closure. Accept the `.planning/` overhead and the learning curve.

**Use Superpowers** if you want the strictest quality controls, cross-platform support, and a mature, well-tested framework. Accept that it's opinionated about workflow and growing in complexity.

**Use spec-driven-dev** if you write your own plans, work in a team, and want simple isolated execution passes with no lock-in to any stack or workflow. Accept that it's minimal and you'll handle edge cases yourself.
