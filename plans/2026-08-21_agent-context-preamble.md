# Plan: align the agent context preamble with how sub-agents actually start

## Context

The nine `agents/sdd-*.md` files open with an identical `<project_context>` block. Its first paragraph tells the agent that `@`-references in the root `CLAUDE.md` are "file imports that Claude Code resolves for the main session but NOT for sub-agents", and that the agent "MUST read each referenced file yourself".

Both claims are false. A sub-agent's starting context already contains the root `CLAUDE.md` and the inlined content of the files it imports with `@`. The official sub-agents documentation (`https://code.claude.com/docs/en/sub-agents`) lists "every level of the CLAUDE.md hierarchy the main conversation loads" as part of a sub-agent's starting context, with only the built-in Explore and Plan agents omitting it. This plugin ships no built-in agent, so the exception does not apply to any of its passes.

Measured on 2026-08-21 against a fixture holding three distinct control codes, one in the root `CLAUDE.md`, one in a file it imports with `@`, one in a nested `CLAUDE.md`. A sub-agent forbidden from calling any tool reported the root code and the imported code from its starting context, and reported the nested code absent. The measurement was run twice in the same session, once with a `general-purpose` sub-agent and once with this plugin's `sdd-plan-reviewer` resolved through `--plugin-dir`. Both returned identical results, which settles the earlier reserve that the behaviour might differ for a custom plugin agent.

The third claim of the block holds: nested `CLAUDE.md` files are not in the starting context and load only when the agent reads a file in their directory.

What this costs today: every pass is instructed to spend tool calls re-reading material it already holds, nine times per full run, in a plugin whose central argument is that each pass keeps a clean and cheap context. The instruction is also simply untrue, and it is shipped to users.

`docs/design-decisions.md` already describes the real platform behaviour, in the paragraph beginning "Isolation has a cost". The documentation is correct and the prompts are what must catch up. No documentation change is expected from this plan.

## Approach

Replace the `<project_context>` block in all nine agents with a version that states what the agent already holds and what it still has to read, then remove the redundant root-`CLAUDE.md` read instructions that survive elsewhere in each agent.

The instruction to stop re-reading has to be explicit rather than merely absent. Deleting the current paragraph would stop encouraging the wasted read without preventing it: orienting itself by reading the project's `CLAUDE.md` is a natural first move for a fresh agent. One sentence pointing the agent at what it holds does both jobs, and it also keeps the agent aware that the project's conventions are available to it.

That sentence describes where the content comes from and tells the agent to work from it. It does not assert that a given file is present. The difference matters in the two cases where the harness loads less than expected, a project with no `CLAUDE.md` and an `@`-import the user declined: an agent told to work from what is there simply finds less, exactly like the main session, while an agent told that a file is in its context would be holding a claim that is false. None of the three sites that mention the file, A, D and E, may presuppose it is present. A phrase like "the project's `CLAUDE.md` in your context" fails that test and must not be used at any of them.

Site A also carries a fallback: if the agent does not find the project instructions in its context, it reads the file itself. This is what makes the change safe to ship to users on Claude Code versions this repository has not measured. The behaviour is verified on one runtime on one date, against a doc page that can change, and the failure mode without a fallback is silent, because an agent cannot tell "the file was never injected" from "the file says nothing about this". Inspecting one's own context costs nothing, so the happy path still spends zero tool calls nine times per run, and the read fires only where it is the difference between holding the project's conventions and not.

The redundancy lives at five kinds of site, and they are not treated alike. The distinction that decides each one is whether the instruction asks the agent to **re-read** something already in its context, or to **consult** what it holds.

## Files to modify

All nine `agents/sdd-*.md`, plus `skills/write-plan/SKILL.md` and `.claude-plugin/plugin.json`.

### Site A: the `<project_context>` block (all nine agents)

The block is currently byte-identical across the nine files (verified: one md5 for the nine), and must stay byte-identical after the change. Each paragraph is a single unwrapped line, as everywhere else in these files, so keep it that way: hard-wrapping would break the line-based conformance greps below. Replace the block in full with:

```
<project_context>
Before starting your task, discover project context:

**Project instructions:** Claude Code loads the project's `CLAUDE.md`, and the content of the files it imports with `@`, into your context at startup. Work from what is there rather than spending tool calls to re-read it. If you do not find it there, read `./CLAUDE.md` yourself.

**Nested instructions:** Nested `CLAUDE.md` files are not loaded that way. They load only when you read a file in their directory, so identify the directories relevant to your task and read their `CLAUDE.md` yourself (e.g., `src/auth/CLAUDE.md`, `lib/payments/CLAUDE.md`), resolving any `@`-references those files contain. Follow all discovered conventions and constraints.
</project_context>
```

### Site B: the Setup step that reads the root `CLAUDE.md` (five agents)

Delete the step `Read ./CLAUDE.md at the project root (if it exists).` and renumber the steps that follow it so the list stays contiguous.

| File | Step to delete | Steps to renumber |
| --- | --- | --- |
| `agents/sdd-plan-reviewer.md` | 2 | 3, 4 become 2, 3 |
| `agents/sdd-plan-diligence.md` | 2 | 3, 4 become 2, 3 |
| `agents/sdd-step-breakdown.md` | 2 | 3, 4 become 2, 3 |
| `agents/sdd-implementer.md` | 3 | 4 becomes 3 |
| `agents/sdd-final-reviewer.md` | 4 | none, it is the last step |

### Site C: the Setup step that reads nested `CLAUDE.md` files (four agents)

Unchanged in wording. It survives in `sdd-plan-reviewer`, `sdd-plan-diligence`, `sdd-step-breakdown` and `sdd-implementer`, and only its number moves, per the table above.

### Site D: the standards-document lists (two agents)

`agents/sdd-plan-standards.md` (Setup step 2) and `agents/sdd-standards-enforcer.md` (Setup step 3) open their list of standards documents with two entries:

```
   - `./CLAUDE.md` (project root)
   - Nested `CLAUDE.md` files in directories the plan touches
```

Delete the first entry in both files. Keep the nested entry, keeping each file's existing wording for it (`directories the plan touches` in `sdd-plan-standards`, `directories containing changed files` in `sdd-standards-enforcer`). The remaining entries of both lists are untouched.

Deleting the entry is not enough on its own. These two passes exist to judge standards conformance, and after the deletion the project's primary standards document would be named nowhere in their prompt, which is the failure this plan's Approach warns about: an instruction that is merely absent. So extend each list's lead-in to name the file as a source the agent already holds. In `agents/sdd-plan-standards.md`, `Discover the project's coding and testing standards by reading these files (if they exist):` becomes:

```
2. Discover the project's coding and testing standards. Start from the project instructions already in your context, then read these files (if they exist):
```

and in `agents/sdd-standards-enforcer.md`, `Discover the project's coding standards by reading these files (if they exist):` becomes the same sentence without `and testing`, keeping its own step number 3.

These two agents are also told to cite their source, as in "per CLAUDE.md: use Result types for errors". That instruction stays: the agent holds the file's content and can cite it without reading it.

### Site E: the "Discover commit conventions" section (four committing agents)

`agents/sdd-step-hardener.md`, `agents/sdd-fact-checker.md`, `agents/sdd-standards-enforcer.md` and `agents/sdd-final-reviewer.md` each open that section with:

```
1. **CLAUDE.md conventions**: Re-read `./CLAUDE.md` (and any nested `CLAUDE.md` in relevant directories). Look for commit-related instructions: ...
```

Replace the leading directive in all four, keeping each file's existing tail verbatim from `Look for commit-related instructions:` onward, since `sdd-final-reviewer` omits the `(e.g., "no git add -A")` example the other three carry:

```
1. **CLAUDE.md conventions**: Use the project instructions already in your context, and read any nested `CLAUDE.md` in relevant directories. Look for commit-related instructions: ...
```

### Site F: the "Discover verification commands" sections (six agents), unchanged

`Check CLAUDE.md files for documented commands` asks the agent to consult what it holds, not to re-read it, so it is correct as written and stays. This is the one site where the audit could plausibly go further and should not.

### Site G: `skills/write-plan/SKILL.md` (one line)

