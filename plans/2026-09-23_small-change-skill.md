# Plan: a `/small-change` skill, a conditional security dimension in the final review, and standards passes that stop taking local style for the norm

## Context

The plugin has one regime today: `/write-plan` then `/implement-plan`. The docs send everything smaller to plain Claude Code or plan mode (`README.md:105`, `docs/design-decisions.md:11` and `:125`), which means a small change gets no fresh-context review at all. The need is a lighter regime where the main session writes the code and the same fresh passes review it afterwards, without a plan file, a breakdown, or per-step implementers and hardeners.

Three things come with it, in the same plan because they touch the same agents:

1. **Security has no pass on the code.** `/implement-plan` covers security only at plan time (`agents/sdd-plan-diligence.md:45-50`: accepted risks, permission scope, lethal trifecta). Nothing reads the diff for a concrete flaw born at implementation: a string-built query, an unsafe deserializer, a secret in a log line. That gap is the same in both skills, so it closes once, in `sdd-final-reviewer`, which already reads the full diff at the end of both runs.
2. **`sdd-final-reviewer` can "fix" a deliberate choice it has no record of.** With a full plan the risk is low. With the three-to-five-line intent of a small change, the reviewer holds much less of what the discussion decided, so the rule separating fixes from flags has to say where the answer comes from.
3. **Standards passes take local style for the norm** (observed on 2026-08-21 on `plans/2026-08-21_agent-context-preamble.md`: the plan-standards pass "rewrapped one line [...] past the file's ~100-column wrap", a wrap the author had introduced by accident and that no document in the repo states). Both standards agents already say "Apply **only** what the project's standards define" (`agents/sdd-standards-enforcer.md:42`, `agents/sdd-plan-standards.md:32`), but their Setup reads surrounding files "to understand current patterns" (`sdd-plan-standards.md:28`) or "to judge naming consistency, import patterns" (`sdd-standards-enforcer.md:25`) without saying what those patterns are for. The natural reading is "these are standards too", which contradicts the documented-only rule. The enforcer applies that reading to code, where it propagates a local divergence instead of catching it.

## Approach

### The `/small-change` flow

