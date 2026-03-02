# Evaluation: GSD vs Superpowers vs spec-driven-dev

## How these three AI coding frameworks work

All three frameworks use the same underlying mechanism: Claude Code's `Task()` tool to spawn subagents. Each subagent gets a fresh 200k token context window. The frameworks differ in how they structure prompts, manage state, and orchestrate these subagents — but there is no process isolation, no Docker containers, no separate CLI invocations. It's prompt engineering with structure.

## How context flows

The three frameworks use different patterns for passing information between agents. This has direct impact on token cost, orchestrator quality over time, and sub-agent freshness.

**GSD**: Agents communicate through files on disk. The orchestrator workflow passes file paths in `Task()` prompts — each agent reads its own inputs (PLAN.md, STATE.md, config.json) with a fresh 200k context window. Results are written to disk (SUMMARY.md, VERIFICATION.md), and the orchestrator only checks that output files exist. State transitions (advancing plan counters, updating progress) go through `gsd-tools.cjs` CLI calls, not through the orchestrator's context. The tradeoff is structured state management (STATE.md, config.json, phase directories).

**Superpowers**: Skills are loaded into the main agent's context via the Skill tool. The orchestrating agent carries the skill text in its own context window throughout the session. Sub-agent results also come back into this context, but the dominant cost is the skill text itself — retransmitted at every turn. In Superpowers' own test (2 tasks, 7 subagents), the main agent consumed ~1.2M tokens (87% of total cost) while each subagent used only ~25k. Sub-agents get fresh context, but the orchestrator itself degrades as context fills and gets compressed.

**spec-driven-dev** (this repo): Agent prompts are custom agent definitions distributed via the plugin. The orchestrator skills reference them by `subagent_type` — the runtime resolves the agent definition and passes it to the sub-agent directly, so the orchestrator never sees their content. The orchestrator sees only short status messages ("STEP COMMITTED", "ISSUES FOUND", "STANDARDS COMPLIANT") and stays lightweight throughout. Sub-agents do redundant file reads, but this is cheaper than retransmitting a growing orchestrator context at every turn.

## Real-world benchmark

All three frameworks were tested on the same feature in the same repository. The benchmark focuses on the **execution phase** — planning is interactive discussion and not meaningful to compare on speed.

|                               | GSD                        | Superpowers         | spec-driven-dev    |
| ----------------------------- | -------------------------- | ------------------- | ------------------ |
| Execution time                | ~15 min                    | ~15 min             | ~22 min            |
| Context after execution       | 33%                        | 60%                 | 42%                |
| Implementation steps          | 3 (2 in parallel)          | 5                   | 2                  |
| Setup overhead                | ~7 min (reformatting plan) | None                | None               |
| Human-in-the-loop during exec | No (review at end)         | Yes (every 2 steps) | No (review at end) |

All three were tested with a fresh session for execution (context cleared between planning/reformatting and implementation).

**Observations:**

GSD and Superpowers are ~30% faster for implementation. GSD finishes with the lowest context (33%), thanks to wave-based parallel execution and file-based state passing. spec-driven-dev finishes at 42% — the headroom over Superpowers (60%) funds the correction passes (step hardening, standards enforcement, final review) that run after implementation with a fresh, uncrowded context window. More steps (Superpowers' 5 vs spec-driven-dev's 2) means more per-step overhead and less budget for these targeted correction passes.

Superpowers' human-in-the-loop during execution is a genuine trade-off: catching issues after every pair of steps vs reviewing at the end. The cost is that you must be present during execution rather than doing other work.

All three frameworks produced working implementations. The differences are in the correction and verification layer — how much context is left for it, and how it's structured.

## When to use what

**Use GSD** if you're building a multi-phase project solo and want automated state tracking, parallel execution, and gap closure. Accept the `.planning/` overhead and the learning curve. In benchmarks, execution is fast (~15 min) with the lowest context usage (33%), but the reformatting overhead to enter GSD's format partially offsets the parallelization gains.

**Use Superpowers** if you want the strict quality controls, cross-platform support, and a mature, well-tested framework. Accept that it's opinionated about workflow and growing in complexity. In benchmarks, execution is fast (~15 min) with useful mid-execution checkpoints, but context usage is higher (60%) due to skill-based accumulation.