Line 20, step 1 of "Explore the codebase", reads `1. Read \`./CLAUDE.md\` at the project root (if it exists).`. Delete it and renumber the two steps that follow, so the list runs 1 to 2.

This is the same wasted read as Site B, minus the false claim: the skill instructs the orchestrator, which is the main session, and the main session has always had that file loaded. `CLAUDE.md` is still reachable to the orchestrator, and step 2 of that list already has it exploring the codebase. `skills/implement-plan/SKILL.md` has no equivalent step and is not touched.

### `.claude-plugin/plugin.json`

Bump `version` from `1.9.2` to `1.9.3`. `CLAUDE.md` requires incrementing `version` on every functional change to skills or agents, and this change rewrites the startup instructions of all nine agents plus one skill.

**Decided: PATCH, 1.9.3.** `CLAUDE.md` requires an increment but names no semver rule, so the rule is whatever the repository actually does. Since 1.8.0 every single version transition has taken the PATCH except one, across `feat`, `fix`, `refactor` and `chore` alike. The exception is 1.9.0 (`c679dd1`), and no rule separates it from its neighbours: it added a new distributed agent (`agents/sdd-fact-checker.md`), but so did `a563984`, which added `agents/sdd-plan-diligence.md` and took the PATCH 1.8.6. An earlier draft of this plan inferred "new agent gives MINOR, everything else gives PATCH" and that inference is false, over-fit to 1.9.0 and contradicted by 1.8.6. The defensible reading is narrower: this repository bumps the PATCH by default and 1.9.0 is unexplained. This change adds no agent and is not the exception, so it takes 1.9.3. The breadth-of-impact argument for 1.10.0 was considered and dropped, being an argument the repository has never applied, on a number that is publicly visible on the Releases page.

## What stays unchanged

- `docs/design-decisions.md`, `docs/workflow.md` and `README.md`. The design decisions page already describes the real behaviour and stays accurate after this change. Confirm rather than assume.
- Site C and Site F, as described above.
- The `skills:`, `model:` and `description:` frontmatter of every agent.
- Everything in the nine agents outside the five sites, including every output contract.
- `skills/implement-plan/SKILL.md`. It instructs the orchestrator, which is the main session, and it carries neither the false claim about `@`-imports nor a root-`CLAUDE.md` read.
- Everything in `skills/write-plan/SKILL.md` except the one line of Site G.

**Decided: in scope, see Site G.** `skills/write-plan/SKILL.md` line 20 carries the same redundant read, minus the false claim, and the argument this plan makes against it in the agents applies unchanged to the orchestrator. Leaving it would ship one instruction to re-read what is already loaded and guarantee that a later audit finds it again. The cost is accepted: the scope becomes eleven files and the scope check below expects a skill in the diff.

## Edge cases

- **A project with no `CLAUDE.md`.** The preamble describes what the harness loads and tells the agent to work from it, rather than asserting a file is present, so with nothing loaded there is nothing to work from and the sentence is inert rather than false. The fallback then finds no file either, and the agent proceeds, so no conditional is needed for this case.
- **An `@`-import the user declined.** An import resolving outside the working directory triggers a one-time approval, and a declined import stays disabled without re-prompting. Its content is then in nobody's context, the main session included. The preamble handles this by construction for the same reason as the case above: the agent works from what is there and finds less, exactly like the session that spawned it. This is why the phrasing describes the loading rather than asserting the result, and it is the case that would have broken an "it is already in your context" wording.
- **A project whose root `CLAUDE.md` imports a file that does not exist.** Unresolvable imports are the harness's problem, and no agent instruction changes the outcome. Out of scope.
- **A nested `CLAUDE.md` that the harness loaded on its own**, because the agent happened to read a file in that directory before reaching the instruction. Reading it explicitly is then redundant but harmless, and the agent cannot know which case it is in. Accept the redundancy rather than instructing a check that costs more than the read.
- **A runtime that does not inject the hierarchy.** The plugin installs from a marketplace and declares no minimum Claude Code version, so it runs on builds this repository has never measured, and the evidence for the injection is one measurement on one runtime plus a doc page that can change. This case is not like the two above: the file exists and holds the project's conventions, and every pass would silently run without them, since an agent cannot distinguish an uninjected file from a file that says nothing on the point. It is also asymmetric against the change, because the instruction being removed is merely redundant if the assumption holds, while its replacement actively suppresses the read. This is the case Site A's fallback exists for, and it is why the fallback is not optional.
- **Renumbering.** Sites B and C interact: four agents lose a step and keep the next one. A list left with a duplicated or skipped number is the most likely mechanical error in this change, which is why the verification greps for it.

