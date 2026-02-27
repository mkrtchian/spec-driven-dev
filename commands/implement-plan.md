---
description: "Implement a versioned markdown plan with review, TDD, and standards passes"
argument-hint: "<path-to-plan.md>"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

## Objective

Implement a markdown plan through 3 isolated passes with fresh context each.
Each pass is a dedicated subagent that focuses on one concern only.

Pass 1: Plan review — find gaps, ambiguities, risks in the plan before coding
Pass 2: Implementation — TDD for business logic, code-then-test for glue/config
Pass 3: Standards review — verify the diff respects project coding standards

## Context

Plan file: $ARGUMENTS

If no argument provided, use AskUserQuestion to ask: "Which plan file should I implement?" with a hint to provide the path.

## Process

## 0. Validate

Read the plan file at $ARGUMENTS. If it doesn't exist, error and stop.

Record the current git HEAD before any changes:
```bash
BASELINE_SHA=$(git rev-parse HEAD)
```

## 1. Pass 1: Plan Review (fresh agent)

Display:
```
--- Pass 1: Plan Review ---
Spawning reviewer to check for gaps, ambiguities, and risks...
```

Read `.claude/skills/plan-review/SKILL.md`. Use its content as the system prompt for the subagent.

Spawn a subagent:

```
Task(
  subagent_type="general-purpose",
  model="sonnet",
  description="Review plan for gaps",
  prompt="
    <skill>
    {content of .claude/skills/plan-review/SKILL.md, excluding frontmatter}
    </skill>

    <context>
    Plan to review: $PLAN_PATH
    </context>
  "
)
```

If ISSUES FOUND with any blocking issues:
- Present the issues to the user
- Ask: "Fix these in the plan before implementing, or proceed anyway?"
- If fix: stop and let user update the plan
- If proceed: continue with noted risks

If ISSUES FOUND with only important/minor:
- Display them to the user as warnings
- Continue to Pass 2

## 2. Pass 2: Implementation (fresh agent)

Display:
```
--- Pass 2: Implementation ---
Spawning implementer...
```

Read `.claude/skills/tdd-implementation/SKILL.md`. Use its content as the system prompt for the subagent.

Spawn a subagent:

```
Task(
  subagent_type="general-purpose",
  description="Implement plan",
  prompt="
    <skill>
    {content of .claude/skills/tdd-implementation/SKILL.md, excluding frontmatter}
    </skill>

    <context>
    Plan to implement: $PLAN_PATH
    </context>
  "
)
```

If implementation returns FAIL on any verification: present to user, ask how to proceed.

## 3. Pass 3: Standards Review (fresh agent)

Display:
```
--- Pass 3: Standards Review ---
Spawning reviewer to check diff against coding standards...
```

Get the diff since baseline:
```bash
DIFF=$(git diff $BASELINE_SHA..HEAD --stat)
CHANGED_FILES=$(git diff $BASELINE_SHA..HEAD --name-only)
```

Read `.claude/skills/standards-review/SKILL.md`. Use its content as the system prompt for the subagent.

Spawn a subagent:

```
Task(
  subagent_type="general-purpose",
  model="sonnet",
  description="Review standards compliance",
  prompt="
    <skill>
    {content of .claude/skills/standards-review/SKILL.md, excluding frontmatter}
    </skill>

    <context>
    These files were modified since baseline:
    $CHANGED_FILES
    </context>
  "
)
```

If ISSUES FOUND:
- Apply the fixes directly (they should be small/mechanical)
- Discover verification commands the same way the implementation skill does:
  check CLAUDE.md, package.json scripts, Makefile targets, pyproject.toml, Cargo.toml, go.mod
- Run whatever verification commands the project defines
- Commit: `refactor: apply coding standards`

## 4. Summary

Display:
```
--- Implementation Complete ---

Plan: $PLAN_PATH
Commits: (list all commits from $BASELINE_SHA to HEAD)
Passes:
  1. Plan review: {GOOD / N issues found}
  2. Implementation: {N commits, deviations: Y/N}
  3. Standards review: {COMPLIANT / N fixes applied}
```
