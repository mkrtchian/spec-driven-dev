# Plan: route an out-of-bound fact correction to a fresh implementer instead of the summary

## Context

`sdd-fact-checker` bounds its writes to the value itself. A fact it confirms wrong whose correction would exceed that bound is therefore not applied: it goes to `REQUIRES YOUR JUDGMENT`, and the run continues through the standards enforcer and the final review. The branch that lands carries a value the pass holds proof is false, and the only trace is one line in the end-of-run summary.

The defect is not the write bound, it is the criterion that triggers the escalation. The current wording, "the true value changes a contract, a signature, or the shape of the code around it", measures the size of the edit rather than the presence of a judgment call, and the two come apart in both directions. An API field the code calls `user.email` that is really `user.contact.email` exceeds the literal and requires no decision at all. An endpoint that returns a paginated list where the code expects an array, or a function that gained a required third argument, force a choice the artifacts do not settle. The criterion that separates them is the one `docs/design-decisions.md` already states, "whether a finding has a single right answer that the artifacts hold", and the prompt substitutes size for it.

Correcting the criterion alone would not touch that defect. The class would still land in `REQUIRES YOUR JUDGMENT`, the value would still be committed wrong, and the run would still continue. What the fix alone buys is a bucket that no longer mislabels its contents, since it would be holding findings that need no judgment at all. So the criterion and the route are one change: the first without the second classifies better and repairs nothing.

**No instance of this has been observed.** The route is argued from the criterion, not from a run that went wrong, and the repo's own precedent is stricter: the plan that added this pass was written on eight fabricated CVSS scores of which six were wrong. The path also fires only where four conditions meet, a diff carrying an external fact, that fact confirmed wrong, its correction exceeding the value itself, and a single right answer still available. How often that lands is unknown, and nothing here should be read as claiming it is common or rare.

## Approach

Give that class a route, mirroring the relay the workflow already has. Where the step hardener does fresh implementer then re-hardening, the post-step fact check does fact check, fresh implementer, targeted re-check. The orchestrator routes and no agent spawns another.

The three-way split replaces the two-way one:

| The pass confirms a fact wrong, and | Route |
| --- | --- |
| the correction fits its write bound | fix in place, verify, commit (unchanged) |
| the correction exceeds the bound but has a single right answer the artifacts hold | `NEEDS A FRESH IMPLEMENTER` (new) |
| the correction requires choosing a value or a strategy | `REQUIRES YOUR JUDGMENT` (narrowed to its true class) |

Four properties the shape preserves. The fact checker's write bound is untouched. The correction is made by the agent whose job is writing code, in the context that job needs. Everything happens before the standards enforcer and the final review, so the property gained by placing the fact check first still holds and the relay's commit is read by two later passes. And a conclusion that crosses a context boundary is re-established rather than inherited: the relay implementer does act on the fact check's conclusion, which is the whole point of the route, but the targeted re-check behind it treats the relayed value and source as a pointer to where to look rather than as a result, exactly as the full pass treats a `SETTLED:` line in the plan's `## Due diligence record`. Say it that way and not as "no agent inherits another agent's conclusion", which this route makes false.

The signal is a report section rather than a fourth state, because the relay is orthogonal to what the state encodes. A pass can commit an in-bound correction and need a relay in the same run, and a fourth state would have to duplicate the body of `FACTS CORRECTED` to express that. The precedent is in the same file: `REQUIRES YOUR JUDGMENT` is already a section that every state carries for the same reason.

One relay per run, then the run stops and asks. The targeted re-check is scoped to the implementer's uncommitted diff, so it cannot surface a fact the first pass did not see, and a second attempt would replay the same prompt with the same prior. Making the second attempt better would mean feeding it what the re-check found, which is the relay-of-a-relay the design refuses. So the guard is written into the agent itself: in targeted mode it never emits `NEEDS A FRESH IMPLEMENTER`.

## Files to modify

### 1. `agents/sdd-fact-checker.md`

**`## Context received from the orchestrator`.** It currently names one prompt shape: plan path, baseline ref, and the unverified-value list. Name the second one beside it, the targeted re-check prompt (plan path, baseline ref, and a `## Targeted re-check` list), so the mode branch below has something to branch on that the file has already described.

**Mode branch, at the top of `## Setup`.** Add a first step: if the orchestrator's prompt carries a `## Targeted re-check` section, follow `## Targeted re-check mode` instead of the rest of `## Setup`, `## The gate` and `## The due diligence record`. Everything else in the file (verification rule and web hygiene, verification-command discovery, commit-convention discovery, staging by name, no `--no-verify`) applies unchanged in both modes.

**`## Fix vs flag`, fourth bullet.** Replace the single bullet keyed on edit size with two bullets keyed on whether the artifacts hold one answer. The out-of-bound-but-single-answer bullet directs the finding to `NEEDS A FRESH IMPLEMENTER`, carrying the true value, the source that settles it, and the file the false value landed in, and states that the pass does not apply it and does not send it to the human. The requires-a-choice bullet keeps `REQUIRES YOUR JUDGMENT` with the true value and the source. Both bullets state the test in one line: whether a fresh agent holding the plan, the code and the source could reach one answer with no decision only the developer can make. Keep the two bullets above it (not-the-latest, cannot-be-verified) as they are.