## Test scenarios

Markdown-prompt behavior, no automated runner. Verify by driving the passes with `--plugin-dir` pointed at the working tree, against the fixture the Context measurement used: a root `CLAUDE.md`, a file it imports with `@`, and a nested `CLAUDE.md` in a subdirectory, each holding a distinct control code.

1. **The root file is used without being read.** `sdd-plan-standards`, run on a plan touching the fixture, cites the root control code and the imported one, and its transcript carries no tool call reading `./CLAUDE.md` or the file it imports.
2. **The nested file is still read.** The same pass, run on a plan touching the fixture's subdirectory, reads that subdirectory's `CLAUDE.md` itself and cites its control code.
3. **A committing pass keeps its convention discovery.** `sdd-standards-enforcer`, run over a diff in the fixture's subdirectory, discovers the commit convention from the root `CLAUDE.md` it already holds, reads the nested `CLAUDE.md`, and commits following the discovered convention.
4. **A project with no root `CLAUDE.md`.** Every pass proceeds without reporting a missing file and without hunting for one, and the nested instruction still fires where a nested file exists.
5. **The renumbered Setup lists still execute in order.** In the five agents of Site B, the pass runs the remaining Setup steps in sequence, with nothing skipped where a number was removed.

The conformance greps that assert the text itself, one per site, live in `## Verification` below.

## Verification

No automated tests (markdown-only repo). Commands 1 to 6 are the conformance greps over the nine agent files, one per site, each with an expected result. Command 7 is the repository-convention check. None of them covers the behavioural scenarios above, which are driven manually.

```bash
# 1. preamble uniform across the nine agents (expect a single checksum)
for f in agents/sdd-*.md; do sed -n '/<project_context>/,/<\/project_context>/p' "$f" | md5sum; done | sort -u

# 2. no root re-read survives (expect no output for all three)
grep -rn 'Read `./CLAUDE.md` at the project root' agents/   # 14 matches before the change
grep -rn 'Re-read `./CLAUDE.md`' agents/                    # 4 matches before the change
grep -rnF '`./CLAUDE.md` (project root)' agents/            # 2 matches before the change, site D
grep -rn 'Read `./CLAUDE.md` at the project root' skills/   # 1 match before the change, site G

# 3. the nested instruction survives (expect 9, then 4, then 4)
grep -rc 'read their `CLAUDE.md`' agents/ | grep -c ':1$'                        # site A, one per agent
grep -rc 'check for and read any nested `CLAUDE.md` files' agents/ | grep -c ':1$'  # site C
grep -rc 'read any nested `CLAUDE.md` in relevant directories' agents/ | grep -c ':1$'  # site E

# 4. setup lists contiguous (inspect each, expect 1..n with no gap)
for f in agents/sdd-*.md; do echo "== $f"; sed -n '/^## Setup/,/^## /p' "$f" | grep -E '^[0-9]+\.'; done

# 5. site F untouched (expect 6, the same count as before the change)
grep -rc 'Check `CLAUDE.md` files for documented commands' agents/ | grep -c ':1$'

# 6. scope (expect exactly eleven files: the nine agents, skills/write-plan/SKILL.md, plugin.json)
#    nothing under docs/, not README.md, not skills/implement-plan/SKILL.md, and not the plan file,
#    which is committed separately before this step runs
git diff --stat            # once anything is committed, use `git diff --stat $BASELINE..HEAD`

# 7. repository conventions
grep -rn '—' agents/ skills/write-plan/SKILL.md   # expect no output, no em dashes (0 before)
claude plugin validate . --strict
```

The `claude plugin validate --strict` run is the only check that exercises the plugin machinery rather than the text. It confirms the nine agents still parse as agent definitions after the edit.

