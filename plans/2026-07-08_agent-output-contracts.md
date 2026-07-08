# Plan: harden the agent output contracts

## Context

The execution agents share a few output-contract weaknesses that surface during real runs:

1. **The implementer has only one exit state.** `sdd-implementer` returns `IMPLEMENTATION COMPLETE` and nothing else. When verification fails and it cannot repair, or when the step's approach is fundamentally wrong, the contract still forces a "complete" declaration with a `FAIL` line buried in the Verification section. The `implement-plan` orchestrator then hands a falsely-complete step to the hardener instead of routing to the developer.

2. **Verification results are asserted, not shown.** The implementer reports self-declared `PASS`/`FAIL`, and the hardener, standards-enforcer, and final-reviewer declare their commits/compliance without quoting the verification output they ran. A reader cannot audit the claim without re-running the commands.

3. **`sdd-step-breakdown` has no way to signal an unbreakdownable plan.** Its only output is `STEPS ADDED`. A plan too ambiguous to cut into concrete steps still gets cut, producing vague steps.

4. **Committing agents can bypass pre-commit hooks.** None of the three committing agents (hardener, standards-enforcer, final-reviewer) are told not to use `git commit --no-verify`, so a blocking pre-commit hook remains silently bypassable.

5. **The skip path commits without convention discovery.** In `implement-plan`, when the hardener returns `ISSUES FOUND` and the developer chooses to skip, the orchestrator commits the step as-is without discovering the project's commit conventions. This is the only place a commit is made without the discovery every committing agent performs.

6. **Hardener remarks have no guaranteed surface.** A trade-off flagged by the hardener at an early step does not appear anywhere in the final `implement-plan` summary, which only carries the final reviewer's remarks.

7. **A stale shell rule.** Most executing agents carry "Never chain commands with `&&`". This existed to keep each command individually reviewable before permission auto mode. It is no longer needed and is dead guidance.

## Approach

Tighten the output contracts of the five execution agents and the two orchestrator skills, without changing the isolation model or the sequential flow. Each change is localized prompt text. No new agent, no code, no state.

The one structural change is the implementer's exit state: from a single `IMPLEMENTATION COMPLETE` to two mutually exclusive states, `IMPLEMENTATION COMPLETE` and `IMPLEMENTATION BLOCKED`. The orchestrator routes `BLOCKED` to the developer (a status report, not an edit, so isolation and the single-writer model are untouched).

## Files to modify

### 1. `agents/sdd-implementer.md`

- **Two exit states.** Replace the single `### IMPLEMENTATION COMPLETE` output with two:
  - `### IMPLEMENTATION COMPLETE`: returned only when verification passes, or when no verification commands were discovered. Keeps the `Deviations` and `Verification` sections.
  - `### IMPLEMENTATION BLOCKED`: returned when the step cannot be brought to a passing state. Two triggers: (a) verification fails and the implementer cannot fix it, or (b) the step's approach is fundamentally invalid (not a local mechanical error). Body sections: **What blocks** (the specific obstacle), **What was tried** (the attempts made), **State of the tree** (what was changed and left uncommitted).
- **Approach invalidation.** Extend Rule 4. It currently covers a mechanically wrong step (wrong type, wrong path, method does not exist) which the implementer fixes and notes as a deviation. Add: if the step's whole approach does not work (not a local fix), do not force it, return `IMPLEMENTATION BLOCKED` explaining why the approach fails.
- **Reconcile Rule 2.** Rule 2 ("Verify before finishing ... Fix failures before declaring done") reads as an unconditional obligation to fix. Add the escape hatch so it does not conflict with the new exit state: if verification cannot be made to pass after reasonable attempts, stop trying and return `IMPLEMENTATION BLOCKED` (do not loop, do not declare `COMPLETE` with a buried `FAIL`).
- **Evidence in verification.** In the `Verification` section, require quoting the tail of the actual command output (e.g. `47 passed, 0 failed`) alongside each command, not just a `PASS`/`FAIL` verdict.
- **Remove the `## Shell commands` section** (the `&&` rule).

### 2. `agents/sdd-step-hardener.md`

- **Evidence in output.** In `### STEP COMMITTED` and `### STEP COMMITTED WITH FIXES`, add a line citing the verification commands run and the tail of their output (e.g. `47 passed, 0 failed`).
- **Forbid `--no-verify`.** In the `## Commit` section, add: never use `git commit --no-verify`.
- **Remove the `## Shell commands` section** (the `&&` rule).