**`## Output` contract.** Change the lead-in so all three states carry both a `**NEEDS A FRESH IMPLEMENTER:**` section and the existing `**REQUIRES YOUR JUDGMENT:**` section, each defaulting to "none". Update the `REQUIRES YOUR JUDGMENT` inventory, which currently ends in "wrong facts whose correction is not a minimal edit", so it names the narrowed class instead. The same inventory is repeated inside each of the three state bodies as a bullet reading "Currency notes, unverified facts, non-minimal corrections, ..." (`FACTS CORRECTED` adds "reverted fixes"). Reword all three the same way: leaving them behind would contradict the new lead-in in three places.

**`### FACTS VERIFIED` body.** Its current first line is "Nothing needed fixing", which becomes false when the only wrong fact is one this pass may not fix. Redefine the two continuing states by whether the pass committed: `FACTS VERIFIED` means it committed nothing, whether because the gate closed, or every fact checked out, or the only wrong facts are listed for another pass or for the developer. `FACTS CORRECTED` means the pass committed, whether the corrections in that commit were its own (full mode) or the relay implementer's that it verified (targeted mode).

**New `## Targeted re-check mode` section.** Its scope is the uncommitted diff the relay implementer left (`git diff` and `git diff --cached`), not the run diff since baseline. It does three things. It re-verifies each value named in the `## Targeted re-check` list against its live source, treating the relayed value and source as a pointer rather than a settled result, for the reason the record section already gives. It gates the implementer's diff for external facts the implementer introduced that were not on the list, and verifies those too, so an invented adjacent value does not pass unread by any pass. Then it runs the discovered verification commands, stages by name and commits, returning `FACTS CORRECTED`.

The section also carries the guard and the three ways it stops. It never emits `NEEDS A FRESH IMPLEMENTER`, whatever it finds. A value still wrong, or a new false fact in the implementer's diff, returns `ISSUES FOUND` with no commit, naming what is still wrong and what the true value is. Verification that fails and cannot be repaired returns `ISSUES FOUND` with no commit. And a source it cannot reach also returns `ISSUES FOUND`: this is a deliberate divergence from the full pass, where an unverifiable fact rides along as a judgment item, because committing here would assert a verification that did not happen on the one value the pass exists to confirm. Note the divergence in the file next to the existing sentence it contradicts, so the two do not read as a contradiction.

### 2. `skills/implement-plan/SKILL.md`

**Section 0.** Add `$RELAY_FACTS` alongside `$UNVERIFIED_FACTS` as a variable the run carries, empty at the start.

**Section 1b, the "one relay" sentence.** The fix branch currently reads "This is the workflow's one relay of a pass's conclusions to another pass to apply". That stops being true the moment section 2a exists. Reword it to name the two relays and what they share (a fresh implementer applies, a fresh pass verifies behind it), so the file does not assert a uniqueness the same file contradicts two sections later.

**Section 2, handling of the fact check result.** Keep the three existing outcomes and add the routing. On `FACTS VERIFIED` or `FACTS CORRECTED` with an empty `NEEDS A FRESH IMPLEMENTER` section, continue to section 3 as today. With a non-empty one, record its items verbatim into `$RELAY_FACTS` and go to section 2a. On `ISSUES FOUND`, present and ask as today, and present any relay items as part of what the developer sees rather than routing them: that state means the tree is in a condition no automated correction should build on.

Bound what enters the orchestrator, in the same words the section already uses for `$UNVERIFIED_FACTS`: the true value, the source, and the file the false value landed in, and nothing else from the report.

**New section 2a, relay to a fresh implementer.** Spawn `spec-driven-dev:sdd-implementer` on the template of the section 1b fix path, with a `## State of the tree` block and the relay items verbatim. Both new sections follow the spawn shape the file already uses everywhere: a `Display:` block with a `--- ... ---` header naming the pass, then a `Task(...)` call carrying `model="opus"` like the six already there. The prompt states four things. The fact check already committed the corrections that fit its own write bound, so the tree is clean at the start. Each item carries a value already verified against the source quoted beside it, so the implementer must not spend web calls re-verifying it. The edit is bounded to what the true value requires and nothing else. And it must not commit.

The section 1b template cannot be copied as-is, because this relay has no step. `agents/sdd-implementer.md` `## Setup` step 1 reads "If step content was provided in context, use that directly. Otherwise, read the plan file at `$ARGUMENTS` and implement it as a whole." A prompt carrying only a plan path and a findings list would trip that fallback and re-implement the whole plan. So the relay items must be presented as the work itself, under a heading that reads as the scope of the task (`## Corrections to apply`), with the prompt saying in one line that these corrections are the entire scope and that the plan path below is context only.

Handle `IMPLEMENTATION BLOCKED` exactly as section 1a does, presenting the block to the developer and asking how to proceed, without running the re-check on a blocked tree. On `IMPLEMENTATION COMPLETE`, carry any `Unverified external value:` bullets this implementer reports into the section 2b prompt rather than into `$UNVERIFIED_FACTS`, which section 2 has already consumed. The section 1a sentence "This applies to every implementer invocation, including the extra implementer spawned on the §1b fix path" must be extended to name this third invocation and its different destination, so no report bullet falls between the two.

**New section 2b, targeted re-check.** Spawn `spec-driven-dev:sdd-fact-checker` again with the same `Plan file:` and `Baseline:` lines section 2 uses, plus a `## Targeted re-check` section carrying `$RELAY_FACTS` verbatim and any unverified values the relay implementer declared. The baseline still travels even though targeted mode does not diff against it, so the two spawns share one prompt shape and the agent's context section stays true. Two outcomes only. `FACTS CORRECTED` notes the commit and continues to section 3. `ISSUES FOUND` **ends the run**: present the issues to the developer, state that the relay implementer's work is still uncommitted in the tree, and go straight to the section 5 summary. There is no continue branch and no second relay.