Manual: point the plugin at the working tree with `--plugin-dir` (the runtime loads the installed build from its version cache, so local edits do not take effect otherwise) and drive test scenarios 1 to 4 against the fixture.

## Implementation steps

Repo verification: markdown only, no test/lint/typecheck runner exists. The available checks are the conformance greps of `## Verification` above and `claude plugin validate . --strict`. Commits follow `CLAUDE.md` (`/commit`, conventional commits, files staged by name, never `git add -A`).

One step. The change is eleven files but a single atomic commit: the nine agents, one line of `skills/write-plan/SKILL.md`, and the version bump `CLAUDE.md` requires with them. Splitting it would leave an intermediate state where verification command 1 (one md5 for the nine preambles) fails by construction, and the total surface is small (848 lines across the nine agents, five edit sites, no file read for context beyond the ten being edited).

Both `<!-- REVIEW -->` markers the review passes left have been resolved by the human and are recorded as decisions in the plan body: the version takes the PATCH (`1.9.3`), and `skills/write-plan/SKILL.md` is in scope as Site G. No marker is open, nothing in this step is waiting on a ruling.

### Step 1: Align the agent context preamble with sub-agent startup context

**Files**

- `agents/sdd-fact-checker.md` (modify)
- `agents/sdd-final-reviewer.md` (modify)
- `agents/sdd-implementer.md` (modify)
- `agents/sdd-plan-diligence.md` (modify)
- `agents/sdd-plan-reviewer.md` (modify)
- `agents/sdd-plan-standards.md` (modify)
- `agents/sdd-standards-enforcer.md` (modify)
- `agents/sdd-step-breakdown.md` (modify)
- `agents/sdd-step-hardener.md` (modify)
- `skills/write-plan/SKILL.md` (modify, one line)
- `.claude-plugin/plugin.json` (modify)

**Do**

Work site by site, not file by file: each site has one target text and applying it across its files in one pass is what keeps the result uniform. Every paragraph stays a single unwrapped line, as everywhere else in these files. Hard-wrapping any of the new text breaks the line-based greps below.

1. **Site A, the `<project_context>` block, all nine agents.** Replace the block in full with the version quoted under `### Site A` of `## Files to modify`. It must land byte-identical in the nine files, so write one canonical version and apply it, rather than retyping it per file. The block currently spans from the `<project_context>` line to the `</project_context>` line inclusive and is byte-identical across the nine today (one md5, verified).
2. **Site B, the Setup step that reads the root file, five agents.** Delete the step `Read \`./CLAUDE.md\` at the project root (if it exists).` in `sdd-plan-reviewer` (step 2), `sdd-plan-diligence` (step 2), `sdd-step-breakdown` (step 2), `sdd-implementer` (step 3) and `sdd-final-reviewer` (step 4), then renumber the following steps per the table in `### Site B` so each list stays contiguous from 1. In `sdd-final-reviewer` the deleted step is last, so nothing renumbers there. The other four each shift exactly the steps after the hole.
3. **Site C, the nested-`CLAUDE.md` Setup step, four agents.** Wording untouched in `sdd-plan-reviewer`, `sdd-plan-diligence`, `sdd-step-breakdown` and `sdd-implementer`. Only its number moves, as a consequence of step 2.
4. **Site D, the standards-document lists, two agents.** In `agents/sdd-plan-standards.md` (Setup step 2) and `agents/sdd-standards-enforcer.md` (Setup step 3), delete the list entry `` - `./CLAUDE.md` (project root) `` and rewrite each list's lead-in so the file is still named as a source, per `### Site D` of `## Files to modify`: `Discover the project's coding and testing standards. Start from the project instructions already in your context, then read these files (if they exist):` in `sdd-plan-standards`, and the same sentence without `and testing` in `sdd-standards-enforcer`. Deleting the entry without the lead-in leaves the two standards passes with their primary standards document named nowhere, which is the failure mode this plan exists to avoid. Keep the nested entry immediately below with each file's existing wording (`directories the plan touches` in `sdd-plan-standards`, `directories containing changed files` in `sdd-standards-enforcer`) and leave the remaining entries alone. Both agents' instruction to cite the source ("per CLAUDE.md: ...") stays as written.
5. **Site E, the "Discover commit conventions" leading directive, four agents.** In `sdd-step-hardener`, `sdd-fact-checker`, `sdd-standards-enforcer` and `sdd-final-reviewer`, replace `Re-read \`./CLAUDE.md\` (and any nested \`CLAUDE.md\` in relevant directories).` with `Use the project instructions already in your context, and read any nested \`CLAUDE.md\` in relevant directories.`, keeping each file's tail verbatim from `Look for commit-related instructions:` onward. The tails differ: `sdd-final-reviewer` omits the `(e.g., "no git add -A")` example the other three carry, so a single blanket line replacement across the four would silently rewrite it.
6. **Site F, the "Discover verification commands" sections, six agents.** Untouched. `Check \`CLAUDE.md\` files for documented commands` asks the agent to consult what it holds, not to re-read it.
7. **Site G, `skills/write-plan/SKILL.md`.** Delete line 20, step 1 of "Explore the codebase" (`Read \`./CLAUDE.md\` at the project root (if it exists).`), and renumber the two steps that follow so the list runs 1 to 2. Nothing else in that file changes.
8. **`.claude-plugin/plugin.json`.** `version`: `1.9.2` to `1.9.3`.

