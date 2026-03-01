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
/implement-plan plans/2025-03-01_my-feature.md
```

## The problem

AI coding assistants hit two walls on non-trivial changes:

1. **They don't know what to build.** The more autonomy you give them, the more they drift from your intent. Without a precise spec, you spend more time correcting than you save.
2. **Context degrades.** A single conversation that discusses requirements, writes code, runs tests, and reviews standards will do all of these poorly. The agent loses focus as context fills up, and large changes exceed what fits in one pass.

Frameworks like [GSD](https://github.com/gsd-build/get-shit-done) and [Superpowers](https://github.com/obra/superpowers) address this with multi-agent orchestration. GSD excels at large multi-phase projects with parallel execution and state tracking. Superpowers brings strict quality controls with mandatory TDD and evidence-based verification. But both introduce significant complexity and impose their own workflow.

## The approach

Two skills, each orchestrating fresh agents with isolated context:

```
/write-plan
  You discuss the feature with the agent              ←  interactive
  Agent drafts a detailed plan                        ←  plans/YYYY-MM-DD_feature.md
  Fresh agent reviews plan for gaps                   ←  auto-corrects
  Fresh agent checks plan against standards           ←  auto-corrects
  Fresh agent breaks plan into atomic steps
  You review the plan                                 ←  human in the loop

/implement-plan plans/YYYY-MM-DD_feature.md
  For each step:
    Fresh agent implements (TDD, tests, lint)
    Fresh agent hardens: catches drift, fixes issues, commits
  Fresh agent enforces standards on full diff — fixes, verifies, commits
  Fresh agent fixes obvious issues, flags trade-offs
```

Each agent starts with a fresh context window, focused on a single concern. No attention pollution between phases.

## Why isolated passes

A single agent asked to "implement this plan, follow TDD, and check coding standards" will do all three poorly. An agent that just spent 20 minutes implementing code is not in the right mindset to review standards — it's biased toward defending what it just wrote.

Fresh context per concern. Same principle as code review — the reviewer shouldn't be the author.

There's also a cost argument. In skill-based frameworks, the orchestrating agent loads skill prompts into its own context. That text is retransmitted at every turn. In Superpowers' own test (2 tasks, 7 subagents), the orchestrator consumed ~1.2M tokens — 87% of total cost — while each subagent used only ~25k. When agent prompts are resolved by the runtime instead, the orchestrator never sees them — only short sub-agent results come back, and the orchestrator stays lightweight throughout.

## Design decisions

**Plans in git, not in a `.planning/` directory.** Your plan is a markdown file you commit, review in a PR, and share with your team. A `.planning/` directory with shared state files (STATE.md, ROADMAP.md) creates merge conflicts when multiple developers work on different features simultaneously. Standalone plan files have no shared state — two developers can plan and implement different features on different branches without interfering.

**No parallelism.** Sequential passes are simpler to reason about, debug, and review. Parallel execution saves time but adds coordination complexity that isn't worth it for single-feature work.

**Dynamic discovery over configuration.** Skills detect your project's test runner, linter, and standards by reading `package.json`, `pyproject.toml`, `CLAUDE.md`, etc. Nothing is hardcoded to a stack.

**Conditional TDD.** Business logic gets test-first. Glue code, wiring, and config changes don't. This matches how experienced developers actually work.

**Step hardening.** After each implementation step, a fresh agent catches drift from the plan and emergent issues, and fixes them. Problems are caught early, not discovered at the end.

## Why not GSD or Superpowers?

Both are real tools that solve real problems. They use the same underlying mechanism as this repo: Claude Code's `Task()` tool to spawn subagents with markdown prompts.

**[GSD](https://github.com/gsd-build/get-shit-done)** is best for multi-phase projects: wave-based parallel execution, state tracking across sessions, gap closure loops. The tradeoff is a `.planning/` directory (gitignored, not PR-reviewable) and significant overhead for single-phase work. It's also structurally solo-only — the shared state files conflict when multiple developers work on different features.

**[Superpowers](https://github.com/obra/superpowers)** is best for strict quality: mandatory TDD, two-stage review, evidence-based verification, cross-platform support. The tradeoff is an opinionated workflow, growing complexity (10k+ lines, 14 skills), and context accumulation — skill prompts are loaded into the main agent's context and retransmitted at every turn, which increases token cost and degrades orchestrator quality on longer features.

**What I wanted** was different: automate my existing workflow without replacing it. My manual process — discuss the feature, write a plan, review it, implement step by step — already worked well. But it was tedious to repeat the same orchestration every time, and a single context window couldn't hold all concerns at once. I needed the workflow automated with fresh agents per concern, version-controlled plans (reviewable in PRs), dynamic discovery of project tools and standards (not hardcoded to any stack), and awareness of nested `CLAUDE.md` files. No hidden state, no new workflow to learn — just my workflow, with agents.

For a detailed comparison across 18 dimensions, see **[the full evaluation](docs/evaluation.md)**.

## What's in this repo

```
skills/
  write-plan/SKILL.md              # Orchestrator — discussion → plan → review → steps
  implement-plan/SKILL.md          # Orchestrator — step-by-step execution → standards → final review

agents/
  sdd-plan-reviewer.md             # Finds gaps, wrong assumptions, integration risks
  sdd-plan-standards.md            # Checks plan against project coding conventions
  sdd-step-breakdown.md            # Splits plan into atomic implementation steps
  sdd-implementer.md               # Implements a step (decides TDD approach), runs verification
  sdd-step-hardener.md             # Hardens each step — catches drift and emergent issues, fixes them, commits
  sdd-standards-enforcer.md        # Fixes coding standards violations, verifies, and commits
  sdd-final-reviewer.md            # Fixes obvious issues, flags trade-offs for the developer

docs/
  workflow.md                      # The full workflow explained
  evaluation.md                    # Comparison with GSD and Superpowers
```

Each agent is a custom agent definition distributed via the plugin. The orchestrator skills reference them by `subagent_type` — their prompt content is never loaded into the orchestrator's own context.

## Installation

```bash
# In Claude Code
/plugin marketplace add mkrtchian/spec-driven-dev
/plugin install spec-driven-dev@mkrtchian
```

The plugin system handles distribution of both skills and agent definitions. Manual installation is not supported — the skills reference agents via plugin-namespaced `subagent_type` which only resolves through the plugin system.

## Limitations

These are prompts, not code. There are no hard guarantees that the agent will:

- Run tests / lint / type checking after every commit
- Follow TDD strictly
- Review carefully

In practice, well-written prompts with Opus are followed 95%+ of the time. The step hardener catches most of the remaining 5% — a fresh agent that catches drift and emergent issues, and fixes them directly. For hard guarantees, use git pre-commit hooks (which agents trigger when they commit).

## License

MIT