That is the one place this skill presents an `ISSUES FOUND` without asking how to proceed, so say why in the skill itself. `ISSUES FOUND` from the re-check means one specific thing, that the pass holds proof the value is still wrong, unlike section 1b's mixed bag of issues a hardener could not fix. Committing over a proven-false value is the exact defect this route exists to remove, so a skip branch would reopen it one layer down with the developer's signature on it. And the two remaining passes diff from `$BASELINE_SHA`, so an uncommitted relay diff is invisible to both: continuing would end the run reporting success over work nothing read.

**Section 5 summary.** Add the relay to the `Fact check:` status line alternatives. Reword `Fact-check judgment items:` so it names the narrowed class instead of "non-minimal corrections". Add a line reporting the relay when one happened: what was relayed, and whether the re-check committed it or ended the run. Section 2b is a third way the run can stop short, so extend the closing paragraph that already lists the section 1b "stop" branch and a fact-check `ISSUES FOUND` answered with "stop": on this path the standards enforcer and the final review never run and their lines read `not run`. The summary must then name the working tree explicitly: that it still carries the relay implementer's uncommitted edits, which files they touch, and that nothing has verified them. An uncommitted tree the run announces is not hidden state, it is reported state, and that is the whole difference. A run that ended this way and said nothing would be the failure the plan is written against, one level down.

### 3. `docs/design-decisions.md`

**"A lightweight orchestrator", the bounded-carry sentence.** It currently names one bounded carry, the implementer's unverified value. Name the second one beside it, the relay triple, so the rule stays a rule with an enumerated set rather than a single example.

**"One writer at a time", the parenthetical.** It reads "One qualified exception" followed by "A second, narrower one" of a different kind. The relay is a second instance of the first kind, so the count changes: two exceptions of the relay form, the hardener fix path and the fact-check relay, then the orchestrator's own commits as the narrower third. State what makes the fact-check relay the same shape, a finding with one right answer handed to the agent that can apply it, with a fresh verification pass behind it.

**And name where the two relays are not the same, in the same breath.** The existing text says of the hardener path that "**you choose** between fix, skip, and stop". That relay is authorized by a human every time it fires. The fact-check relay is not: the orchestrator routes it on its own. Calling the two "the same shape" without saying this would flatten the one property that made the first exception cheap to grant, so the paragraph has to carry both halves. The reason the second is granted without a gate is that its class is defined by having a single right answer, which is by construction the class that needs no decision, and asking about it is the alert fatigue the single-touchpoint design refuses. That is an argument to make in the open, not a difference to leave unmentioned.

**"One writer at a time", the sentence on the hardener exception taking the second path.** Extend it to cover both relays, and say why the fact-check relay is the cheaper of the two to justify: the class it relays is defined by having a single right answer, which is exactly the class the section says survives a context boundary intact.

**"One writer at a time", the closing paragraph on what the dividing line allows.** It reads "read-only reviewers can classify their own findings and hand the single-answer ones to one writer that applies them. This workflow has no use for that shape". The reason given is about running reviewers in parallel, which the relay does not do, but the first sentence describes the relay almost word for word. Narrow it to what it means, the parallel-reviewers shape, so the paragraph does not read as denying the route the same document now defines two paragraphs above.

**"External facts checked against live sources", the four constraints.** The second constraint currently reads that restructuring is forbidden and that a fact whose correction would change a contract or a signature is reported rather than applied. Replace the criterion with the single-right-answer one and name the three routes, keeping the constraint itself, which is unchanged: the fact checker still writes nothing beyond the value.

**Same section, the third of the four constraints.** It reads "each pass decides and applies its own edits so none of them acts on another pass's conclusion". The relay implementer applies a conclusion it is explicitly forbidden to re-derive, so the sentence stops being true the moment this change lands, and no grep in this plan would catch it. The property that survives is not "no conclusion crosses" but "a conclusion that crosses is re-established, never inherited", which is the formulation the same document already uses at "One writer at a time" and which the section 2b re-check is what makes true. Reword the constraint to that, rather than dropping it: it is a too-absolute statement of a rule the document states correctly elsewhere, not a separate rule.

**Same section, the fix-vs-flag paragraph.** It says both fact passes apply the same split. That stays true of the plan pass and becomes a three-way split for the implementation pass. Say so, and say why the plan pass does not gain the third route: its bound is "never restructure the plan", and a plan is prose the human is about to read and approve, so an out-of-bound plan correction has a human gate in front of it already.

### 4. `docs/workflow.md`

**Section 2, the fact check bullet.** It lists three outcomes that do not stop the run, the third being a value confirmed wrong whose correction would change a contract or a signature, with the warning that the code carries a proven-false value. That outcome is what this change removes. Rewrite the third outcome as the narrowed class, and describe the relay path and its stopping condition.

**"When things go wrong", the fact checker bullet.** Add the relay's two failure paths and say they differ. A relay implementer that reports itself blocked is presented to the developer and asked about, like every other block. A re-check that finds the value still wrong ends the run instead of asking, and leaves the implementer's work uncommitted, so the developer finishes by hand. Also update the section's closing paragraph on there being no resume, which now has a second way to leave a dirty tree behind.

### 5. `.claude-plugin/plugin.json`

`1.9.3` to `1.10.0`. A new route through the implementation phase is a functional addition, matching the `1.9.0` bump that introduced the pass itself.

### 6. `README.md`