Nothing else in the nine agents changes: not the `name:`, `description:`, `skills:` or `model:` frontmatter, not any output contract. No em dashes in any file touched. Do not touch `docs/`, `README.md`, or `skills/implement-plan/SKILL.md`.

**Test**

No automated runner. Two layers of checking, and only the first is mechanical.

Conformance, from `## Verification` above: run commands 1 to 5 and 7 and confirm each expected result. Command 4 is an inspection, not a pass/fail, so read the nine printed Setup lists and confirm each numbers 1..n with no gap and no duplicate. The renumbering is the most likely mechanical error in this change.

Contract reading, against `## Test scenarios` and `## Edge cases` above: read the finished `<project_context>` block and confirm it gives one unambiguous instruction path for each of the five scenarios, in particular that scenario 1 (use the root file without reading it) and scenario 2 (still read the nested file) cannot both be satisfied by the same action, and that scenario 4 (no root `CLAUDE.md`) leaves the agent nothing to hunt for. Then read the four Site E sections in full and confirm each still reaches its commit-convention discovery with its own tail intact.

**Verify**

```bash
# site A uniform (expect a single checksum line)
for f in agents/sdd-*.md; do sed -n '/<project_context>/,/<\/project_context>/p' "$f" | md5sum; done | sort -u

# no root re-read survives (expect no output from all three)
grep -rn 'Read `./CLAUDE.md` at the project root' agents/   # 14 before
grep -rn 'Re-read `./CLAUDE.md`' agents/                    # 4 before
grep -rnF '`./CLAUDE.md` (project root)' agents/            # 2 before
grep -rn 'Read `./CLAUDE.md` at the project root' skills/   # 1 before, site G

# nested instruction survives (expect 9, then 4, then 4)
grep -rc 'read their `CLAUDE.md`' agents/ | grep -c ':1$'
grep -rc 'check for and read any nested `CLAUDE.md` files' agents/ | grep -c ':1$'
grep -rc 'read any nested `CLAUDE.md` in relevant directories' agents/ | grep -c ':1$'

# setup lists contiguous (inspect: 1..n, no gap, no duplicate)
for f in agents/sdd-*.md; do echo "== $f"; sed -n '/^## Setup/,/^## /p' "$f" | grep -E '^[0-9]+\.'; done

# site F untouched (expect 6)
grep -rc 'Check `CLAUDE.md` files for documented commands' agents/ | grep -c ':1$'

# scope (expect exactly eleven files: nine agents, skills/write-plan/SKILL.md, plugin.json)
# nothing under docs/, not README.md, not skills/implement-plan/SKILL.md, not the plan file
git diff --stat

# repository conventions
grep -rn '—' agents/ skills/write-plan/SKILL.md   # expect no output
claude plugin validate . --strict
```

Manual, not part of the step's exit criteria: point the plugin at the working tree with `--plugin-dir` (the runtime otherwise loads the installed build from its version cache) and drive test scenarios 1 to 4 against the fixture described in `## Test scenarios`.
