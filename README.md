# spec-driven-dev

A structured workflow for AI-assisted development - from requirements discussion to verified implementation, through a version-controlled plan.

## The problem

AI coding assistants hit two walls on non-trivial changes:

1. **They don't know what to build.** The more autonomy you give them, the more they drift from your intent. Without a precise spec, you spend more time correcting than you save.
2. **Context degrades.** A single conversation that discusses requirements, writes code, runs tests, and reviews standards will do all of these poorly. The agent loses focus as context fills up, and large changes exceed what fits in one pass.

Frameworks like [GSD](https://github.com/gsd-build/get-shit-done) and [Superpowers](https://github.com/obra/superpowers) address this with multi-agent orchestration. GSD excels at large multi-phase projects with parallel execution and state tracking. Superpowers brings strict quality controls with mandatory TDD and evidence-based verification. But both introduce significant complexity and impose their own workflow.

## The approach

My workflow already works well: discuss the feature with an AI agent, have it draft a detailed plan, review and iterate on that plan, then implement. This repo automates the implementation phase — the part where fresh, isolated agents each handle one concern with a clean context.

```
You (human)                          AI Agents
─────────────                        ─────────────────────────────────────────
Discuss feature with AI        ───>  Agent explores codebase, clarifies requirements
Have the agent write a plan    ───>  Agent drafts a detailed implementation plan
Review and iterate             ───>  You refine until the plan is precise
Version the plan in git        ───>  PR-reviewable by your team

/implement-plan plan.md
                               ───>  Pass 1: Plan reviewer (fresh context) — finds gaps
                               ───>  Pass 2: Implementer (fresh context) — TDD, atomic commits
                               ───>  Pass 3: Standards reviewer (fresh context) — checks diff
```

Each pass starts with a fresh context window, focused on a single concern. No attention pollution between phases.

## Why isolated passes

A single agent asked to "implement this plan, follow TDD, and check coding standards" will do all three poorly. An agent that just spent 20 minutes implementing code is not in the right mindset to review standards — it's biased toward defending what it just wrote.

Fresh context per concern. Same principle as code review — the reviewer shouldn't be the author.

## Why not GSD or Superpowers?

Both are real tools that solve real problems. They use the same underlying mechanism as this repo: Claude Code's `Task()` tool to spawn subagents with markdown prompts.

**[GSD](https://github.com/gsd-build/get-shit-done)** is best for multi-phase projects: wave-based parallel execution, state tracking across sessions, gap closure loops. The tradeoff is a `.planning/` directory (gitignored, not PR-reviewable) and significant overhead for single-phase work.

**[Superpowers](https://github.com/obra/superpowers)** is best for strict quality: mandatory TDD, two-stage review, evidence-based verification, cross-platform support. The tradeoff is an opinionated workflow and growing complexity (10k+ lines, 14 skills).

**What I wanted** was different: automate my existing workflow without replacing it. I already discuss, plan, review, and implement in a way that is simple and that works. I needed version-controlled plans (reviewable in PRs), isolated execution passes, dynamic discovery of project tools and standards (not hardcoded to any stack), and awareness of nested `CLAUDE.md` files. No hidden state, no new workflow to learn.

For a detailed comparison, see [docs/evaluation.md](docs/evaluation.md).

## What's in this repo

```
commands/
  implement-plan.md      # Orchestrator — chains all 3 skills in isolated passes

skills/
  plan-review/           # /plan-review — finds gaps, wrong assumptions, integration risks
  tdd-implementation/    # /tdd-implementation — TDD for logic, discovers verification commands
  standards-review/      # /standards-review — checks diff against dynamically discovered standards

docs/
  workflow.md            # The full workflow: discussion > plan > implement
  evaluation.md          # Comparison with GSD and Superpowers
```

Each skill works standalone (`/plan-review my-plan.md`) and composes via the orchestrator (`/implement-plan my-plan.md`).

## Installation

Copy the command and skills to your project's `.claude/` directory:

```bash
mkdir -p .claude/commands .claude/skills
cp commands/implement-plan.md .claude/commands/
cp -r skills/* .claude/skills/
```

Then use it (full workflow or individual skills):

```bash
# Full workflow: review → implement → standards check
/implement-plan path/to/your/plan.md

# Or use skills individually
/plan-review path/to/your/plan.md
/tdd-implementation path/to/your/plan.md
/standards-review HEAD~3   # review changes since 3 commits ago
/standards-review main     # review changes since main
```

For personal use (not committed to the project):

```bash
mkdir -p ~/.claude/commands ~/.claude/skills
cp commands/implement-plan.md ~/.claude/commands/
cp -r skills/* ~/.claude/skills/
```

## What makes a good plan

The agent drafts the plan, but you review it. Here's what to look for — a precise plan produces precise code, a vague plan produces drift.

- **Context**: What problem this solves and why this approach
- **Files to modify**: Exact paths, what changes in each
- **Code details**: Type signatures, method signatures, key logic
- **What stays unchanged**: Explicitly list what should NOT be touched
- **Edge cases**: Scenarios and expected behavior
- **Test scenarios**: What to test, with expected inputs/outputs
- **Verification**: Commands to run to confirm correctness

## Limitations

These are prompts, not code. There are no hard guarantees that the agent will:

- Run tests / lint / type checking after every commit
- Follow TDD strictly
- Review carefully

In practice, well-written prompts with Opus are followed 95%+ of the time. For hard guarantees, use git pre-commit hooks (which agents trigger when they commit).

## License

MIT