**`## Reliability`, the sentence "When a pass cannot resolve something on its own, the run stops and asks you".** The relay is exactly a pass that cannot resolve something on its own, and the run neither stops nor asks: it routes the correction to another agent. That is a direct contradiction this change creates, not the pre-existing looseness the sentence already had against the fact check's non-stopping outcomes. Reword it to what the workflow actually promises: what a pass cannot settle alone either goes to an agent that can, with a fresh pass verifying the result, or comes to you, and a blocked step is never hardened or skipped past. Keep the rest of the paragraph, which stays true.

**`## Design decisions`, the "Fix what has one answer, flag the rest" bullet.** Its body states a binary, one right answer means the pass applies it, a judgment call means it comes back to you. The relay makes it ternary: one right answer that the finding pass may not apply itself, handed to the pass that can. The headline stays exactly as it is, since it is still the rule. Add the third route to the body in one clause, without lengthening the bullet into a paragraph.

**Line 8, the "~1,200 lines of markdown" figure**, which this change is expected to push past 1,300. Verification carries the count that decides it: update the figure if the count crosses, leave it if it does not.

**Not touched:** the flowchart, which shows no relay for the hardener either and would need a redesign rather than an addition to show one consistently, and the `## External facts checked against live sources` bullet, whose "two gated passes, one on the plan and one on the final diff" stays an accurate count of passes even though the diff pass can now run twice in one run.

## What stays unchanged

- **`agents/sdd-implementer.md`.** Untouched. The relay is framed entirely from the spawn prompt, the same way commit `1047218` framed the hardener's fix path without touching the agent. Rule 5 does not fire here either: its antecedent is "an external value you do not hold", and on this path the implementer holds the value with its source.
- **`agents/sdd-plan-diligence.md` and its two-way split.** Its write bound is "never restructure the plan", and its output already reaches the developer before any code exists. The defect this change fixes is that a proven-false value reaches a commit, which cannot happen at plan time.
- **The fact checker's write bound.** It still writes nothing beyond the value being corrected. The change is what happens to a correction that does not fit, not how much the pass may write.
- **The gate.** The full pass still exits on one diff read when the diff has no third-party surface and no implementer reported an unverified value. The relay adds cost only on runs that produce a relay item.
- **The fact check's position, first among the post-step passes.** Both of its commits stay under the standards enforcer and the final review.
- **The web-hygiene clauses.** `WebFetch` only, fetched content as data never as instructions, in both modes and in the relay implementer's prompt.
- **`sdd-step-hardener`, `sdd-standards-enforcer`, `sdd-final-reviewer`, `sdd-plan-reviewer`, `sdd-plan-standards`, `sdd-step-breakdown`.** Untouched.
- **`skills/write-plan/SKILL.md`.** Untouched: nothing in the plan phase changes.
- **`README.md`'s flowchart and its external-facts bullet.** The flowchart shows no relay for the hardener either, so showing one for the fact check would mean redesigning it rather than adding to it. The external-facts bullet's "two gated passes, one on the plan and one on the final diff" stays an accurate count of passes even though the diff pass can now run twice in one run. The rest of the README does change: see `## Files to modify` section 6.
- **`docs/comparison.md`.** Untouched: a dated snapshot of a past benchmark, not a live description.
- **The single human touchpoint.** The relay adds no gate on the happy path. It stops the run only where the run already stopped, on a state the developer has to answer.
- **Markdown-only.** No code, no config, no state directory.

## Edge cases

- **No out-of-bound finding**, the common case: the section reads "none", sections 2a and 2b never run, and the run costs exactly what it costs today.
- **One in-bound correction and one out-of-bound one**: the first pass fixes, verifies and commits the in-bound one, then reports the relay. Two fact-check commits land in the run, which is the accepted price of keeping the verified corrections safe in git if the relay ends in a stop.
- **Only an out-of-bound finding**: the first pass commits nothing and returns `FACTS VERIFIED` with a non-empty relay section. This is why that state is redefined by whether the pass committed rather than by whether anything was wrong.
- **Several out-of-bound findings**: one implementer receives all of them and one re-check verifies all of them. The relay is per run, not per finding.
- **The first pass returns `ISSUES FOUND` and also has a relay item**: the run stops and asks, and the relay item is presented rather than routed. Building a correction on a tree whose verification is broken is the case the guard exists for.
- **The relay implementer returns `IMPLEMENTATION BLOCKED`**: presented to the developer, and the re-check does not run, matching section 1a and the hardener's fix path.
- **The re-check finds the value still wrong**: `ISSUES FOUND`, no commit, no second relay, and the run ends there rather than offering a choice. The implementer's work stays uncommitted and the developer finishes by hand.
- **The developer would rather continue after a section 2b `ISSUES FOUND`**: not offered. Continuing has only two coherent shapes, commit the relay diff or leave it, and both are excluded: committing lands a value the pass proved wrong, and leaving it hides a multi-file change from the two passes that diff from `$BASELINE_SHA`. The run ends and the tree is the developer's.
- **The re-check finds a new false external fact the implementer introduced** while making the correction: same route, `ISSUES FOUND`. This is why the re-check gates the whole implementer diff rather than only the relayed value.
- **The relay implementer declares an `Unverified external value:` of its own** while applying the correction: it travels into the section 2b prompt, not into `$UNVERIFIED_FACTS`, which section 2 already consumed. The re-check would gate it anyway as part of the implementer's diff, so the carry is a pointer, not a second source of truth.
- **The relay implementer changed nothing**: the value is still wrong, so the re-check returns `ISSUES FOUND` like any other unfixed value. There is no separate empty-diff case to handle.
- **The re-check cannot reach the source**: `ISSUES FOUND`, no commit. The full pass lets an unverifiable fact ride along as a judgment item, and targeted mode may not, because signing off is the only thing it is there to do.
- **The re-check's verification commands fail and cannot be repaired**: `ISSUES FOUND`, no commit.
- **No verification commands exist in the project** (this repo): the re-check commits after checking the value, exactly as the other committing agents do when discovery comes back empty.
- **The run stopped at section 1b with "stop"**: section 2 never runs, so no relay, and the summary reports the fact check as not run.
- **The relay implementer edits files the relay list did not name**, because the true value requires it: allowed and expected, since the class is defined by exceeding the literal. Nothing goes unread, because the re-check gates its whole diff.
- **A fact is out of bound and also needs a choice**: it is one finding, and the choice decides. It goes to `REQUIRES YOUR JUDGMENT`, never to both sections.

