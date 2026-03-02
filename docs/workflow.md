# Workflow

Part of [spec-driven-dev](../README.md).

## 1. Discuss and plan

Start a conversation with Claude Code. If you have requirements from a product discovery (user stories, acceptance criteria, technical constraints), give them as context:

```
/write-plan path/to/requirements.md
```

Or just launch it with an idea:

```
/write-plan Add a dark mode toggle to the user settings page
```

The agent will:

1. **Discuss**: Explore the codebase, ask you clarifying questions about requirements and technical approach, until you both have a shared understanding.
2. **Draft**: Write a detailed implementation plan to `plans/YYYY-MM-DD_feature-name.md`.
3. **Review the plan** (fresh agent): Check for gaps, wrong assumptions, integration risks. Auto-corrects the plan.
4. **Check standards** (fresh agent): Verify the plan respects project coding and testing conventions. Auto-corrects.
5. **Break into steps** (fresh agent): Split the plan into the fewest ordered implementation steps that fit within an agent's context budget.

The command ends here. You review the plan and make necessary changes — manually or by asking the agent to make the adjustments.

## 2. Implement

Once the plan looks good, clear context and run:

```
/implement-plan plans/YYYY-MM-DD_feature-name.md
```

The orchestrator executes each step from the plan:

**For each step:**

- **Implementer** (fresh agent): For business logic, writes all tests first (red), then implements to pass (green). For glue code, implements directly. Runs tests, lint, typecheck. Does not commit.
- **Step hardener** (fresh agent): Catches drift from the plan and emergent issues (broken imports, type mismatches, edge cases), fixes them, and commits the step. Flags trade-offs it can't resolve.

**After all steps:**

- **Standards enforcement** (fresh agent): Checks the full diff against project coding standards. Fixes violations directly, verifies (tests, lint, typecheck), and commits.
- **Final review** (fresh agent): Reads the full plan and full diff. Fixes obvious issues directly (typos, wrong imports, dead code, convention violations) and commits them. Flags trade-offs and architectural choices as remarks for you to decide.

## 3. Review and adjust

Check the result. The plan is still there as the reference — you can point to specific sections and ask for adjustments.

### When things go wrong

The orchestrator stops and asks you when an agent can't resolve an issue on its own:

- **Implementer fails verification** (tests, lint, or typecheck don't pass): the orchestrator presents the failure and asks how to proceed.
- **Hardener finds issues it can't fix** (architectural trade-offs, ambiguous requirements): you choose to fix, skip, or stop.
- **Standards enforcer finds unresolvable violations**: same — you decide.

In all cases, the plan is still the reference. You can adjust it, ask the agent to retry a step, or finish manually.

Note: there is no state tracking across sessions. If you stop mid-way and close the session, you restart `/implement-plan` from the beginning. Already-committed steps are in git — the agents will see the existing code and tests, but the orchestrator won't skip steps automatically.

## Why isolated context?

**Between passes.** A single long conversation degrades in quality as context fills up. An agent that just spent 20 minutes implementing code is not in the right mindset to review coding standards: it's biased toward defending what it just wrote. A fresh agent with only the diff and the standards document has no such bias. Same principle as code review — the reviewer shouldn't be the author.

There's also a token cost dimension. In skill-based frameworks, the orchestrating agent loads skill prompts into its own context, and that text is retransmitted at every turn. When agent prompts are resolved by the runtime instead (via `subagent_type`), the orchestrator never sees them — only short sub-agent results come back, and the orchestrator stays lightweight throughout.

**Between planning and implementation.** The discussion phase fills context with exploration, questions, dead ends, and design alternatives. By the time you run `/implement-plan`, you want a fresh agent that reads only the plan — no residual bias from the discussion.
