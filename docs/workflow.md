# Workflow

## 1. Discuss

Start a conversation with Claude Code. If you have requirements from a product discovery (user stories, acceptance criteria, technical constraints), give them to the agent upfront as context. Then let it explore the codebase, ask questions, and go back and forth until you have a shared understanding.

This is where the real engineering happens — scoping, trade-off analysis, understanding existing patterns.

## 2. Write the plan

Ask Claude to write a detailed implementation plan as a markdown file. Modern models (Opus, Sonnet) write well-structured plans out of the box — let the agent draft, then review and iterate.

A good plan typically covers: context and approach, files to modify, code details (signatures, types, key logic), what stays unchanged, edge cases, test scenarios, and verification commands. But don't force a rigid template — review the plan for precision and completeness rather than format compliance.

## 3. Version the plan

Save the plan in your repo (e.g., `docs/plans/`). Commit it. It's now reviewable in PRs by your team, even if they don't use AI tooling.

## 4. Clear context and implement

Clear the conversation context. Run:

```
/implement-plan path/to/plan.md
```

This spawns 3 isolated agents sequentially:

**Pass 1 — Plan Review**: A fresh agent reads the plan AND the actual source files. It checks if assumptions are correct, if anything is missing, if there are integration risks. This catches problems before any code is written.

**Pass 2 — Implementation**: A fresh agent implements the plan with atomic commits. TDD for business logic, code-then-test for glue. Discovers and runs the project's verification commands (tests, linter, type-checker) after each commit.

**Pass 3 — Standards Review**: A fresh agent reads the full diff and checks it against project coding standards (discovered dynamically from CLAUDE.md, CONTRIBUTING.md, linter configs, etc.). Applies mechanical fixes if needed.

## 5. Review and adjust

Check the result. Ask for adjustments if needed. The plan is still there as the reference — you can point to specific sections.

## Why clear context between phases?

A single long conversation degrades in quality as context fills up. An agent that just spent 20 minutes implementing code is not in the right mindset to review coding standards: it's biased toward defending what it just wrote. A fresh agent with only the diff and the standards document has no such bias.
