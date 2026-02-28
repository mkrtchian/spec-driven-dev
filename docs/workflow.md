# Workflow

## 1. Discuss and plan

Start a conversation with Claude Code. If you have requirements from a product discovery (user stories, acceptance criteria, technical constraints), give them as context:

```
/write-plan path/to/requirements.md
```

Or just launch it with an idea:

```
/write-plan
```

The agent will:
1. **Discuss**: Explore the codebase, ask you clarifying questions about requirements and technical approach, until you both have a shared understanding.
2. **Draft**: Write a detailed implementation plan to `plans/YYYY-MM-DD_feature-name.md`.
3. **Review the plan** (fresh agent): Check for gaps, wrong assumptions, integration risks. Auto-corrects the plan.
4. **Check standards** (fresh agent): Verify the plan respects project coding and testing conventions. Auto-corrects.
5. **Break into steps** (fresh agent): Split the plan into ordered, atomic implementation steps with TDD guidance.

The command ends here. You review the plan — manually or by asking the agent for changes.

## 2. Implement

Once the plan looks good, clear context and run:

```
/implement-plan plans/YYYY-MM-DD_feature-name.md
```

The orchestrator executes each step from the plan:

**For each step:**
- **Implementer** (fresh agent): Writes failing tests first (for business logic), then implements. Runs tests, lint, typecheck. Commits.
- **Drift checker** (fresh agent): Verifies the step produced exactly what the plan asked for — nothing more, nothing less. Flags issues if found.

**After all steps:**
- **Standards review** (fresh agent): Checks the full diff against project coding standards. Auto-applies mechanical fixes.
- **Final review** (fresh agent): Reads the full plan and full diff. Produces remarks — coverage gaps, deviations, risks, test quality, things to watch.

## 3. Review and adjust

Check the result. The plan is still there as the reference — you can point to specific sections and ask for adjustments.

## Why fresh context at each pass?

A single long conversation degrades in quality as context fills up. An agent that just spent 20 minutes implementing code is not in the right mindset to review coding standards: it's biased toward defending what it just wrote. A fresh agent with only the diff and the standards document has no such bias.

Same principle as code review — the reviewer shouldn't be the author.

## Why clear context between planning and implementation?

The discussion phase fills context with exploration, questions, dead ends, and design alternatives. By the time you run `/implement-plan`, you want a fresh agent that reads only the plan — no residual bias from the discussion.