### 3. `agents/sdd-standards-enforcer.md`

- **Evidence in output.** In `### STANDARDS ENFORCED`, add a line citing the verification commands run and the tail of their output.
- **Forbid `--no-verify`.** In the `## Action` commit step, add: never use `git commit --no-verify`.
- **Remove the `## Shell commands` section** (the `&&` rule).

### 4. `agents/sdd-final-reviewer.md`

- **Evidence in output.** In the `## Output` block, when fixes were committed, cite the verification commands run and the tail of their output (add to `Fixes applied` or as a short verification line).
- **Forbid `--no-verify`.** In the fix/commit paragraph, add: never use `git commit --no-verify`.
- **Remove the `## Shell commands` section** (the `&&` rule).

### 5. `agents/sdd-step-breakdown.md`

- **Ambiguity exit state.** Add a second output state alongside `### STEPS ADDED`:
  - `### PLAN TOO AMBIGUOUS TO BREAK DOWN`: returned when the plan is too underspecified to cut into concrete, executable steps. Body: **What is unclear** and **What the plan must resolve** before a breakdown is possible. When this state is returned, do NOT append or modify the `## Implementation steps` section.
- Note in the `## Action` section that appending steps applies only when the plan is concrete enough. Otherwise, return the ambiguity state.

### 6. `skills/implement-plan/SKILL.md`

- **§1a route the two states.** Replace "If the implementer reports FAIL on verification: present to user, ask how to proceed." with handling for the two states. On `IMPLEMENTATION COMPLETE`, proceed to hardening (§1b). On `IMPLEMENTATION BLOCKED`, present the block (what blocks, what was tried, tree state) to the developer and ask how to proceed, and do NOT spawn the hardener on a blocked step. Keep it open-ended (the developer may rework the plan and retry, unblock manually, or stop) rather than offering a skip-and-continue, since a blocked step usually leaves later dependent steps unbuildable.
- **§1b fix path handles a blocked re-implementer.** The `ISSUES FOUND` fix branch re-spawns an implementer, then re-hardens. If that re-spawned implementer returns `IMPLEMENTATION BLOCKED`, handle it exactly as §1a: present the block to the developer and ask how to proceed, do NOT re-harden a blocked tree.
- **§1b skip path discovers conventions.** Replace "If skip: the orchestrator commits as-is, then continue to next step" with: on skip, discover the project's commit conventions using the same priority order as `agents/sdd-standards-enforcer.md` "Discover commit conventions" (CLAUDE.md rules, then a `/commit` skill or command, then commitlint/commitizen config, else conventional commits), then stage only the changed files by name (never `git add -A` or `git add .`) and commit following them, then continue.
- **§4 summary carries hardener remarks.** Add a `Hardener remarks:` section to the summary template, collecting the remarks reported by the hardener across steps (from `STEP COMMITTED WITH FIXES`). Keep the existing `Final review remarks` section.

### 7. `skills/write-plan/SKILL.md`

- **Phase 6 handles the ambiguity state.** After the step-breakdown spawn, handle its output: on `STEPS ADDED`, report the step summary and proceed to Phase 7 as today; on `PLAN TOO AMBIGUOUS TO BREAK DOWN`, surface what is unclear and what the plan must resolve, and stop before the Phase 7 "Plan Ready" summary (the plan has no steps yet, so it is not ready for `/implement-plan`).

### 8. `.claude-plugin/plugin.json`

- Bump `version` `1.8.7` to `1.8.8`.

### 9. `docs/workflow.md`, `docs/design-decisions.md`, and `README.md`

Keep the public docs consistent with the new behavior. Factual only, no new claims.