## Test scenarios

Markdown-prompt behavior, no automated runner. Verify by reading the contracts, and by driving `/implement-plan` against crafted inputs with `--plugin-dir` pointed at the working tree:

1. A diff whose code reads a third-party field at a path that is really nested one level deeper: the fact check reports it under `NEEDS A FRESH IMPLEMENTER` instead of `REQUIRES YOUR JUDGMENT`, the orchestrator relays, the implementer corrects the path and its call sites without committing, and the re-check verifies, commits, and returns `FACTS CORRECTED`.
2. The same run's summary: the fact-check line reports the relay, the corrected value appears in the commit list, and the judgment items do not carry the finding, because it was resolved rather than deferred.
3. A diff calling an endpoint that really returns a paginated envelope where the code expects a bare array: the fact check reports it under `REQUIRES YOUR JUDGMENT` with the true shape and the source, no relay is spawned, and the run continues to the standards enforcer.
4. A run with one in-bound correction and one out-of-bound one: two fact-check commits, the first landing before the relay, and the branch still readable in order.
5. A purely internal refactor: the gate closes on one diff read, the relay section reads "none", and no implementer is spawned.
6. A relay whose implementer corrects the value but invents an adjacent field name: the re-check gates its diff, catches the invented field, and returns `ISSUES FOUND` rather than committing.
7. A relay whose re-check finds the value still wrong: `ISSUES FOUND`, the run goes straight to the summary without offering fix, skip or continue, the standards and final review lines read `not run`, and the summary says the tree still carries the implementer's uncommitted work.
8. A relay whose implementer returns `IMPLEMENTATION BLOCKED`: the block reaches the developer and the re-check never runs.
9. A first pass returning `ISSUES FOUND` with a relay item pending: the developer sees both, and nothing is routed.
10. The final review, running after a relay, sees both fact-check commits in `git diff $BASELINE..HEAD` and reports the corrected value as a justified deviation rather than accidental drift.
11. `FACTS CORRECTED` from the re-check quotes the real tail of the verification commands it ran, and its commit follows the project's discovered convention with files staged by name.

## Verification

No automated tests (markdown-only repo). The available check is `claude plugin validate . --strict`, which currently passes. Checks:

