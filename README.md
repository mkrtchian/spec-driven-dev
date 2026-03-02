# spec-driven-dev

Your plan, fresh agents, zero drift.

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Markdown only](https://img.shields.io/badge/zero_code-markdown_prompts_only-brightgreen.svg)](#whats-in-this-repo)
[![Claude Code](https://img.shields.io/badge/Claude_Code-plugin-blueviolet.svg)](https://claude.ai/claude-code)

A structured workflow for AI-assisted development — from discussion to reviewed, tested, standards-compliant code, through a version-controlled plan.

## Quick start

```bash
# In Claude Code
/plugin marketplace add mkrtchian/spec-driven-dev
/plugin install spec-driven-dev@mkrtchian
```

```bash
# 1. Discuss the feature, draft and review the plan
/write-plan

# 2. Review the plan yourself, adjust if needed

# 3. Execute the plan step by step
/clear
/implement-plan plans/2025-03-01_my-feature.md
```

The plugin system handles distribution of both skills and agent definitions. Manual installation is not supported — the skills reference agents via plugin-namespaced `subagent_type` which only resolves through the plugin system.

## The problem

AI coding assistants hit two walls on non-trivial changes:

1. **They don't know what to build.** The more autonomy you give them, the more they drift from your intent. Without a precise spec, you spend more time correcting than you save.
2. **Context degrades.** A single conversation that discusses requirements, writes code, runs tests, and reviews standards will do all of these poorly. The agent loses focus as context fills up, and large changes exceed what fits in one pass.

## The approach

Two skills, each orchestrating fresh agents with isolated context:

```
/write-plan
  You discuss the feature with the agent              ←  interactive
  Agent drafts a detailed plan                        ←  plans/YYYY-MM-DD_feature.md
  Fresh agent reviews plan for gaps                   ←  auto-corrects
  Fresh agent checks plan against standards           ←  auto-corrects
  Fresh agent breaks plan into fewest possible steps
  You review the plan                                 ←  human in the loop

/implement-plan plans/YYYY-MM-DD_feature.md
  For each step:
    Fresh agent implements (TDD, tests, lint)
    Fresh agent hardens: catches drift, fixes issues, commits
  Fresh agent enforces standards on full diff — fixes, verifies, commits
  Fresh agent reviews, fixes obvious issues, flags trade-offs
```

Each agent starts with a fresh context window, focused on a single concern. No attention pollution between phases.

## Design decisions

**Isolated passes.** A single agent asked to "implement this plan, follow TDD, and check coding standards" will do all three poorly. An agent that just spent 20 minutes implementing code is not in the right mindset to review standards — it's biased toward defending what it just wrote. Fresh context per concern — same principle as code review. For the detailed rationale (including the token cost dimension), see [workflow.md](docs/workflow.md#why-fresh-context-at-each-pass).

**Plans in git.** Your plan is a markdown file you commit, review in a PR, and share with your team. Standalone plan files have no shared state — two developers can plan and implement different features on different branches without interfering.

**No parallelism.** Sequential passes are simpler to reason about, debug, and review. Parallel execution saves time but adds coordination complexity that isn't worth it for single-feature work.

**Dynamic discovery over configuration.** Skills detect your project's test runner, linter, and standards by finding and reading `CLAUDE.md` and other relevant files. Nothing is hardcoded to a stack.

**Conditional TDD.** Business logic gets test-first. Glue code, wiring, and config changes don't. This matches how experienced developers actually work.

**Step hardening.** After each implementation step, a fresh agent verifies alignment with the plan and fixes emergent issues. Problems are caught early, not discovered at the end.

## What's in this repo

```
skills/
  write-plan/SKILL.md              # Orchestrator — discussion → plan → review → steps
  implement-plan/SKILL.md          # Orchestrator — step-by-step execution → standards → final review

agents/
  sdd-plan-reviewer.md             # Finds gaps, wrong assumptions, integration risks
  sdd-plan-standards.md            # Checks plan against project coding conventions
  sdd-step-breakdown.md            # Splits plan into fewest possible implementation steps
  sdd-implementer.md               # Implements a step (test-first by default), runs verification
  sdd-step-hardener.md             # Verifies each step against the plan, fixes issues, commits
  sdd-standards-enforcer.md        # Fixes coding standards violations, verifies, and commits
  sdd-final-reviewer.md            # Fixes obvious issues, flags trade-offs for the developer

docs/
  workflow.md                      # The full workflow explained
  evaluation.md                    # Comparison with GSD and Superpowers
```

Each agent is a custom agent definition distributed via the plugin. The orchestrator skills reference them by `subagent_type` — their prompt content is never loaded into the orchestrator's own context.

## Limitations

These are prompts, not code. There are no hard guarantees that the agent will:

- Run tests / lint / type checking after every commit
- Follow test first approach
- Review carefully

In practice, well-written prompts with Opus are followed 95%+ of the time. The step hardener catches most of the remaining 5%. For hard guarantees, use git pre-commit hooks (which agents trigger when they commit).

## Compared

Two major Claude Code frameworks — [GSD](https://github.com/gsd-build/get-shit-done), [Superpowers](https://github.com/obra/superpowers) — and spec-driven-dev were tested on the same feature in the same repo.

|                               | GSD                        | Superpowers         | spec-driven-dev    |
| ----------------------------- | -------------------------- | ------------------- | ------------------ |
| Execution time                | ~15 min                    | ~15 min             | ~22 min            |
| Context after execution       | 33%                        | 60%                 | 42%                |
| Implementation steps          | 3 (2 in parallel)          | 5                   | 2                  |
| Setup overhead                | ~7 min (reformatting plan) | None                | None               |
| Human-in-the-loop during exec | No (review at end)         | Yes (every 2 steps) | No (review at end) |

GSD is the leanest on context. spec-driven-dev finishes at 42% — headroom that funds the correction passes (step hardening, standards enforcement, final review) that catch drift. For the full analysis, see **[the evaluation](docs/evaluation.md)**.

## License

MIT
