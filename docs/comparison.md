# Framework comparison: GSD vs Superpowers vs spec-driven-dev

Part of [spec-driven-dev](../README.md).

_Last updated: March 2026. These frameworks evolve fast — observations may not reflect their current versions._

All three frameworks use the same underlying mechanism: Claude Code's `Task()` tool to spawn subagents. Each subagent gets a fresh 200k token context window. The frameworks differ in how they structure prompts, manage state, and orchestrate these subagents — but there is no process isolation, no Docker containers, no separate CLI invocations. It's prompt engineering with structure.

## When to use what

**Use [GSD](https://github.com/gsd-build/get-shit-done)** if you're building a multi-phase project solo and want automated state tracking, parallel execution, and gap closure. Accept the `.planning/` overhead and the learning curve. In benchmarks, execution is fast (~15 min) with the lowest context usage (33%), but the reformatting overhead to enter GSD's format partially offsets the parallelization gains.

**Use [Superpowers](https://github.com/obra/superpowers)** if you want the strict quality controls, cross-platform support, and a mature, well-tested framework. Accept that it's opinionated about workflow and growing in complexity. In benchmarks, execution is fast (~15 min) with useful mid-execution checkpoints, but context usage is higher (60%) due to skill-based accumulation.

**Use spec-driven-dev** if you work in a team, want strict quality, and want simple isolated execution passes with no lock-in to any stack or workflow. Accept that it's minimal and you'll handle edge cases yourself. In benchmarks, execution is slower (~22 min) but finishes at 42% context — enough headroom for the many correction passes.

## Benchmark

All three frameworks were tested on the same feature in the same repository. The benchmark focuses on the **execution phase** — planning is interactive discussion and not meaningful to compare on speed.

|                               | GSD                         | Superpowers               | spec-driven-dev              |
| ----------------------------- | --------------------------- | ------------------------- | ---------------------------- |
| Execution time                | ~15 min                     | ~15 min                   | ~22 min                      |
| Context after execution       | 33%                         | 60%                       | 42%                          |
| Implementation steps          | 3 (2 in parallel)           | 5                         | 2                            |
| Review per step               | None (executor self-checks) | 2 agents (spec + quality) | 1 agent (step hardener)      |
| Post-implementation passes    | 1 (static verifier)         | 1 (final code review)     | 2 (standards + final review) |
| Setup overhead                | ~7 min (reformatting plan)  | None                      | None                         |
| Human-in-the-loop during exec | No (review at end)          | Yes (every 2 steps)       | No (review at end)           |

All three were tested with a fresh session for execution (context cleared between planning/reformatting and implementation).

**Observations:**

GSD and Superpowers are ~30% faster for implementation. GSD finishes with the lowest context (33%), thanks to wave-based parallel execution and file-based state passing. All three catch issues at different stages: Superpowers reviews each task with 2 dedicated agents (spec + quality) during implementation; spec-driven-dev hardens (plan drift + emerging issues) each step with a fresh agent aware of the full plan, then runs 2 dedicated passes (standards + review) after all implementation is done — reviewers start clean, with no bias from having watched the code being written.

Superpowers' human-in-the-loop during execution is a genuine trade-off: catching issues after every pair of steps vs reviewing at the end. The cost is that you must be present during execution rather than doing other work.

All three frameworks produced working implementations. The differences are in the correction and verification layer — how much context is left for it, and how it's structured.

---

## How context flows

The three frameworks use different patterns for passing information between agents. This has direct impact on token cost, orchestrator quality over time, and sub-agent freshness.

**GSD**: Agents communicate through files on disk. The orchestrator workflow passes file paths in `Task()` prompts — each agent reads its own inputs (PLAN.md, STATE.md, config.json) with a fresh 200k context window. Results are written to disk (SUMMARY.md, VERIFICATION.md), and the orchestrator only checks that output files exist. State transitions (advancing plan counters, updating progress) go through `gsd-tools.cjs` CLI calls, not through the orchestrator's context. The tradeoff is structured state management (STATE.md, config.json, phase directories).

**Superpowers**: Skills are loaded into the main agent's context via the Skill tool. The orchestrating agent carries the skill text in its own context window throughout the session. Sub-agent results also come back into this context, but the dominant cost is the skill text itself — retransmitted at every turn. Sub-agents get fresh context, but the orchestrator itself degrades as context fills and gets compressed.

**spec-driven-dev** (this repo): Agent prompts are custom agent definitions distributed via the plugin. The orchestrator skills reference them by `subagent_type` — the runtime resolves the agent definition and passes it to the sub-agent directly, so the orchestrator never sees their content. The orchestrator sees only short status messages ("STEP COMMITTED", "ISSUES FOUND", "STANDARDS COMPLIANT") and stays lightweight throughout. Sub-agents do redundant file reads, but this is cheaper than retransmitting a growing orchestrator context at every turn.

## Framework details

### GSD (Get Shit Done)

**What it is**: A full project lifecycle manager: requirements gathering, roadmap creation, phase planning, parallel execution, verification, gap closure. Large codebase (markdown prompts + JavaScript tooling).

**How it's built**: Three layers. **Commands** (`/gsd:*`) are markdown entry points that delegate to **workflows** — markdown files containing the actual orchestration logic with inline `Task()` calls and bash commands. Workflows spawn **agents** (gsd-executor, gsd-planner, gsd-verifier, etc.) via Claude Code's native `Task()` subagent mechanism — 11 specialized agent types in total. A JavaScript CLI (`gsd-tools.cjs`, ~60 subcommands) handles state management and discovery but never spawns agents — it returns structured JSON that workflows and agents consume to know what to read and what to do.

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
- Designed for multi-phase projects — may be more structure than needed for single-feature work

**In practice**: GSD requires reformatting existing plans into its own structure (~7 min overhead). Once there, execution is fast (~15 min, 33% context) with effective wave-based parallelism. The framework is designed for its own planning format — bringing an external plan skips the project-management features.

### Superpowers

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
- Context accumulation in same-session mode — in the benchmark, execution hit 95% context in ~10 minutes, requiring a restart (separate-session mode finished at 60%)
- Implementation plan is generated but not prompted for review before execution begins
- Subagents may still miss nested CLAUDE.md files depending on skill configuration

**In practice**: Same-session mode hit 95% context in ~10 minutes; separate-session mode finished at 60% context in ~15 min. Produced 5 implementation steps with human checkpoints every 2 steps — useful for catching issues early at the cost of requiring the developer to be present during execution.

### spec-driven-dev

**What it is**: ~800 lines of markdown across 2 orchestrator skills and 7 custom agent definitions. A full plan-first workflow: discussion → plan → review → step-by-step execution with step hardening.

**Strengths**:

- Full cycle: `/write-plan` (discussion → reviewed plan) and `/implement-plan` (step-by-step execution)
- Plans are version-controlled, PR-reviewable, team-compatible
- Each pass gets fresh context (no attention pollution between concerns)
- Step hardening per implementation step (fresh agent catches drift and emergent issues, and fixes them)
- Agent prompts are custom agent definitions, resolved by the runtime — never loaded into orchestrator context
- Dynamically discovers project verification commands and project coding standards
- Explicitly reads nested CLAUDE.md files
- No hidden state, no special directories

**Weaknesses**:

- Slowest of the three (~22 min vs ~15 min) — sequential passes with no parallelism, and agents loading the whole plan from fresh context
- Higher token cost per feature: each step spawns 2 agents (implementer + hardener), plus 2 post-implementation passes — a total of 6 agent invocations for a 2-step plan
- No state tracking across sessions — if execution fails mid-way, you restart manually (committed steps are in git)
- Claude Code only (no cross-platform support)
- Prompt-based, not enforced — same reliability characteristics as the others

**In practice** (tested on the same feature and repo as GSD and Superpowers):

The planning discussion (`/write-plan`) ended at 54% context. After clearing context and running `/implement-plan`, execution took ~22 minutes — the slowest of the three, due to sequential passes and full plan re-reading from fresh context agents. Finished at 42% context (between GSD's 33% and Superpowers' 60%). The plan was broken into 2 implementation steps, each spawning 2 agents (implementer + hardener), followed by 2 post-implementation passes.