- `claude plugin validate . --strict` passes.
- `grep -c 'NEEDS A FRESH IMPLEMENTER' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md` returns a hit in each: the signal is produced and consumed under one name.
- `grep -nE 'contract, a signature|a contract or a signature' agents/sdd-fact-checker.md docs/design-decisions.md docs/workflow.md` returns nothing: the size-based criterion is gone from every surface that stated it. The two spellings both have to be in the pattern, since the agent says "a contract, a signature" and the two docs say "a contract or a signature": a grep for one spelling alone passes vacuously on the files that use the other.
- `grep -nE 'non-minimal|not a minimal edit' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md` returns nothing: the narrowed class replaced the old wording in the `## Output` lead-in, in all three per-state bullets, and in the section 5 summary block.
- `grep -n "workflow's one relay" skills/implement-plan/SKILL.md` returns nothing: section 1b no longer claims a uniqueness section 2a breaks.
- `grep -c 'single right answer' agents/sdd-fact-checker.md` is at least 1: the replacement criterion is stated in the agent, not only in the docs.
- `grep -n 'Targeted re-check' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md` shows the same section name in the agent that branches on it and the skill that emits it.
- In `agents/sdd-fact-checker.md`, read the `## Targeted re-check mode` section in full: it must state that it never emits `NEEDS A FRESH IMPLEMENTER`, and must give all four stopping conditions (value still wrong, new false fact in the implementer's diff, verification unrepairable, source unreachable).
- `grep -n 'Nothing needed fixing' agents/sdd-fact-checker.md` returns nothing: the `FACTS VERIFIED` body no longer claims nothing was wrong.
- `grep -c 'no-verify' agents/sdd-fact-checker.md` still returns a hit, and the targeted mode does not introduce a commit path that bypasses the prohibition.
- `grep -n 'sdd-implementer' skills/implement-plan/SKILL.md` shows three spawns (section 1a, the section 1b fix path, section 2a), and `grep -n 'sdd-fact-checker' skills/implement-plan/SKILL.md` shows two (section 2 and section 2b).
- `git diff --stat` shows `agents/sdd-implementer.md` untouched, and `grep -c 'Targeted re-check\|relay' agents/sdd-implementer.md` returns 0: the relay is framed from the spawn prompt alone.
- Read the section 2a prompt: it names the true value, the source and the landing file, forbids re-verifying against the web, forbids committing, and bounds the edit.
- `grep -n 'One qualified exception' docs/design-decisions.md` returns nothing, and the replacement enumerates two relay-form exceptions plus the orchestrator's own commits.
- `grep -n 'you choose' docs/design-decisions.md` still returns its hit, and the rewritten parenthetical around it says in plain words that the fact-check relay fires without that choice and why. The two relays must not be described as the same shape without this. Read the paragraph rather than trusting the grep, which only proves the old clause survived.
- Read the section 5 summary template: the branch where section 2b ended the run must name the uncommitted tree and the files it touches, not merely report `not run` on the two passes behind it.
- `grep -n "none of them acts on another pass" docs/design-decisions.md` returns nothing: the third of the four constraints was reworded, not left standing against the new route. This one has to be checked by name, because the plan's other design-decisions greps all target different sentences and none of them would fire on this one.
- Read section 2b of `skills/implement-plan/SKILL.md` in full: its `ISSUES FOUND` branch must go to the summary with no question to the developer, and must say the tree is left dirty. Then `grep -n 'Fix these issues, skip them, or stop' skills/implement-plan/SKILL.md` returns one hit, in section 1b only: the three-way choice must not have been copied into section 2b.
- `grep -c 'Fact check' skills/implement-plan/SKILL.md docs/workflow.md` returns a hit in each, and the section 5 status line carries wording for a run that relayed.
- `grep -c 'not run' skills/implement-plan/SKILL.md` still returns 3: adding sections 2a and 2b did not drop the stopped-short wording from any of the three pass status lines.
- `grep -c '—' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md docs/design-decisions.md docs/workflow.md` returns 0 in each: the distributed surface carries no em dash today and must not gain one. Do not grep root `CLAUDE.md` or `.claude/skills/commit/SKILL.md`, which do carry some.
- `grep -c ';' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md docs/design-decisions.md docs/workflow.md` does not exceed the pre-change values, which are 1, 2, 0 and 0. Semicolons are discouraged in this repo's prose the same way em dashes are, and the four files are close enough to zero that any new one is worth seeing. This is a ceiling, not an equality: it does not ask the change to remove the three that already exist.
- `.claude-plugin/plugin.json` shows `1.10.0`.
- `wc -l agents/*.md skills/*/SKILL.md | tail -1` against README line 8, which reads "~1,200 lines of markdown". The pre-change total is 1279, so it takes only about twenty added lines to reach 1300 and this change adds more than that. Expect to update the README figure to "~1,300", which makes the README the sixth modified file rather than an untouched one.
- Manual: drive scenarios 1, 3 and 6 end to end with `--plugin-dir`.

## Implementation steps

Markdown-only repo: there is no test runner and no build. The only automated check is `claude plugin validate . --strict`, which passes today. Every other check below is a grep or a close read of the contract, run from the repo root. Commits follow `CLAUDE.md`: use `/commit`, stage by name, never `git add -A`.

### Step 1: the relay route in the distributed surface

**Files**

- `agents/sdd-fact-checker.md` (modify)
- `skills/implement-plan/SKILL.md` (modify)
- `.claude-plugin/plugin.json` (modify)

**Do**

This step makes the signal exist and be consumed. Apply the edits described in `## Files to modify` sections 1, 2, 5 and 6 of this plan, in this order, so each file is left consistent before the next is opened.

`agents/sdd-fact-checker.md`, six edits:

1. `## Context received from the orchestrator`: name the second prompt shape beside the first, the targeted re-check prompt (plan path, baseline ref, `## Targeted re-check` list).
2. Top of `## Setup`: add a first step branching on the presence of a `## Targeted re-check` section in the prompt, sending the run to `## Targeted re-check mode` instead of the rest of `## Setup`, `## The gate` and `## The due diligence record`. State that the verification rule, web hygiene, verification-command discovery, commit-convention discovery, staging by name and the `--no-verify` prohibition apply unchanged in both modes.
3. `## Fix vs flag`, fourth bullet: replace the single size-keyed bullet with two bullets keyed on whether the artifacts hold one answer, per the plan's `## Files to modify` section 1. Keep the first three bullets untouched.
4. `## Output` lead-in and the three per-state bullets: every state carries both a `**NEEDS A FRESH IMPLEMENTER:**` section and the existing `**REQUIRES YOUR JUDGMENT:**` section, each defaulting to "none". Reword the judgment inventory in the lead-in and in all three state bodies so "non-minimal corrections" / "not a minimal edit" is replaced by the narrowed class. `FACTS CORRECTED` keeps "reverted fixes".
5. `### FACTS VERIFIED` body: redefine the two continuing states by whether the pass committed, removing "Nothing needed fixing".
6. New `## Targeted re-check mode` section: scope is the relay implementer's uncommitted diff (`git diff` and `git diff --cached`), not the run diff since baseline. Three things it does (re-verify each listed value against its live source treating the relayed value and source as a pointer, gate the implementer's whole diff for external facts it introduced that were not on the list, then run the discovered verification commands, stage by name and commit returning `FACTS CORRECTED`), the guard (never emits `NEEDS A FRESH IMPLEMENTER` whatever it finds), and the four stopping conditions that return `ISSUES FOUND` with no commit. Note the unreachable-source divergence next to the existing closing sentence it contradicts ("An unverifiable fact is NOT `ISSUES FOUND` ..."), so the two do not read as a contradiction.

`skills/implement-plan/SKILL.md`, seven edits:

