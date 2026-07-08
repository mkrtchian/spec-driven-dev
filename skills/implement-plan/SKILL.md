---
name: implement-plan
description: "Execute a reviewed implementation plan step by step with verification and standards review"
argument-hint: "<path-to-plan.md>"
disable-model-invocation: true
---

## Objective

Execute a reviewed plan through its implementation steps. Each step gets a dedicated implementer agent, followed by a step hardener that verifies completeness and fixes emergent issues. After all steps, a standards review and final review close the loop.

## Context

The plan to implement is given as `$ARGUMENTS`. It may be a direct path, a natural-language reference (e.g. "the plan we just wrote"), or empty.

## 0. Validate and resolve the plan path

Resolve `$ARGUMENTS` to a concrete plan file:

1. If `$ARGUMENTS` is a path to an existing file, use it directly.
2. If `$ARGUMENTS` is empty or is a phrase rather than a path (e.g. "the plan we just wrote", "the latest one"), interpret it and locate the intended plan among `plans/*.md` (for "the latest" or "the one we just wrote", the most recently modified `plans/*.md` file). This is a guess that launches a full implementation run, so state the resolved path back to the user and confirm before proceeding.
3. If it cannot be resolved to a single file (`$ARGUMENTS` looks like a path but the file does not exist, `plans/` is empty or absent, nothing matches, or several plausibly match), use AskUserQuestion to ask: "Which plan file should I implement?" with a hint to provide the path.

From this point on, `$PLAN_PATH` refers to this resolved plan file. Use `$PLAN_PATH`, not the raw `$ARGUMENTS`, everywhere below.

Read the plan file at `$PLAN_PATH`. If it doesn't exist, error and stop.

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

    ## Plan file path

    $PLAN_PATH
  "
)
```

Handle the implementer's exit state:

- **IMPLEMENTATION COMPLETE**: proceed to hardening (§1b).
- **IMPLEMENTATION BLOCKED**: do NOT spawn the hardener. Present the block to the developer (what blocks, what was tried, state of the tree) and ask how to proceed. Keep it open-ended: the developer may rework the plan and retry the step, unblock the tree manually, or stop. Do not offer a skip-and-continue — a blocked step usually leaves later dependent steps unbuildable.

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

    $PLAN_PATH
  "
)
```

Handle the result:

- **STEP COMMITTED**: Continue to next step.
- **STEP COMMITTED WITH FIXES**: Note the fixes applied, continue to next step.
- **ISSUES FOUND**: No commit was made. Present the issues to the user. Ask: "Fix these issues, skip them, or stop implementation?"
  - If fix: spawn another implementer to address the issues. If that implementer returns `IMPLEMENTATION BLOCKED`, handle it exactly as §1a — present the block to the developer and ask how to proceed, and do NOT re-harden a blocked tree. Otherwise (`IMPLEMENTATION COMPLETE`), re-harden.
  - If skip: discover the project's commit conventions using the same priority order as `agents/sdd-standards-enforcer.md` "Discover commit conventions" (CLAUDE.md rules first, then a `/commit` skill or command, then commitlint/commitizen config, else standard conventional commits). Stage only the changed files by name (never `git add -A` or `git add .`), commit following those conventions, then continue to next step.
  - If stop: go directly to the summary

Record the step result (committed / committed with fixes / issues found + action taken).

## 2. Standards enforcement (fresh sub-agent)

Display:
```
--- Standards Enforcement ---
Checking all changes against project coding standards...
```

Get the full diff since baseline:
```bash
CHANGED_FILES=$(git diff $BASELINE_SHA..HEAD --name-only)
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-standards-enforcer",
  model="opus",
  description="Enforce standards compliance",
  prompt="
    These files were modified since baseline ($BASELINE_SHA):
    $CHANGED_FILES
  "
)
```

Handle the result:

- **STANDARDS COMPLIANT**: Continue.
- **STANDARDS ENFORCED**: Note the fixes applied, continue.
- **ISSUES FOUND**: Present the issues to the user and ask how to proceed.

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
    Plan file: $PLAN_PATH
    Baseline: $BASELINE_SHA
  "
)
```

## 4. Summary

Display:

```
--- Implementation Complete ---

Plan: $PLAN_PATH

Steps:
  Step 1: [title] — [committed / committed with N fixes / issues: action taken]
  Step 2: [title] — [committed / committed with N fixes / issues: action taken]
  ...

Standards enforcement: [COMPLIANT / N fixes applied]

Commits: (list all commits from $BASELINE_SHA to HEAD with hash and message)

Hardener remarks:
[remarks reported by the hardener across steps, collected from each STEP COMMITTED WITH FIXES; or "None"]

Final review remarks:
[remarks from the final review agent]
```
