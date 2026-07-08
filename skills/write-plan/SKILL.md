---
name: write-plan
description: "Discuss a feature, draft an implementation plan, and review it through isolated passes"
argument-hint: "<optional-path-to-requirements-or-spec>"
disable-model-invocation: true
---

## Objective

Guide the user from an idea or spec to a reviewed, step-by-step implementation plan ready for `/implement-plan`.

The process has two modes: interactive discussion (you talk to the user), and isolated review passes (fresh sub-agents each handle one concern).

## Phase 1: Discussion (interactive)

If `$ARGUMENTS` is provided, read it as requirements context (spec, user stories, discovery notes, or a rough plan).

### Explore the codebase

1. Read `./CLAUDE.md` at the project root (if it exists).
2. Identify which areas of the codebase are relevant to the user's idea. Read key source files to understand current patterns, types, and architecture.
3. Check for nested `CLAUDE.md` files in relevant directories.

### Clarify with the user

Ask questions to narrow the scope. Focus on:

- **What**: What exactly should the feature/change do? What are the acceptance criteria?
- **Where**: Which parts of the codebase are affected? Are there existing patterns to follow?
- **How**: What technical approach? Are there trade-offs to decide?
- **Boundaries**: What should NOT change? What's out of scope?
- **Testing**: What scenarios need test coverage?

Keep asking until you have enough to write a precise plan. The user signals readiness by saying something like "OK", "let's go", "write it", "that's enough", etc.

## Phase 2: Draft the plan

Write the plan to `plans/YYYY-MM-DD_slug.md` where:
- `YYYY-MM-DD` is today's date
- `slug` is a short kebab-case name derived from the feature (e.g., `user-auth`, `api-pagination`)

Create the `plans/` directory if it doesn't exist.

The plan should cover:

- **Context**: What problem this solves, why this approach
- **Approach**: High-level strategy
- **Files to modify**: Exact paths, what changes in each
- **Code details**: Type signatures, method signatures, key logic
- **What stays unchanged**: Explicitly list what should NOT be touched
- **Edge cases**: Scenarios and expected behavior
- **Test scenarios**: What to test, with expected inputs/outputs
- **Verification**: Commands to run to confirm correctness

Tell the user the plan path when done. From this point on, `$PLAN_PATH` refers to the path of the plan file you just created.

## Phase 3: Plan review (fresh sub-agent)

Display:
```
--- Pass 1: Plan Review ---
Checking for gaps, wrong assumptions, and integration risks...
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-plan-reviewer",
  model="opus",
  description="Review plan for gaps",
  prompt="
    Plan to review: $PLAN_PATH
  "
)
```

Report what was found/fixed to the user.

## Phase 4: Plan standards (fresh sub-agent)

Display:
```
--- Pass 2: Plan Standards ---
Checking plan against project coding and testing standards...
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-plan-standards",
  model="opus",
  description="Check plan standards",
  prompt="
    Plan to review: $PLAN_PATH
  "
)
```

Report what was found/fixed to the user.

## Phase 5: Due diligence (fresh sub-agent)

Display:
```
--- Pass 3: Due Diligence ---
Web-verifying external facts and surfacing decisions that need your judgment...
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-plan-diligence",
  model="opus",
  description="Due-diligence pass",
  prompt="
    Plan to vet: $PLAN_PATH
  "
)
```

Present both buckets (`### FIXED` and `### REQUIRES YOUR JUDGMENT`) to the user verbatim: never drop, merge, or reword a finding. You may append a short annotation to an individual item that was already settled during the discussion (e.g. "already decided in discussion: you accepted this risk with mitigation X"), since you alone hold the discussion context and the isolated agent cannot know what the user already chose. The agent's original text always stays visible above your annotation. Show `REQUIRES YOUR JUDGMENT` prominently.

## Phase 6: Step breakdown (fresh sub-agent)

Display:
```
--- Pass 4: Step Breakdown ---
Splitting plan into implementation steps...
```

Spawn a sub-agent:

```
Task(
  subagent_type="spec-driven-dev:sdd-step-breakdown",
  model="opus",
  description="Break plan into steps",
  prompt="
    Plan to process: $PLAN_PATH
  "
)
```

Report the step summary to the user.

## Phase 7: Done

Display:

```
--- Plan Ready ---

Plan: $PLAN_PATH
Steps: N total

Review the plan and adjust as needed (manually or with my help).
When ready, run: /implement-plan $PLAN_PATH
```

Then:

1. **Re-surface `REQUIRES YOUR JUDGMENT`**: repeat the diligence agent's `REQUIRES YOUR JUDGMENT` items as the things to resolve before implementing. If that bucket was empty, say so.
2. **Propose committing the plan (never do it automatically)**: ask the user whether to commit the plan now, so it exists in git before `/implement-plan`.
   - On **yes**: discover the project's commit convention using the same priority order the committing agents use (see `agents/sdd-standards-enforcer.md`, "Discover commit conventions"): CLAUDE.md rules first, then a `/commit` skill or command, then commitlint/commitizen config. Stage ONLY the plan file (never `git add -A` or `git add .`) and commit following that convention. If nothing more specific is discovered, fall back to `docs(plans): add <slug>`.
   - On **no**: leave the plan uncommitted. `/implement-plan` still works on the uncommitted file.