1. Section 0: add `$RELAY_FACTS`, empty at the start, beside `$UNVERIFIED_FACTS`.
2. Section 1a, `IMPLEMENTATION COMPLETE` bullet: extend "This applies to every implementer invocation, including the extra implementer spawned on the §1b fix path" to name the third invocation (§2a) and its different destination (the §2b prompt, not `$UNVERIFIED_FACTS`).
3. Section 1b fix branch: reword "This is the workflow's one relay of a pass's conclusions to another pass to apply" to name the two relays and what they share.
4. Section 2, result handling: keep the three outcomes and add the routing described in the plan. Bound what enters the orchestrator to the true value, the source and the landing file, in the same words the section already uses for `$UNVERIFIED_FACTS`.
5. New section 2a: spawn `spec-driven-dev:sdd-implementer` with `model="opus"`, following the `Display:` block plus `Task(...)` shape used by the six existing spawns. Carry a `## State of the tree` block, and present the relay items under `## Corrections to apply` as the entire scope, with the plan path below marked as context only, so `agents/sdd-implementer.md` `## Setup` step 1 does not fall through to implementing the whole plan. State that the values are already verified so no web calls are spent re-verifying them, that the edit is bounded to what the true value requires, and that it must not commit. Handle `IMPLEMENTATION BLOCKED` exactly as section 1a, with no re-check on a blocked tree.
6. New section 2b: spawn `spec-driven-dev:sdd-fact-checker` with the same `Plan file:` and `Baseline:` lines section 2 uses, plus a `## Targeted re-check` section carrying `$RELAY_FACTS` verbatim and any value the relay implementer declared unverified. Two outcomes only. `FACTS CORRECTED` continues to section 3. `ISSUES FOUND` ends the run: present the issues, say the relay implementer's work is still uncommitted, and go straight to the section 5 summary, with no fix/skip/stop question and no second relay. Carry the three reasons from `## Files to modify` section 2 into the skill, since this is the only `ISSUES FOUND` in the file that does not ask the developer anything.
7. Section 5 summary: add the relay to the `Fact check:` status line alternatives, reword `Fact-check judgment items:` to the narrowed class, add a line reporting what was relayed and whether the re-check committed it or ended the run, make the ended-the-run case name the uncommitted tree and the files it touches, and extend the closing paragraph on runs that stop short to cover section 2b as a third such path, where the standards and final review lines read `not run` and the summary says the tree is dirty.

`.claude-plugin/plugin.json`: `1.9.3` to `1.10.0`.

`README.md` is not touched by this step. Its prose edits are documentation and its line-count figure is only decidable once this step's two prompt files are final, so both belong to step 2.

**Test**

No runner. The contract is checked by grep and by reading, against the exact baselines measured on the pre-change tree:

- `grep -c 'NEEDS A FRESH IMPLEMENTER' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md` returns a non-zero hit in each: the signal is produced and consumed under one name.
- `grep -nE 'contract, a signature|a contract or a signature' agents/sdd-fact-checker.md` returns nothing. Both spellings must stay in the pattern, since the agent says "a contract, a signature" and the docs say "a contract or a signature".
- `grep -nE 'non-minimal|not a minimal edit' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md` returns nothing: the old wording is gone from the `## Output` lead-in, from all three per-state bullets, and from the section 5 summary block.
- `grep -c 'single right answer' agents/sdd-fact-checker.md` is at least 1.
- `grep -n "workflow's one relay" skills/implement-plan/SKILL.md` returns nothing.
- `grep -n 'Targeted re-check' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md` shows the same section name in the agent that branches on it and the skill that emits it.
- `grep -n 'Nothing needed fixing' agents/sdd-fact-checker.md` returns nothing.
- `grep -c 'no-verify' agents/sdd-fact-checker.md` still returns a hit, and targeted mode introduces no commit path that bypasses the prohibition.
- `grep -c 'sdd-implementer' skills/implement-plan/SKILL.md` shows three spawns (1a, the 1b fix path, 2a) and `grep -c 'sdd-fact-checker' skills/implement-plan/SKILL.md` shows two (2 and 2b).
- `grep -c 'not run' skills/implement-plan/SKILL.md` still returns exactly 3 (pre-change value): adding sections 2a and 2b dropped the stopped-short wording from none of the three pass status lines.
- `grep -c 'Fact check' skills/implement-plan/SKILL.md` returns a hit and the section 5 status line carries wording for a run that relayed.
- `grep -c '—' agents/sdd-fact-checker.md skills/implement-plan/SKILL.md` returns 0 in each (pre-change value is 0 in both). Do not grep root `CLAUDE.md` or `.claude/skills/commit/SKILL.md`, which do carry some.
- `git diff --stat` shows `agents/sdd-implementer.md` untouched, and `grep -c 'Targeted re-check\|relay' agents/sdd-implementer.md` returns 0.
- Read `## Targeted re-check mode` in full: it must state that it never emits `NEEDS A FRESH IMPLEMENTER`, and must give all four stopping conditions (value still wrong, new false fact in the implementer's diff, verification unrepairable, source unreachable).
- Read the section 2a prompt: it names the true value, the source and the landing file, forbids re-verifying against the web, forbids committing, and bounds the edit.
- `grep -n version .claude-plugin/plugin.json` shows `1.10.0`.

**Verify**

- `claude plugin validate . --strict` passes (it passes on the pre-change tree, so a failure is caused by this step).
- Every grep above returns the stated result.
- `wc -l agents/*.md skills/*/SKILL.md | tail -1`: record the number. Step 2 compares it against README line 8. `git diff --stat` shows no `README.md` for this step.

### Step 2: align the documentation with the new route

**Files**

- `docs/design-decisions.md` (modify)
- `docs/workflow.md` (modify)
- `README.md` (modify)

**Do**

Read the step 1 result first: `agents/sdd-fact-checker.md` `## Fix vs flag` and `## Targeted re-check mode`, and `skills/implement-plan/SKILL.md` sections 2, 2a and 2b. The docs must describe the wording that actually landed, not the wording this plan proposed.

`docs/design-decisions.md`, seven edits, per `## Files to modify` section 3:

1. `## A lightweight orchestrator`, the bounded-carry sentence: name the relay triple beside the implementer's unverified value, so the rule enumerates a set rather than a single example.
2. `## One writer at a time`, the parenthetical starting "One qualified exception": restate it as two exceptions of the relay form (the hardener fix path and the fact-check relay) plus the orchestrator's own commits as the narrower third, saying what makes the fact-check relay the same shape. In the same paragraph, name where they differ: the existing text says "you choose between fix, skip, and stop", so the hardener relay is human-authorized on every firing and the fact-check relay is not. Give the reason the second is granted without a gate rather than leaving the difference unsaid.
3. Same section, the sentence "The hardener exception above takes the second path ...": extend it to both relays, and say why the fact-check relay is the cheaper of the two to justify.
4. Same section, the closing paragraph "The dividing line allows more than this workflow uses ...": narrow "This workflow has no use for that shape" to the parallel-reviewers shape it actually means, so the paragraph stops reading as a denial of the route defined two paragraphs above.
5. `## External facts checked against live sources`, the four constraints: replace the size-based criterion in the second constraint with the single-right-answer one and name the three routes, keeping the constraint itself (the fact checker still writes nothing beyond the value).
6. Same section, the third of the four constraints ("each pass decides and applies its own edits so none of them acts on another pass's conclusion"): reword it to the property that survives, that a conclusion crossing a context boundary is re-established and never inherited, which is what the section 2b re-check makes true. Do not drop it: it is a too-absolute statement of a rule this document states correctly at `## One writer at a time`.
7. Same section, the closing fix-vs-flag paragraph: say the split stays two-way for the plan pass and becomes three-way for the implementation pass, and why the plan pass does not gain the third route (its bound is "never restructure the plan", and its output reaches a human gate before any code exists).

`docs/workflow.md`, three edits, per `## Files to modify` section 4:

1. Section 2, the fact check bullet: rewrite the third non-stopping outcome as the narrowed class, and describe the relay path and its stopping condition.
2. `### When things go wrong`, the fact checker bullet: add the relay's two failure paths and say they differ. A relay implementer that reports itself blocked is presented and asked about. A re-check that finds the value still wrong ends the run instead of asking, leaving the implementer's work uncommitted for the developer to finish by hand.
3. `### When things go wrong`, the closing paragraph on there being no resume: it now has a second way to leave a dirty tree behind, so name it.

`README.md`, three edits, per `## Files to modify` section 6:

1. `## Reliability`, the sentence "When a pass cannot resolve something on its own, the run stops and asks you": reword it so the relay is covered, since that is a pass which cannot resolve something on its own and where the run neither stops nor asks. What a pass cannot settle alone either goes to an agent that can with a fresh pass verifying the result, or comes to you. Keep the rest of the paragraph.
2. `## Design decisions`, the "Fix what has one answer, flag the rest" bullet: the headline stays verbatim, the body gains the third route in one clause (one right answer the finding pass may not apply itself, handed to the pass that can). Do not grow the bullet into a paragraph.
3. Line 8: run `wc -l agents/*.md skills/*/SKILL.md | tail -1` now that step 1 is final. The pre-change total is 1279 and line 8 reads "~1,200 lines of markdown". If the new total is 1300 or more, update the figure to "~1,300". If it is not, leave the figure alone. `docs/` does not count toward this total.

**Test**

- `grep -nE 'contract, a signature|a contract or a signature' docs/design-decisions.md docs/workflow.md` returns nothing: the size-based criterion is gone from every surface that stated it. Combined with step 1, the same grep across all four files now returns nothing.
- `grep -n 'One qualified exception' docs/design-decisions.md` returns nothing, and the replacement enumerates two relay-form exceptions plus the orchestrator's own commits.
- `grep -c 'Fact check' docs/workflow.md` returns a hit.
- `grep -c '—' docs/design-decisions.md docs/workflow.md` returns 0 in each (pre-change value is 0 in both).
- Read the narrowed closing paragraph of `## One writer at a time`: it must not deny the route the same section now defines.

**Verify**

- `claude plugin validate . --strict` passes.
- Every grep above returns the stated result.
- `grep -n 'the run stops and asks you' README.md` returns nothing: the sentence the relay contradicts was reworded, not left standing.
- `grep -n 'Fix what has one answer, flag the rest' README.md` still returns its hit: the headline is the rule and does not change, only the body gains the third route.
- `wc -l agents/*.md skills/*/SKILL.md | tail -1` against README line 8. If the total is 1300 or more the figure reads "~1,300", otherwise it is untouched. Either way the README is in this step's diff for the two prose edits.
- `git diff --stat` shows only `docs/design-decisions.md`, `docs/workflow.md` and `README.md` for this step.

### After both steps

The manual scenarios in `## Test scenarios` are not part of either step's verification, since they need a crafted target repo and a live run. Drive scenarios 1, 3 and 6 end to end with `--plugin-dir` pointed at this working tree once both steps have landed.
