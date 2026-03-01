---
name: implement-plan
description: "Execute a reviewed implementation plan step by step with verification and standards review"
argument-hint: "<path-to-plan.md>"
disable-model-invocation: true
---

## Objective

Execute a reviewed plan through its implementation steps. Each step gets a dedicated implementer agent, followed by a step hardener that verifies completeness and fixes emergent issues. After all steps, a standards review and final review close the loop.

## Context

Plan file: $ARGUMENTS

If no argument provided, use AskUserQuestion to ask: "Which plan file should I implement?" with a hint to provide the path.

## 0. Validate

Read the plan file at $ARGUMENTS. If it doesn't exist, error and stop.

Check that the plan has an `## Implementation steps` section. If not, tell the user: "This plan has no implementation steps. Run `/write-plan` first, or add an `## Implementation steps` section manually."

Record the current git HEAD before any changes:
```bash
BASELINE_SHA=$(git rev-parse HEAD)
```

Parse the implementation steps from the plan. Each step starts with `### Step N:` or `**Step N:`.

## 1. Execute steps

For each step in order:

### 1a. Implement (fresh sub-agent)

Display:
```
--- Step N: [step title] ---
Spawning implementer...
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-implementer",
  model="opus",
  description="Implement step N",
  prompt="
    ## Step to implement

    {content of step N from the plan}

    ## Full plan context

    {content of the plan file, for reference — but focus on the step above}
  "
)
```

If the implementer reports FAIL on verification: present to user, ask how to proceed.

### 1b. Step hardening (fresh sub-agent)

Display:
```
--- Step N: Hardening ---
Verifying and fixing if needed...
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-step-hardener",
  model="opus",
  description="Harden step N",
  prompt="
    ## Step that was supposed to be implemented

    {content of step N from the plan}

    ## Plan file path

    $ARGUMENTS
  "
)
```

Handle the result:

- **STEP COMMITTED**: Continue to next step.
- **STEP COMMITTED WITH FIXES**: Note the fixes applied, continue to next step.
- **ISSUES FOUND**: No commit was made. Present the issues to the user. Ask: "Fix these issues, skip them, or stop implementation?"
  - If fix: spawn another implementer to address the issues, then re-harden
  - If skip: the orchestrator commits as-is, then continue to next step
  - If stop: go directly to the summary

Record the step result (committed / committed with fixes / issues found + action taken).

## 2. Standards review (fresh sub-agent)

Display:
```
--- Standards Review ---
Checking all changes against project coding standards...
```

Get the full diff since baseline:
```bash
CHANGED_FILES=$(git diff $BASELINE_SHA..HEAD --name-only)
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-standards-reviewer",
  model="opus",
  description="Review standards compliance",
  prompt="
    These files were modified since baseline ($BASELINE_SHA):
    $CHANGED_FILES
  "
)
```

If ISSUES FOUND:
- Apply the fixes directly
- Discover verification commands the same way the implementation skill does: check CLAUDE.md, package.json scripts, Makefile targets, pyproject.toml, Cargo.toml, go.mod
- Run verification commands
- Commit: `refactor: apply coding standards`

## 3. Final review (fresh sub-agent)

Display:
```
--- Final Review ---
Reviewing implementation against the full plan...
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-final-reviewer",
  model="opus",
  description="Final implementation review",
  prompt="
    Plan file: $ARGUMENTS
    Baseline: $BASELINE_SHA
  "
)
```

## 4. Summary

Display:

```
--- Implementation Complete ---

Plan: $ARGUMENTS

Steps:
  Step 1: [title] — [committed / committed with N fixes / issues: action taken]
  Step 2: [title] — [committed / committed with N fixes / issues: action taken]
  ...

Standards review: [COMPLIANT / N fixes applied]

Commits: (list all commits from $BASELINE_SHA to HEAD with hash and message)

Final review remarks:
[remarks from the final review agent]
```