- **`docs/workflow.md`**, the "When things go wrong" list: update the implementer line. It currently reads "Implementer fails verification (tests, lint, or typecheck don't pass): the orchestrator presents the failure and asks how to proceed." Change it to reflect the `IMPLEMENTATION BLOCKED` state: the implementer reports it is blocked (verification cannot be made to pass, or the step's approach is invalid), and the orchestrator presents the block and asks how to proceed, without hardening the step.
- **`docs/design-decisions.md`**, the pre-commit-hooks sentence ("For stronger guarantees on tests, lint, and typecheck, pair the workflow with pre-commit hooks: agents trigger them on every commit."): add that the committing agents are instructed never to bypass hooks with `--no-verify`, so the pairing is not defeated in-prompt.
- **`README.md`**, the Reliability section: change "For hard guarantees on test/lint/typecheck, pair with git pre-commit hooks." from "hard" to "stronger", aligning with `design-decisions.md` (the `--no-verify` prohibition strengthens the pairing, but adherence remains prompt-strength).

## What stays unchanged

- The isolation model, the fresh-context-per-pass principle, and sequential execution.
- Every agent's core role and review dimensions. Only their output contracts and the noted rules change.
- The hardener still always runs after a completed step, still leaves the step's work uncommitted on `ISSUES FOUND` (the developer decides fix/skip/stop). The standards-enforcer still reverts its own failed fixes on `ISSUES FOUND`.
- The `--no-verify` prohibition is not added to the implementer (it does not commit).
- The `ISSUES FOUND` handling wording across §1b and §2 is left as-is (§3 final review has no result-handling block to change), except the §1b fix path gains handling for a blocked re-implementer (see Files to modify).

## Edge cases

- **Verification passes but a deviation occurred**: still `IMPLEMENTATION COMPLETE`, deviation noted, output tail quoted.
- **No verification commands discovered**: `IMPLEMENTATION COMPLETE` is allowed (the Verification section states none were found), since there is nothing to fail.
- **Approach invalid with a clean tree**: `IMPLEMENTATION BLOCKED` with "State of the tree: no changes made".
- **Skip on a step with no discoverable commit convention**: the discovery falls back to conventional commits, same as the agents.
- **Plan is concrete**: `sdd-step-breakdown` returns `STEPS ADDED` as before. The ambiguity state is only for genuinely unbreakdownable plans.
- **A blocked step**: `implement-plan` presents the block to the developer and asks how to proceed, without spawning the hardener on the blocked step. The developer can rework the plan and retry, unblock manually, or stop.

## Test scenarios

Markdown-prompt behavior, no automated runner. Verify by reading the contracts and by driving `/implement-plan` and `/write-plan` against crafted inputs:

1. A step whose verification cannot be made to pass, with no fix available, yields `IMPLEMENTATION BLOCKED` (not `COMPLETE` with a buried `FAIL`), and `implement-plan` routes it to the developer without spawning the hardener. The same routing applies to a re-spawned implementer on the `ISSUES FOUND` fix path that returns `IMPLEMENTATION BLOCKED`.
2. A step whose approach is fundamentally wrong yields `IMPLEMENTATION BLOCKED` explaining why, rather than a forced implementation.
3. A passing step's `IMPLEMENTATION COMPLETE` quotes the real verification output tail. The hardener, enforcer, and final reviewer likewise quote the output they ran.
4. A deliberately underspecified plan run through `/write-plan` yields `PLAN TOO AMBIGUOUS TO BREAK DOWN` with what is unclear, and the command stops before "Plan Ready".
5. On the hardener's `ISSUES FOUND` with the developer choosing skip, the resulting commit follows the discovered project convention (correct type/scope, no `git add -A`).
6. A hardener remark flagged at an early step appears in the final `implement-plan` summary.
7. No agent prompt contains a `## Shell commands` `&&` rule.

## Verification

No automated tests (markdown-only repo). Checks:

- `grep -rl 'Never chain commands' agents/ skills/` returns nothing (the `&&` rule is gone from every agent and skill).
- `grep -c 'IMPLEMENTATION BLOCKED' agents/sdd-implementer.md` is at least 1, and `implement-plan` references `IMPLEMENTATION BLOCKED`.
- `grep -c 'PLAN TOO AMBIGUOUS' agents/sdd-step-breakdown.md` is at least 1, and `write-plan` references it.
- `grep -c 'no-verify' agents/sdd-step-hardener.md agents/sdd-standards-enforcer.md agents/sdd-final-reviewer.md` returns a hit in each.
- `grep -c 'Hardener remarks' skills/implement-plan/SKILL.md` is 1.
- `grep -n 'Discover commit conventions\|commit conventions' skills/implement-plan/SKILL.md` shows the skip path now discovers conventions.
- `.claude-plugin/plugin.json` shows `1.8.8`.
- Manual: point the plugin at the working tree (`--plugin-dir`) and drive scenarios 1, 4, 5 end to end.

## Implementation steps

This is a markdown-only prompt repo (no build, no test runner). All changes are localized prompt-text edits across 11 small files that together form one cohesive change and one atomic commit. Verification is by `grep` (per the plan's Verification section) plus a read-through of the edited contracts.

### Step 1: Harden the agent output contracts

**Files** (all under repo root `/home/roman/Projects/spec-driven-dev/`):

- `agents/sdd-implementer.md`
- `agents/sdd-step-hardener.md`
- `agents/sdd-standards-enforcer.md`
- `agents/sdd-final-reviewer.md`
- `agents/sdd-step-breakdown.md`
- `skills/implement-plan/SKILL.md`
- `skills/write-plan/SKILL.md`
- `.claude-plugin/plugin.json`
- `docs/workflow.md`
- `docs/design-decisions.md`
- `README.md`

**Do** (apply exactly the "Files to modify" section of this plan; per-file summary):

1. `agents/sdd-implementer.md`
   - Replace the single `### IMPLEMENTATION COMPLETE` exit with two mutually exclusive states. `### IMPLEMENTATION COMPLETE` is returned only when verification passes or when no verification commands were discovered; keep its `Deviations` and `Verification` sections. Add `### IMPLEMENTATION BLOCKED`, returned when the step cannot reach a passing state, with two triggers: (a) verification fails and cannot be fixed, (b) the step's whole approach is fundamentally invalid (not a local mechanical error). Body sections: **What blocks**, **What was tried**, **State of the tree** (changed and left uncommitted).
   - Extend Rule 4: it currently covers a mechanically wrong step the implementer fixes and notes as a deviation. Add that if the whole approach does not work (not a local fix), do not force it, return `IMPLEMENTATION BLOCKED` explaining why the approach fails.
   - Reconcile Rule 2 ("Fix failures before declaring done"): add that if verification cannot be made to pass after reasonable attempts, stop and return `IMPLEMENTATION BLOCKED` instead of looping or burying a `FAIL` under `COMPLETE`.
   - In the `Verification` section, require quoting the tail of the actual command output (e.g. `47 passed, 0 failed`) alongside each command, not just a `PASS`/`FAIL` verdict.
   - Remove the `## Shell commands` section (the `&&` rule). Do NOT add the `--no-verify` prohibition here (the implementer does not commit).

2. `agents/sdd-step-hardener.md`
   - In `### STEP COMMITTED` and `### STEP COMMITTED WITH FIXES`, add a line citing the verification commands run and the tail of their output.
   - In the `## Commit` section, add: never use `git commit --no-verify`.
   - Remove the `## Shell commands` section.

3. `agents/sdd-standards-enforcer.md`
   - In `### STANDARDS ENFORCED`, add a line citing the verification commands run and the tail of their output.
   - In the `## Action` commit step, add: never use `git commit --no-verify`.
   - Remove the `## Shell commands` section.

4. `agents/sdd-final-reviewer.md`
   - In the `## Output` block, when fixes were committed, cite the verification commands run and the tail of their output (add to `Fixes applied` or as a short verification line).
   - In the fix/commit paragraph, add: never use `git commit --no-verify`.
   - Remove the `## Shell commands` section.

5. `agents/sdd-step-breakdown.md`
   - Add a second output state `### PLAN TOO AMBIGUOUS TO BREAK DOWN`, returned when the plan is too underspecified to cut into concrete steps. Body: **What is unclear** and **What the plan must resolve**. When returned, do NOT append or modify the `## Implementation steps` section. Note in `## Action` that appending applies only when the plan is concrete enough; otherwise return the ambiguity state.

6. `skills/implement-plan/SKILL.md`
   - §1a: replace the FAIL-on-verification handling with routing for the two implementer states. On `IMPLEMENTATION COMPLETE`, proceed to hardening (§1b). On `IMPLEMENTATION BLOCKED`, present the block (what blocks, what was tried, tree state) to the developer and ask how to proceed, and do NOT spawn the hardener. Keep it open-ended (rework the plan and retry, unblock manually, or stop) rather than a skip-and-continue, since a blocked step usually leaves later dependent steps unbuildable. In the `ISSUES FOUND` fix branch, if the re-spawned implementer returns `IMPLEMENTATION BLOCKED`, apply the same §1a handling (present and ask how to proceed) instead of re-hardening a blocked tree.
   - §1b skip path: replace "commits as-is, then continue" with: on skip, discover the project's commit conventions using the same priority order as `agents/sdd-standards-enforcer.md` "Discover commit conventions" (CLAUDE.md rules, then a `/commit` skill or command, then commitlint/commitizen config, else conventional commits), stage only the changed files by name (never `git add -A` / `git add .`), commit following them, then continue.
   - §4 summary: add a `Hardener remarks:` section collecting remarks reported across steps (from `STEP COMMITTED WITH FIXES`); keep the existing `Final review remarks` section.

7. `skills/write-plan/SKILL.md`
   - Phase 6: after the step-breakdown spawn, handle its output. On `STEPS ADDED`, report the step summary and proceed to Phase 7 as today. On `PLAN TOO AMBIGUOUS TO BREAK DOWN`, surface what is unclear and what the plan must resolve, and stop before the Phase 7 "Plan Ready" summary.

8. `.claude-plugin/plugin.json`
   - Bump `version` from `1.8.7` to `1.8.8`.

9. Docs (factual consistency, no new claims)
   - `docs/workflow.md`: in "When things go wrong", update the implementer line to the `IMPLEMENTATION BLOCKED` behavior (blocked = verification cannot pass or the approach is invalid; the orchestrator presents the block and asks how to proceed, without hardening the step).
   - `docs/design-decisions.md`: in the pre-commit-hooks sentence, add that committing agents never bypass hooks with `--no-verify`.
   - `README.md`: in Reliability, change "hard guarantees" to "stronger guarantees" (align with design-decisions).

Keep everything under "What stays unchanged" intact: isolation model, sequential flow, each agent's core role and review dimensions, the `ISSUES FOUND` wording across §1b/§2 (§3 has no result-handling block), the hardener always running after a completed step and leaving work uncommitted on `ISSUES FOUND`.

**Test** (behavioral, no runner, validate against the plan's Test scenarios by reading the edited contracts):

1. A step whose verification cannot pass with no available fix yields `IMPLEMENTATION BLOCKED` (not `COMPLETE` with a buried `FAIL`), and `implement-plan` routes it to the developer without spawning the hardener. A re-spawned implementer on the fix path that returns `IMPLEMENTATION BLOCKED` gets the same routing.
2. A fundamentally wrong approach yields `IMPLEMENTATION BLOCKED` explaining why.
3. A passing step's `IMPLEMENTATION COMPLETE` quotes the real verification output tail; hardener, enforcer, and final reviewer likewise quote what they ran.
4. An underspecified plan yields `PLAN TOO AMBIGUOUS TO BREAK DOWN`; `write-plan` stops before "Plan Ready".
5. On hardener `ISSUES FOUND` + skip, the commit follows the discovered convention (correct type/scope, no `git add -A`).
6. A hardener remark at an early step appears in the final `implement-plan` summary.
7. No agent prompt retains a `## Shell commands` `&&` rule.

**Verify** (run from repo root; each must hold):

- `grep -rl 'Never chain commands' agents/ skills/` returns nothing (the `&&` rule is gone everywhere).
- `grep -c 'IMPLEMENTATION BLOCKED' agents/sdd-implementer.md` is at least 1, and `grep -c 'IMPLEMENTATION BLOCKED' skills/implement-plan/SKILL.md` is at least 1.
- `grep -c 'PLAN TOO AMBIGUOUS' agents/sdd-step-breakdown.md` is at least 1, and `grep -c 'PLAN TOO AMBIGUOUS' skills/write-plan/SKILL.md` is at least 1.
- `grep -l 'no-verify' agents/sdd-step-hardener.md agents/sdd-standards-enforcer.md agents/sdd-final-reviewer.md` returns all three files.
- `grep -c 'Hardener remarks' skills/implement-plan/SKILL.md` is 1.
- `grep -n 'commit conventions' skills/implement-plan/SKILL.md` shows the skip path now discovers conventions.
- `grep '1.8.8' .claude-plugin/plugin.json` matches.
- `grep -n 'blocked' docs/workflow.md` shows the updated implementer failure line; `grep -c 'no-verify' docs/design-decisions.md` is at least 1; `grep -c 'hard guarantees' README.md` is 0 and `grep -c 'stronger guarantees' README.md` is at least 1.
- Read-through: implementer has exactly two exit states, each committing agent quotes verification output, `plugin.json` remains valid JSON.