1. **Intent.** Before writing code, the main session writes three to five lines: the need, the done criterion, the files it expects to touch. It displays them without asking for approval: the single human touchpoint stays at the end of the run.
2. **Triage, owned by the project.** The plugin fixes no size criterion. The skill looks in the project's instructions for a rule on which changes need a plan, including the nested `CLAUDE.md` of the directories the intent's expected files live in: written after the intent, the triage reads those files from a list rather than a guess, and judges the rule against a written need. If a rule exists and the change exceeds it, the skill says so and recommends `/write-plan`, and the intent already written is a starting point for it. If none exists, there is no triage and nothing is said about it.
3. **Tree check.** `git status`. If a file the intent expects to touch already carries uncommitted changes, stop and ask: the change commit stages by name and would swallow the developer's work in progress. Unrelated uncommitted files do not stop the run, they are named in the summary.
4. **Implementation by the main session**, not a sub-agent: the main session holds the discussion, which is the whole reason this regime exists. Conditional TDD, discovered verification commands, and the implementer's rule on external values (verify, or write a provisional value and declare it). If verification cannot be made to pass: no commit, no review passes, report and stop.
5. **Change commit.** Record `BASELINE_SHA` before implementing. Commit the change, staging by name, with the intent verbatim in the commit body. The intent is then durable in git rather than living only in a prompt, and the passes behind read `BASELINE_SHA..HEAD` exactly as they do in `/implement-plan`.
6. **Fact check** (`sdd-fact-checker`, new intent mode), always spawned: its own gate costs one diff read when the diff has no external fact. The main session does not decide whether its own diff needs checking, because it is the context that wrote the value from memory. `NEEDS A FRESH IMPLEMENTER` items go to the developer, not to a relay: the relay and its targeted re-check would add two agents for a rare case on a small change.
7. **Standards enforcement** (`sdd-standards-enforcer`, unchanged interface): the changed files since baseline, no intent, consistent with the decision not to give the plan to the enforcer.
8. **Final review** (`sdd-final-reviewer`, new intent mode): the intent in place of the plan, plus the baseline.
9. **Summary**: intent, every commit since baseline (so any pass's fix is one `git revert` away), each pass's status, every item that needs the developer, and a note that squashing is the developer's call.

Sequential, and each review pass applies its own single-answer fixes, as in `/implement-plan`. The alternative, reviewers that only report and the main session that applies, is Cognition's "communication bridge" and it has a stronger case here than in `/implement-plan`: the main session is the only context holding the discussion, and the plan gate that makes fix-by-pass safe in `/implement-plan` (`docs/design-decisions.md:63`) does not exist here. It is still rejected, for one reason only: the main session wrote the code, and filtering the review through the author reintroduces the bias isolation exists to remove (`docs/design-decisions.md:55`). The docs must say plainly that the bet is riskier here, and name what compensates: the tightened fix/flag rule in the final reviewer, fixes in their own commits, and the summary listing them.

### `sdd-final-reviewer`: two input modes, criterion first, a reworded fix rule, conditional security

- **Mode branch** at the top of Setup, following the precedent of `agents/sdd-fact-checker.md:27`: a prompt carrying `Intent:` instead of `Plan file:` puts the agent in intent mode, and the file states once, there, what changes in that mode. This keeps the rest of the file single-mode.
- **Criterion before code.** After reading the plan or intent and before reading the diff, the reviewer writes down for itself what a correct change must satisfy. Exposing the candidate solution before the evaluation criterion measurably anchors a model on it (Ficek et al., NVIDIA), and the fix is an ordering.
- **Fix rule, reworded.** A fix has a single right answer when the artifacts the reviewer received (the code, plus the plan or the intent) settle it. When the answer depends on a choice those artifacts do not record, it is a flag, and the flag names the deliberate alternative that makes it plausible. A manifest defect (a crash, a wrong variable, an injection) is settled by the code and is fixed. And "no fixes needed" is a valid outcome, stated as a success, not a fallback. Stating the abstention as a success matters as much as stating the rule: on already-fixed code, telling GPT-5.4 mini to fix or give up raised correct abstention from 60.5% to 88.5%, while describing a procedure without naming abstention as a success did worse than the baseline for that model (60.5% to 47.5%) (Gloaguen et al., FIXEDBENCH). The same paper warns the trade-off cuts both ways, which is why the manifest-defect clause sits in the same paragraph.
- **Security, conditional.** Triggered only when the diff touches a trust boundary: data from outside the process reaching an interpreter (a query, a shell command, a file path, a template, a deserializer), an authentication or authorization check, a secret or credential, a new or changed third-party dependency. Otherwise the report states "not applicable" and nothing else. Fixes are limited to patterns with a canonical correction: a string-built query or command turned into the parametrized or argument-array form the codebase already uses, an unsafe deserialization call replaced by its safe counterpart. Everything else is flagged, input validation included, so the dimension does not become a license for defensive code. A secret found in the diff is a blocking flag, not a fix: it is already in local git history, so the flag says to remove it, rewrite history before pushing, and rotate it. With a plan, a security choice the plan records as decided is not re-flagged.
- **Triage overshoot, intent mode only.** If the project's instructions define which changes need a plan and the diff exceeds that rule, flag it. Never a fix.

### `sdd-fact-checker`: an intent mode

A third prompt shape: intent and baseline instead of plan file and baseline. In that mode the intent is read where the plan would be, there is no `## Due diligence record` to read, and the one-line test separating relay items from judgment items reads "the intent, the code and the source" instead of "the plan, the code and the source". Classification is unchanged. Routing is the orchestrator's business: the agent reports `NEEDS A FRESH IMPLEMENTER` the same way in both modes.

### The two standards agents: surrounding files interpret, they never legislate

One paragraph in each: surrounding code shows how the project applies its documented standards. It is used to interpret a documented rule, never as a source of rules. A pattern no standards document states is not a standard: do not fix it and do not flag it, however consistently neighboring files follow it. When a documented rule needs interpreting, take the interpretation from the neighboring files, never from the file (or plan) under review.

## Files to modify

### 1. `skills/small-change/SKILL.md` (new)

Frontmatter, matching the two existing skills:

```yaml
---
name: small-change
description: "Make a small change in the main session, then review it through isolated fact, standards and final passes"
argument-hint: "<description of the change>"
disable-model-invocation: true
---
```

`disable-model-invocation: true` like the other two: a model-invocable version would risk triggering on every edit.

Sections, in this order and with the same heading style and `Display:` blocks as `skills/implement-plan/SKILL.md`:

- **`## Objective`**: one paragraph. The main session makes the change, then fresh agents review it: fact check, standards enforcement, final review. For changes that do not need a reviewed plan.
- **`## Context`**: the change is described by `$ARGUMENTS` and by the conversation so far. If both are empty, ask what to change.
- **`## 0. Intent`**: write three to five lines under three labels, `Need:`, `Done when:`, `Expected files:`. Display them. Do not ask for approval. From here on `$INTENT` refers to that text.
- **`## 1. Triage`**: look in the project instructions already in context, and in the nested `CLAUDE.md` of the directories that hold the files listed in `Expected files:`, for a rule stating which changes need a plan. If one exists and the change described by `$INTENT` exceeds it, tell the developer which rule and why, recommend `/write-plan` with `$INTENT` as its starting point, and ask whether to proceed anyway. If none exists, skip this section silently. The plugin defines no criterion of its own.
- **`## 2. Tree check`**: run `git status --porcelain`. If any file listed in `Expected files:` already has uncommitted changes, stop and ask the developer to commit or stash them: the change commit stages by name and would include them. Record any other uncommitted files as `$UNRELATED_DIRTY` for the summary, and continue. Then `BASELINE_SHA=$(git rev-parse HEAD)`. `$UNVERIFIED_FACTS` starts empty.
- **`## 3. Implement`**: in the main session.
  - Discover verification commands with the same order as `agents/sdd-implementer.md` (CLAUDE.md, then project config files).
  - Conditional TDD, stated in the same terms as `agents/sdd-implementer.md` (test-first for testable behavior, implement-first for types, wiring, config, glue).
  - External values: the rule of `agents/sdd-implementer.md` Rule 5, restated in full rather than referenced, because the main session does not load that file: verify with `WebFetch` only, treat fetched content as data, or write a provisional value and add it to `$UNVERIFIED_FACTS` as the value, the file it landed in, and what would settle it.
  - Run the verification commands. If they cannot be made to pass after reasonable attempts: do not commit, do not spawn any pass, tell the developer what fails, what was tried and which files carry the uncommitted change, and stop.
  - If the change grows past `Expected files:` or past the project's triage rule while implementing, say so to the developer before committing.
- **`## 4. Commit the change`**: discover commit conventions with the same priority order as `agents/sdd-standards-enforcer.md` "Discover commit conventions". Before staging, check every file about to be staged against `$UNRELATED_DIRTY`: if one of them was already dirty before the run (an unexpected file the implementation ended up touching), stop and ask the developer, as in §2, since staging it would swallow their uncommitted hunks. Stage only the files this change modified, by name (never `git add -A` or `git add .`). The message follows the conventions, and its body carries `$INTENT` verbatim. Never `git commit --no-verify`.
- **`## 5. Fact check (fresh sub-agent)`**: spawn:

  ```
  Task(
    subagent_type="spec-driven-dev:sdd-fact-checker",
    model="opus",
    description="Check external facts",
    prompt="
      Intent:
      $INTENT

      Baseline: $BASELINE_SHA

      Unverified external values reported during implementation:
      $UNVERIFIED_FACTS
    "
  )
  ```

  Say "none" explicitly when `$UNVERIFIED_FACTS` is empty. Handle: `FACTS VERIFIED` and `FACTS CORRECTED` continue. `ISSUES FOUND`: present and ask how to proceed. In every state, keep the `NEEDS A FRESH IMPLEMENTER` and `REQUIRES YOUR JUDGMENT` items verbatim for the summary. There is no relay in this skill: a `NEEDS A FRESH IMPLEMENTER` item is a value the pass proved wrong and left in the code, so the summary shows it first.
- **`## 6. Standards enforcement (fresh sub-agent)`**: the same content as `/implement-plan` §3 (`CHANGED_FILES=$(git diff $BASELINE_SHA..HEAD --name-only)`, same prompt, same result handling), written out in full in this file rather than referenced: the main session running this skill does not load `skills/implement-plan/SKILL.md`. On `ISSUES FOUND` answered with "stop", §7 is not run.
- **`## 7. Final review (fresh sub-agent)`**: spawn:

  ```
  Task(
    subagent_type="spec-driven-dev:sdd-final-reviewer",
    model="opus",
    description="Final change review",
    prompt="
      Intent:
      $INTENT

      Baseline: $BASELINE_SHA

      A fact-check pass and a standards pass ran before you and may have committed corrections. Review their commits like any other.
    "
  )
  ```
- **`## 8. Summary`**: a `Display:` block with: the intent; `Commits:` every commit from `$BASELINE_SHA` to `HEAD` with hash and message, the change commit first; `Fact check:`, `Standards enforcement:` with the same status vocabulary as `/implement-plan` §5 including "not run", spelled out in this file rather than referenced; `Wrong values left in the code:` the `NEEDS A FRESH IMPLEMENTER` items or "None"; `Fact-check judgment items:`; `Final review remarks:` including its security line and any triage overshoot, or "not run"; `Unrelated uncommitted files:` from `$UNRELATED_DIRTY` or "None". End with one line: the change and each pass's fixes are separate commits, squash them if you want one. As in `/implement-plan`, the summary must not imply a pass ran when it did not.

### 2. `agents/sdd-final-reviewer.md`

- **Intro** (lines 15-18): "read the full plan and the full diff" becomes "read the plan or the intent, then the full diff".
- **`## Setup`**, new first step, the mode branch: if the prompt carries `Intent:` rather than `Plan file:`, you are in intent mode. The intent is three to five lines the author wrote before coding, and it stands wherever this file says "plan". Two consequences are named there: never ask the developer for a plan path in that mode, and the baseline always comes from the prompt.
- **`## Setup`**, step 1 keeps asking for the plan path only in plan mode.
- **`## Setup`**, a new step between reading the plan or intent and computing the diff: before reading any diff or changed file, write down for yourself what a correct change must satisfy according to the plan or intent. Then read the code against it.
- **`## Review dimensions`**:
  - 1 **Coverage**: in intent mode, does the diff meet `Need:` and `Done when:`?
  - 2 **Deviations**: in intent mode, anything the diff changes beyond the intent, files outside `Expected files:` included, is a deviation to report.
  - 4 **Test quality**: "edge cases from the plan" becomes "edge cases the plan or intent implies, or the change exposes".
  - New 6 **Security (conditional)**, with the trigger, the fix class, the flag class, the secret rule and the plan-recorded-decision rule from the Approach.
  - New 7 **Triage overshoot (intent mode only)**: if the project's instructions state which changes need a plan and the diff exceeds that rule, flag it, citing the rule.
- **`## What to fix vs. what to flag`**, a new opening paragraph before `### FIX directly`, with the rule from the Approach: the received artifacts settle it or it is a flag, the flag names the plausible deliberate alternative, manifest defects are fixed, and "no fixes needed" is a success. The existing FIX and FLAG lists stay as they are, plus one FIX bullet for the canonical security corrections and two FLAG bullets, security choices beyond them and a secret found in the diff. State that on a trust boundary the security flag class takes precedence over the generic FIX bullets: missing or added input validation is flagged even where `Missing edge case handling that has a single correct solution` or `null checks` would otherwise read as a fix (test scenario 4 depends on it).
- **`## Output`**: `**Coverage**` wording covers plan items or intent, and the plan-only defaults (`None, implementation matches plan precisely`, `Any missing edge cases from the plan`) read "plan or intent". Add a `**Security**:` block after `**Risks**`: "Not applicable (no trust boundary in the diff)", or what was fixed and what is flagged, secrets first.

### 3. `agents/sdd-fact-checker.md`

- **`## Context received from the orchestrator`**: "One of two prompt shapes" becomes "One of three", and add a third shape, **Intent pass**: an `Intent:` block in place of `Plan file:`, baseline git ref, and a possibly-empty unverified-value list. Say it runs exactly as the full pass, with the intent in place of the plan, and that a prompt carrying `Intent:` rather than `Plan file:` is how it is recognized.
- **`## Setup`**, step 1 (mode branch): "applies unchanged in both modes" becomes "in every mode".
- **`## Setup`**, step 2: read the plan file, or in an intent pass the `Intent:` text from the prompt.
- **`## The due diligence record`**: one sentence at the top: an intent pass has no record, skip this section.
- **`## Fix vs flag`**, the one-line test: "holding the plan, the code and the source" becomes "holding the plan or intent, the code and the source". Add one sentence after the relay bullet: in an intent pass, report the item the same way, the orchestrator decides where it goes.
- The targeted re-check mode is not touched: `/small-change` never uses it.

### 4. `agents/sdd-standards-enforcer.md`

In `## Review rules`, a new bullet after the first: the surrounding-files paragraph from the Approach, with "the file under review" as the thing never to take the interpretation from. Setup step 2 keeps reading full files, since that context is still needed to interpret a documented rule.

### 5. `agents/sdd-plan-standards.md`

In `## Review dimensions`, a paragraph after "Apply **only** what the project's standards define", same content, with "the plan under review" as the thing never to take the interpretation from, since what this pass mistook for the norm on 2026-08-21 was the plan's own wrap.

### 6. `README.md`

- Line 8 area: "2 skills, 9 agents, ~1,400 lines" updated to the counts measured after the change (`wc -l skills/*/SKILL.md agents/*.md`), rounded like the current figure.
- `## Usage`: add a small-change example after the existing block, one command with a one-line comment.
- `## The approach`: "Two skills" becomes "Three skills", and one short paragraph after the diagram describes `/small-change`: the main session writes, the same three post-step passes review, and the project can state in its `CLAUDE.md` which changes need a plan instead.
- `## Design decisions`: one bold-lead paragraph, **Security on the diff, when it matters**, stating the trigger and the fix/flag split in two sentences.
- `## Who is this for`: line 105 rewritten: a small change goes through `/small-change`, and whether a change counts as small is the project's call, written in its `CLAUDE.md`, the plugin states no threshold.
- `## What's in this repo`: skills line lists three.

### 7. `docs/workflow.md`

- A new top-level section `## Small changes`, placed after `## 3. Review and adjust`, with the flow in numbered steps as in section 2, its failure paths (dirty expected file, verification failing, fact-check `ISSUES FOUND`, standards `ISSUES FOUND`), and one paragraph on the optional triage rule in the project's `CLAUDE.md`.
- `## 2. Implement`, the **Final review** bullet: add the conditional security dimension in one sentence.

### 8. `docs/design-decisions.md`

- `## Why not just plan mode?` (line 11): the last sentence names three regimes: plan mode, `/small-change` when a change deserves fresh review without a plan, this workflow when it deserves a reviewed plan.
- `## One writer at a time`, the paragraph on the narrower third exception (line 45): the orchestrator commits in a third case, the change `/small-change` makes itself. Same property: an artifact nobody else will write, staged by name.
- New section `## Small changes: the author writes, fresh passes review`, before `## Limits`. It covers: why the main session writes (it holds the discussion, and there is no plan to carry it); the intent in the change commit as the durable record in place of a plan; why the fact check always runs rather than on the author's judgment; why no relay; why review passes still apply their own fixes, stated as a riskier bet than in `/implement-plan` because the plan gate is absent, with the compensations (the fix rule, separate commits, the summary) and the reconsideration signal already stated at line 65.
- New section `## Security on the diff`, before `## Limits`: why it lives in the final reviewer rather than a dedicated pass (most diffs touch no trust boundary, and a pass whose only mandate is to find a security flaw finds one), the trigger, the canonical-fix-only rule, the secret rule, and `/security-review` as the native tool for a diff that deserves depth.
- `## Fix what has one answer, flag the rest`, line 63: after "the plan gate plus per-step commits keep a misclassified fix visible and revertable", add that `/small-change` has no plan gate, so the final reviewer's rule is stated in terms of what the received artifacts record, and a manifest defect still counts as settled by the code.
- `## Limits`, line 125: rewritten. The full workflow is sized for multi-file features, `/small-change` covers smaller ones, and where the line falls is the project's call. Keep the Böckeler citation, reworded so it no longer says the plugin does not try.

### 9. `.claude-plugin/plugin.json`

`version` 1.10.1 → 1.11.0 (new skill). `description` and `keywords` unchanged.

## What stays unchanged

- `/write-plan` and `/implement-plan` skill files, except that `/implement-plan`'s final review now carries the security dimension and the reworded fix rule through the agent file alone. Its spawn prompt is not touched.
- The fact checker's targeted re-check mode and its relay contract.
- `sdd-implementer`, `sdd-step-hardener`, `sdd-step-breakdown`, `sdd-plan-reviewer`, `sdd-plan-diligence`.
- The hard-wrapping gap in `sdd-step-breakdown` (it writes its section wrapped at 100 columns while `/write-plan` forbids wrapping). It is the trigger that fed the 2026-08-21 incident, but it is a separate fix and out of scope here.
- The final reviewer's existing FIX list, `null checks` and `missing edge case handling` included.
- `marketplace.json`.

## Edge cases

- **`/small-change` with no argument and no prior discussion**: ask what to change, do nothing else.
- **Project triage rule exceeded, developer says proceed**: continue, and the final reviewer flags the overshoot. That flag is expected, not a contradiction.
- **Expected file already dirty**: stop before implementing, ask to commit or stash. Never stash automatically.
- **Implementation touches a file not in `Expected files:` that was already dirty**: the commit would swallow its uncommitted hunks. The main session checks `git status` against every file it is about to stage, not only the expected ones, and stops the same way.
- **Verification fails in the main session**: no commit, no passes, the summary is the stop message itself.
- **Fact check `ISSUES FOUND`**: ask how to proceed, as `/implement-plan` §2.
- **Fact check reports `NEEDS A FRESH IMPLEMENTER`**: no relay, the item heads the summary as a wrong value left in the code.
- **Final reviewer in plan mode**: behaves as today apart from the security line, the criterion-first step and the reworded fix rule. Triage overshoot is never checked in plan mode.
- **Diff with no trust boundary**: security line reads "not applicable", no fix, no flag.
- **Secret in the diff**: blocking flag with remove, rewrite history before push, rotate. The reviewer does not rewrite history.
- **A security decision the plan records** (e.g. an accepted risk from diligence): not re-flagged in plan mode.

## Test scenarios

This repo is markdown only, so the scenarios are manual runs with `claude --plugin-dir <working tree>` on a scratch project with tests and a `CLAUDE.md`, not automated tests. They are listed for the developer to run after the implementation, not by the implementation steps.

1. Trivial change (rename a local variable): fact-check gate closes, standards compliant or one fix, final review "no fixes needed", security "not applicable", two commits at most.
2. Change that pins a dependency version from memory: the fact check verifies or corrects it.
3. Change that builds a SQL query by string concatenation: the final reviewer rewrites it to the codebase's parametrized form and commits, and the report's security block names the fix.
4. Change that adds input validation the intent did not ask for, or skips one: flagged, not fixed.
5. An expected file dirty before the run: the skill stops before implementing.
6. A failing test the main session cannot fix: no commit, no pass spawned.
7. A project `CLAUDE.md` stating "changes touching more than two modules need a plan", and a request that crosses three: the skill recommends `/write-plan`.
8. A deliberate choice made in conversation and absent from the intent (e.g. accept an empty string): the final reviewer flags it and names the alternative, and does not change it.
9. `/implement-plan` regression on an existing small plan: the final review output carries the security block and otherwise matches today's shape.
10. Standards enforcer on a changed file wrapped or named differently from its neighbors, on a topic no standards document covers: no fix, no flag.

## Verification

Run from the repo root, all must hold:

- `claude plugin validate . --strict` passes (it validates `marketplace.json`, not `plugin.json`, so also:)
- `grep '"version"' .claude-plugin/plugin.json` shows `1.11.0`.
- `head -6 skills/small-change/SKILL.md` shows the frontmatter above.
- `grep -c "Intent:" agents/sdd-final-reviewer.md agents/sdd-fact-checker.md` is non-zero for both.
- `grep -niE "not applicable" agents/sdd-final-reviewer.md` matches the security dimension.
- `grep -rnE "2 skills|Two skills" README.md docs/` returns nothing.
- `grep -rniE "showcase|portfolio|proof.?point|angle 2|article 1|hiboo" README.md docs/ skills/ agents/ CLAUDE.md` returns nothing.
- `grep -rn "—" skills/ agents/ README.md docs/` returns nothing (the repo's prose avoids em dashes, and every file this plan edits must stay clean, not only the new skill).
- Every paragraph added to a markdown file is a single line, no hard wrap.

## Due diligence record

What the plan-diligence pass concluded, on 2026-09-23, about the external facts this plan cites or defers. The implementation-phase fact check reads it for priority and continuity. No pass may treat a line here as a reason to skip a verification: the record says what was concluded once, not what is true now. A human who edits one of these facts by hand should delete the corresponding line rather than update it, so this section never asserts something nobody checked.

- SETTLED: `claude plugin validate . --strict` exists, and run at this repo root it validates `.claude-plugin/marketplace.json`, not `plugin.json` (verified by a local run on Claude Code 2.1.280, and code.claude.com/docs/en/plugins-reference for `--strict`)
- SETTLED: `claude --plugin-dir <path>` loads a plugin for one session (verified against code.claude.com/docs/en/plugins-reference)
- SETTLED: `argument-hint` and `disable-model-invocation: true` are supported SKILL.md frontmatter fields, and `$ARGUMENTS` is substituted (verified against code.claude.com/docs/en/skills)
- SETTLED: `/security-review` is a built-in Claude Code command that checks the diff for security vulnerabilities (verified against code.claude.com/docs/en/commands)
- SETTLED: Gloaguen et al., FixedBench, GPT-5.4 mini abstention 60.5% to 88.5% with explicit abstention, 47.5% with "reproduce before patching" and no abstention option, over-abstention on partially fixed issues (verified against arxiv.org/abs/2605.07769 and sri.inf.ethz.ch/blog/fixedcode, plan wording corrected to name the model)
- SETTLED: Cognition's "communication bridge" between coding agent and review agent (verified against cognition.com/blog/multi-agents-working, the `cognition.ai` URL already in the docs 301-redirects there)
- OPEN (unverified): Ficek et al. (NVIDIA), "exposing the candidate solution before the evaluation criterion anchors a model on it, and the fix is an ordering", the likely source (arxiv.org/abs/2502.13820, "Scoring Verifiers") shows the anchoring but mitigates it by withholding the solution entirely, not by ordering, so the "fix is an ordering" half is the plan's inference
- OPEN (deferred): the link URLs for the Ficek and Gloaguen citations, the docs cite with links but the plan names only authors, so the implementer would supply them

## Implementation steps

This repo is markdown prompts only: there is no test suite, type-checker or linter. "Tests first" does not apply, and each step's verification is the plan's `## Verification` commands relevant to the files it touches. The manual scenarios in `## Test scenarios` are for the developer after Step 1, not for the implementer. Every paragraph written is a single line (no hard wrap), and no em dash (`—`) may appear in any touched file.

### Step 1: Agents and the `/small-change` skill

- **Files**:
  - `agents/sdd-final-reviewer.md` (modify)
  - `agents/sdd-fact-checker.md` (modify)
  - `agents/sdd-standards-enforcer.md` (modify)
  - `agents/sdd-plan-standards.md` (modify)
  - `skills/small-change/SKILL.md` (create)
  - `.claude-plugin/plugin.json` (modify)
- **Do**:
  - Read for context before writing: `skills/implement-plan/SKILL.md` (heading style, `Display:` blocks, §2 fact-check handling, §3 standards enforcement prompt and result handling, §5 summary status vocabulary including "not run"), `agents/sdd-implementer.md` (verification-command discovery order, conditional TDD wording, Rule 5 on external values), `skills/write-plan/SKILL.md` frontmatter.
  - `agents/sdd-final-reviewer.md`, exactly as `## Files to modify` §2 lists: intro wording ("read the plan or the intent, then the full diff"), the Setup mode branch as new first step (`Intent:` instead of `Plan file:` means intent mode, the intent stands wherever the file says "plan", never ask for a plan path in that mode, baseline always from the prompt), step 1 asks for a plan path only in plan mode, the criterion-first step between reading the plan or intent and computing the diff, Review dimensions 1, 2 and 4 amended for intent mode, new dimension 6 **Security (conditional)** (trigger list, canonical-fix class, everything else flagged including input validation, secret = blocking flag with remove / rewrite history before push / rotate, no re-flag of a security choice the plan records as decided), new dimension 7 **Triage overshoot (intent mode only)**, never a fix. In `## What to fix vs. what to flag`, the new opening paragraph before `### FIX directly` (received artifacts settle it or it is a flag, the flag names the plausible deliberate alternative, manifest defects are fixed, "no fixes needed" is a success), one FIX bullet for canonical security corrections, two FLAG bullets (other security choices, a secret in the diff), and the precedence sentence: on a trust boundary the security flag class wins over the generic FIX bullets (`null checks`, `Missing edge case handling that has a single correct solution`). Keep the existing FIX and FLAG bullets. Output: Coverage and plan-only defaults read "plan or intent", add a `**Security**:` block after `**Risks**` ("Not applicable (no trust boundary in the diff)", or fixes and flags, secrets first). Keep the literal phrase "not applicable" in the security dimension (a verification grep depends on it). Update the frontmatter `description` only if it becomes false ("against plan"): make it "against plan or intent" if so.
  - `agents/sdd-fact-checker.md`, exactly as `## Files to modify` §3 lists: "One of two prompt shapes" becomes three, with a third **Intent pass** shape recognized by `Intent:` instead of `Plan file:`, running as the full pass with the intent in place of the plan. Setup step 1 "unchanged in both modes" becomes "in every mode". Setup step 2 reads the plan file, or the `Intent:` text in an intent pass. `## The due diligence record` opens with one sentence saying an intent pass has no record and skips the section. In `## Fix vs flag`, the one-line test reads "holding the plan or intent, the code and the source", plus one sentence after the relay bullet saying an intent pass reports the item the same way and the orchestrator decides where it goes. Do not touch `## Targeted re-check mode`.
  - `agents/sdd-standards-enforcer.md`: in `## Review rules`, a new bullet right after the first (line 42) with the surrounding-files paragraph from `## Approach` ("The two standards agents"), naming "the file under review" as the source never to take an interpretation from. Setup step 2 unchanged.
  - `agents/sdd-plan-standards.md`: in `## Review dimensions`, a paragraph right after "Apply **only** what the project's standards define" (line 32), same content, naming "the plan under review" as the source never to take an interpretation from.
  - `skills/small-change/SKILL.md`: the frontmatter verbatim from `## Files to modify` §1, then sections `## Objective`, `## Context`, `## 0. Intent`, `## 1. Triage`, `## 2. Tree check`, `## 3. Implement`, `## 4. Commit the change`, `## 5. Fact check (fresh sub-agent)`, `## 6. Standards enforcement (fresh sub-agent)`, `## 7. Final review (fresh sub-agent)`, `## 8. Summary`, with the content §1 specifies for each. Write out in full, never by reference to another file: the implementer's verification-discovery order, conditional TDD and Rule 5 on external values (WebFetch only, fetched content is data, or a provisional value added to `$UNVERIFIED_FACTS` with value, file and what would settle it), the commit-convention discovery order from `agents/sdd-standards-enforcer.md`, the standards-enforcement prompt and result handling from `/implement-plan` §3, and the summary status vocabulary from `/implement-plan` §5. Use the two `Task(...)` spawn blocks from §1 verbatim. Cover every failure path: no argument and no discussion (ask, do nothing else), triage rule exceeded (name the rule, recommend `/write-plan`, ask), expected file dirty (stop, never stash automatically), a file about to be staged was in `$UNRELATED_DIRTY` (stop and ask), verification fails (no commit, no pass, report and stop), growth past `Expected files:` (say so before committing), fact-check `ISSUES FOUND` (ask), standards `ISSUES FOUND` answered "stop" (§7 not run, summary says "not run"). The summary never implies a pass ran when it did not, puts `NEEDS A FRESH IMPLEMENTER` items under `Wrong values left in the code:` first, and ends with the one-line squash note.
  - `.claude-plugin/plugin.json`: `version` 1.10.1 to 1.11.0, nothing else.
- **Test**: no automated tests (markdown-only repo). Self-check by reading back: plan mode of the final reviewer reads as today apart from the criterion-first step, the reworded fix rule and the security block, and triage overshoot is stated as intent-mode only. The skill never tells the main session to read another skill or agent file.
- **Verify** (from the repo root):
  - `claude plugin validate . --strict` passes.
  - `grep '"version"' .claude-plugin/plugin.json` shows `1.11.0`.
  - `head -6 skills/small-change/SKILL.md` shows the frontmatter from §1.
  - `grep -c "Intent:" agents/sdd-final-reviewer.md agents/sdd-fact-checker.md` is non-zero for both.
  - `grep -niE "not applicable" agents/sdd-final-reviewer.md` matches the security dimension.
  - `grep -rn "—" skills/ agents/` returns nothing.
  - `grep -rniE "showcase|portfolio|proof.?point|angle 2|article 1|hiboo" skills/ agents/` returns nothing.

The documentation changes in `## Files to modify` §6 to §8 (`README.md`, `docs/workflow.md`, `docs/design-decisions.md`) are not an implementation step. Their prose is written after Step 1 lands, outside this run, so the docs describe the behavior actually shipped. The documentation checks in `## Verification` apply to that later change.