**Use spec-driven-dev** if you work in a team, want strict quality, and want simple isolated execution passes with no lock-in to any stack or workflow. Accept that it's minimal and you'll handle edge cases yourself. In benchmarks, execution is slower (~22 min) but finishes at 42% context — enough headroom for the many correction passes.

---

## Framework details

### GSD (Get Shit Done)

**What it is**: ~33k lines of markdown + ~17k lines of JavaScript. A full project lifecycle manager: requirements gathering, roadmap creation, phase planning, parallel execution, verification, gap closure.

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
- Complex for single-phase work (31 commands, 11 specialized agents)

**In practice** (tested on the same feature and repo as Superpowers and spec-driven-dev):

GSD is optimized for its own lifecycle: requirements → roadmap → phases. To work on unrelated features across sessions in a team setting, I had to write a shell script that resets `.planning/` with fake project/requirement files, bypassing the project-management layer entirely. The actual path was: write a plan in git → reset `.planning/` → inject the plan via `/gsd:plan-phase 1 --prd @path/to/plan.md` → wait for GSD to reformat it into its own structure → `/gsd:execute-phase 1`.

The reformatting step took ~7 minutes just to initialize GSD's own plan from the existing implementation plan. Execution then took ~15 minutes, finishing at 33% context (after clearing between reformatting and execution). The wave-based parallel execution works well — GSD ends up with the lowest context usage of the three — but the reformatting overhead is unavoidable when bringing an existing plan.

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
- Context accumulation in same-session mode — execution hit 95% context in ~10 minutes, forcing a restart (separate-session mode finished at 60%; see [benchmark](#real-world-benchmark) and [details](#superpowers))
- Implementation plan is generated but not prompted for review — a gap in the human-in-the-loop story
- Subagents may still miss nested CLAUDE.md files depending on skill configuration

**In practice** (tested on the same feature and repo as GSD and spec-driven-dev):

The planning phase produces a short design document (~50 lines), reviewed in the terminal in several chunks — this felt well-paced for human review. An implementation plan is then generated on top of it, but there's no prompt to review it before execution starts, which is a gap given the framework's emphasis on human oversight.

Two execution modes are available: continue in the same session or start a separate session. In same-session mode, context hit 95% after ~10 minutes of implementation, forcing a restart via a memory file saved by the agent. In separate-session mode (fresh context for execution), implementation took ~15 minutes and finished at 60% context, with human-in-the-loop checkpoints after every pair of steps. This is a genuine trade-off: catching issues earlier requires being present during execution rather than reviewing at the end.

The framework produced 5 implementation steps — more granular than spec-driven-dev's 2 steps. More steps mean more human checkpoints but also more agent overhead per step, leaving less context budget for targeted correction passes like standards enforcement or drift detection.

### spec-driven-dev

**What it is**: ~800 lines of markdown across 2 orchestrator skills and 7 custom agent definitions. A full plan-first workflow: discussion → plan → review → step-by-step execution with step hardening.

**Strengths**:

- Full cycle: `/write-plan` (discussion → reviewed plan) and `/implement-plan` (step-by-step execution)
- Plans are version-controlled, PR-reviewable, team-compatible
- Each pass gets fresh context (no attention pollution between concerns)
- Step hardening per implementation step (fresh agent catches drift and emergent issues, fixes them)
- Agent prompts are custom agent definitions, resolved by the runtime — never loaded into orchestrator context
- Dynamically discovers project verification commands (not hardcoded to any stack)
- Dynamically discovers project coding standards (not hardcoded to any checklist)
- Explicitly reads nested CLAUDE.md files
- No hidden state, no special directories

**Weaknesses**:

- No parallelism (sequential passes)
- No state tracking across sessions
- Claude Code only (no cross-platform support)
- Prompt-based, not enforced — same reliability characteristics as the others

**In practice** (tested on the same feature and repo as GSD and Superpowers):

The planning discussion (`/write-plan`) ended at 54% context. After clearing context and running `/implement-plan`, execution took ~22 minutes and finished at 42% context — the lowest of the three frameworks. The plan was broken into only 2 implementation steps, each larger but leaving substantial headroom for the correction passes (step hardening, standards enforcement, final review) that run after implementation. In my manual workflow I would have launched these checks one by one anyway — the framework just automates the sequencing.
