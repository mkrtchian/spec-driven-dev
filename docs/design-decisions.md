# Design decisions

Part of [spec-driven-dev](../README.md). The short version lives in the [README](../README.md#design-decisions). This page gives the detailed rationale behind each choice. Where published engineering work backs a choice, it is cited.

## A workflow, not an autonomous agent

The plugin is a fixed sequence of passes with human gates, not an agent that decides its own path. Anthropic's [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) draws this line: workflows fit tasks with a predictable structure, autonomous agents fit open-ended tasks where the path can't be known in advance. Turning a discussed feature into reviewed code has a predictable structure: draft, review, check standards, run due diligence, break into steps, implement, verify. A fixed sequence makes each run predictable, debuggable, and cheap to reason about.

## Why not just plan mode?

Claude Code ships a native answer to "plan before you code": plan mode. It [separates exploration from execution](https://code.claude.com/docs/en/best-practices#explore-first-then-plan-then-code) inside one session. What it doesn't change: the plan lives in the conversation, and the context that wrote it is the context that implements it. This workflow adds what plan mode doesn't have. The plan becomes a durable file that goes through PR review. Dedicated agents review it without having seen the discussion. Implementation starts fresh, reading only the plan. Each step lands as its own commit. Checkpointing and `/rewind` cover undoing mistakes after the fact, review passes catch them before. The two compose: plan mode for quick scoped changes, this workflow when the change deserves a reviewed plan.

## Isolated context per concern

Each review and implementation pass runs in a dedicated agent that has not seen the previous passes. Drafting is the deliberate exception: the orchestrator writes the plan itself, because the draft needs the discussion. Two reasons for the isolation, one about attention and one about bias.

**Attention.** A single long conversation degrades as context fills up. This is measured, not anecdotal: [Chroma's context rot report](https://research.trychroma.com/context-rot) shows performance dropping as input length grows, across 18 models, with task difficulty held constant. A review pass that carries the whole discussion, the draft, and its own tool history works under exactly the conditions that report shows degrade performance, and Chroma expects the effect to be stronger on complex real-world tasks than on their controlled ones.

**Bias.** An agent that just spent 20 minutes implementing code is not in the right mindset to review it. Claude Code's [best practices](https://code.claude.com/docs/en/best-practices#run-multiple-claude-sessions) state it directly: a fresh context improves review because the model "won't be biased toward code it just wrote", or in their one-line version, the agent doing the work isn't the one grading it.

Cognition reached the same conclusion in production with Devin, their coding agent: its review loop works best when "the coding and review agents do not share any context beforehand", and even on PRs written by Devin itself the clean-context reviewer catches an average of 2 bugs per PR ([Multi-agents working](https://cognition.ai/blog/multi-agents-working)). Their explanation is not the human defensiveness analogy (they note these agents "don't have egos"). It's informational: the clean reviewer is forced to reason backward from the artifact alone, so it questions things the author overlooked, and its short context keeps it sharp where the author's is already full.

Early empirical support points the same way: a [controlled preprint study](https://arxiv.org/abs/2603.12123) found fresh-context review beating both self-review (F1 28.6% vs 24.6%) and context-aware sub-agent review, with the benefit coming from the context separation itself. One small, single-model study, so treat it as a signal, not proof.

Isolation has a cost: a fresh agent inherits none of the main session's project context. In particular, Claude Code resolves `CLAUDE.md` `@`-imports for the main session but not for sub-agents. Every agent here therefore starts with the same discovery preamble: read the root `CLAUDE.md`, resolve its `@`-references itself, check for nested `CLAUDE.md` files in the directories it touches. Each pass re-pays that reading cost. That's the price of fresh eyes.

## A lightweight orchestrator

The orchestrator never loads the sub-agent prompts: the runtime resolves them through `subagent_type`. In skill-based frameworks, by contrast, the orchestrating agent loads the full text of each skill into its own context, and that text is resent to the model on every turn. Here each sub-agent reads the project context itself, explores the codebase, spends its own context doing it, and returns a short report (a status line like `STEP COMMITTED` plus findings). The orchestrator keeps the plan and those reports, nothing else.

This applies the principle 12-factor agents calls "[own your context window](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md)" (there, about controlling what tokens reach the model) to the agent where it matters most: the orchestrator is the longest-lived agent and the one you talk to, so its context is the most valuable to protect. Anthropic's [context engineering guidance](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) describes the same sub-agent pattern: the detailed work stays isolated in the sub-agent, the coordinator only sees condensed summaries.

## A fresh start between planning and implementation

The discussion phase fills context with exploration, questions, dead ends, and design alternatives. By the time you run `/implement-plan`, you want an agent that reads only the plan: the same attention and bias arguments as above, applied between the two phases. This is also why the plan has to be self-contained, it's the only channel between the two commands. The separation is yours to enforce: you clear context (or open a fresh session) between `/write-plan` and `/implement-plan`, as the [README usage](../README.md#usage) shows.

## One writer at a time

The passes run in sequence, and each pass that decides an edit applies it itself. No pass hands a list of proposed edits to another agent to apply. (One qualified exception: when the step hardener reports issues it can't fix, you choose between fix, skip, and stop. On fix, a fresh implementer receives the hardener's findings as prose and a new hardening pass verifies the result.) Both halves of the rule are deliberate.

**Sequential, not parallel.** Parallel writers make implicit decisions (style, interpretation of an ambiguity, edge-case handling) that conflict with each other. Cognition's [Don't build multi-agents](https://cognition.ai/blog/dont-build-multi-agents) names the principle: actions carry implicit decisions, and conflicting decisions carry bad results. Their production follow-up ten months later keeps the rule: multi-agent systems work best today when "writes stay single-threaded and the additional agents contribute intelligence rather than actions" ([Multi-agents working](https://cognition.ai/blog/multi-agents-working)). Anthropic's [multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) adds the economics: parallel sub-agents pay off for breadth-first, read-only research (many independent questions fanned out at once), while coding offers fewer truly parallelizable subtasks. Sequential passes on verified state also keep failures local: when something breaks, one pass broke it.

**The pass that decides an edit applies it.** The dividing line is what a sub-agent produces. Facts compress without loss, so fact-gathering can be delegated, parallelized, and reported back. Edit decisions don't compress: describing a change in prose for another model to apply loses fidelity at every ambiguity. The industry learned this with "edit apply models": a large model described changes and a small model applied them. The pattern was largely dropped because the applier kept misreading the describer, and today deciding and applying an edit are usually done by a single model ([Don't build multi-agents](https://cognition.ai/blog/dont-build-multi-agents)). Anthropic describes the same hazard as a game of telephone and recommends sub-agents write their outputs directly to the filesystem instead of piping them through a coordinator ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system), appendix). The review passes here edit the plan file directly for exactly that reason.

There's a second reason the orchestrator doesn't apply review findings itself: it wrote the plan. Filtering the review through the author reintroduces the bias that isolation exists to remove.

## Fix what has one answer, flag the rest

Every review pass separates two kinds of findings. Corrections with a single right answer (a wrong file path, a type signature that doesn't match the code, a violation of a documented standard) are applied directly. Anything that needs a judgment call is flagged instead, as a `<!-- REVIEW: ... -->` marker in the plan or as a remark in the pass report, and routed to you.

Cognition calls the equivalent mechanism in Devin the "communication bridge": the filter that decides which review findings deserve action, using context the reviewer doesn't have ([Multi-agents working](https://cognition.ai/blog/multi-agents-working)). This plugin splits that filter in two. Mechanical fixes are the ones a reviewer can verify from the artifact alone, so they are safe to apply without the discussion context, and the plan gate plus per-step commits keep a misclassified fix visible and revertable. Judgment calls go to the person who has the discussion context. On the happy path you're interrupted once per command, at the end. Agents stop mid-run only on failures they can't resolve. The fewer the interruptions, the more attention each one gets.

## External facts checked against live sources

The model's training cutoff mechanically stales third-party facts: library and CI action versions, API contracts, platform behavior. A review that runs from memory cannot catch that staleness, because a stale fact looks exactly like a correct one. So a dedicated due diligence pass runs at the end of `/write-plan`, after review and standards, and verifies the plan's external facts against live sources. It is gated: a plan that cites no external facts pays nothing for it.

The pass applies the same fix-vs-flag split as the review passes, this time to facts. A fact that is wrong or broken (a version, endpoint, or field that does not exist or is nonfunctional) has one right answer, so it is fixed in place, quoting old → new so the human can audit the edit. A fact that is merely valid-but-not-latest (a pinned `v4` when `v7` exists) is flagged, not changed: the pin may be deliberate, and only the human holds the discussion context to decide. Fetched web content is treated as data, never as instructions, and fetching goes through `WebFetch` only, whose summarizing layer is the mitigation between untrusted pages and the agent's context. The same pass also collects the `<!-- REVIEW: ... -->` markers earlier passes leave behind, which closes the routing loop "Fix what has one answer, flag the rest" describes: the markers finally reach the human before execution.

## Dynamic discovery over configuration

The agents hardcode nothing about your stack. Each one discovers how the project works at run time: documented commands in `CLAUDE.md` first, then the project's own config files (`package.json` scripts, `Makefile` targets, `pyproject.toml`, `Cargo.toml`, `go.mod`, and equivalents) for tests, lint, and typecheck. Committing agents discover commit conventions the same way: `CLAUDE.md` rules first, then a `/commit` skill if the project has one, then commitlint or commitizen config. The alternative, a plugin config file mapping each project to its commands, would be one more artifact to write, keep in sync, and get stale. Reading the project beats configuring the plugin.

## Conditional TDD

Business logic gets test-first: the implementer writes the tests, runs them to confirm they fail, then implements until they pass. Glue code, wiring, and configuration get implemented directly, with tests after if they add value. The line is whether the step has behavior worth pinning down: a data transformation has expected inputs and outputs a failing test can capture, a re-export or a config block doesn't. Applying test-first everywhere would produce ceremony tests that assert nothing. This is a prompt-level default with a stated exception, not an enforced gate (see Limits).

## Step hardening and the commit split

The implementer never commits. After each step, a fresh agent (the step hardener) reads the diff against the step and the plan, catches what emerged during implementation (broken imports, type mismatches, missed edge cases, shallow tests), fixes what has a clear answer, runs the project's verification commands, and only then commits. The split is the point: nothing enters git history reviewed only by the agent that wrote it. It also keeps problems local to the step where they appeared, instead of surfacing at the end with five steps stacked on top. If the hardener can't get verification to pass, it doesn't commit. It reports, and you decide.

## Plans in git, no hidden state

The plan is an ordinary markdown file, committed to your repository, reviewed like any other change. No state directory, no database, no memory shared between agents outside artifacts you can read. Everything the workflow will do is inspectable before it runs, and everything it did is in git history after. To make the "committed to your repository" property real rather than a suggestion, `/write-plan` ends by proposing to commit the plan (staging only the plan file, never automatically): the workflow itself produces the committed artifact instead of leaving it to the user to remember.

This also supports the supervision posture [Kief Morris describes](https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html) for agentic delivery. When the output is wrong you can fix the plan directly, which Morris calls working "in the loop". Or you can fix the prompts and process that produced it, working "on the loop", where the correction improves every future run. Plans and prompts both being plain files in git is what keeps that second option cheap. Per-step commits give recoverable checkpoints, the same discipline Anthropic found necessary for [long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents): incremental progress, descriptive commits, a clean state after every session.

## Limits

The guarantees are prompt-strength, not machine-strength. TDD, standards checks, and verification runs are instructions that models follow reliably in practice, not enforced gates. For stronger guarantees on tests, lint, and typecheck, pair the workflow with pre-commit hooks: agents trigger them on every commit.

Isolation is paid in wall-clock time and tokens. On the benchmark feature, the full workflow took ~22 min against ~15 min for frameworks without isolated review passes, with 6 agent invocations for a 2-step plan (see the [framework comparison](comparison.md), a March 2026 snapshot). The claim is quality and reviewability, not speed: net time saved against plain AI-assisted coding is unmeasured.

The workflow is sized for multi-file features. For a single-file fix or a small refactor, the overhead exceeds the benefit: use plain Claude Code, or plan mode. One fixed workflow can't fit every size of change, a point [Birgitta Böckeler's analysis](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) of spec-driven tools makes well, so this one doesn't try.

The machinery optimizes conformance to the plan, not the plan itself. If implementation reveals the plan's approach is wrong, agents flag what they can't fix and stop. Rewriting the plan stays your decision ([workflow.md](workflow.md) covers the failure paths).
