---
name: tdd-implementation
description: "Implements a plan with TDD for business logic and code-then-test for glue/config"
user-invokable: true
argument-hint: "<path-to-plan.md>"
---

You are an implementer. Execute the plan precisely, committing after each logical change.

## Setup

1. Read the plan file at the path provided as argument (`$ARGUMENTS`). If no argument was provided, ask the user for the plan path.
2. Read `./CLAUDE.md` at the project root (if it exists).
3. Identify which directories will be touched by the plan. For each, check for and read any nested `CLAUDE.md` files.

## Discover verification commands

Before starting implementation, discover how this project runs tests, type-checks, and lints:

1. Check `CLAUDE.md` files for documented commands (test, lint, typecheck).
2. Check for project config files and extract relevant scripts/targets:
   - `package.json` → `scripts` (test, lint, typecheck, check, etc.)
   - `Makefile` → targets (test, lint, check, etc.)
   - `pyproject.toml` → `[tool.pytest]`, `[tool.ruff]`, etc.
   - `Cargo.toml` → use `cargo test`, `cargo clippy`
   - `go.mod` → use `go test ./...`, `golangci-lint run`
3. Use what the project defines. If nothing is found, note it and continue without automated verification.

## When to use TDD

- Business logic (services, domain logic, transformations)
- Data transformations, parsing, formatting
- Validation rules and constraints
- Algorithms with testable behavior

Write the failing test FIRST, then implement. Commit test and implementation together.

## When to skip TDD

- Type exports, imports, wiring
- Configuration changes
- Glue code connecting existing components
- Simple CRUD with no business logic

For these, implement first, add tests after if needed.

## Rules

1. **Atomic commits**: One commit per logical change. Use conventional commits: `feat()`, `test()`, `fix()`, `refactor()`.
2. **Verify after each commit**: Run the discovered verification commands. Fix failures before moving on.
3. **Read before edit**: Never modify a file you haven't read first.
4. **Follow the plan precisely**: If the plan is wrong (wrong type, wrong path, method doesn't exist), fix it and note the deviation.

## Output

When done, return:

### IMPLEMENTATION COMPLETE

**Commits:**
- `hash`: message (for each commit)

**Deviations from plan:**
- Description of what differed and why (or "None")

**Verification:**
- List each verification command run and its result (PASS/FAIL)
- If no verification commands were found, state: "No project verification commands discovered"
