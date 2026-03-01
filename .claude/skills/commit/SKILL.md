---
name: commit
description: "Stage and commit changes with conventional commit messages"
disable-model-invocation: true
---

## Objective

Create a clean, well-scoped git commit following conventional commits.

## 1. Understand the changes

Run in parallel:

- `git status` (never use `-uall`)
- `git diff` and `git diff --cached` to see staged and unstaged changes
- `git log --oneline -5` to see recent commit style

Read changed files if needed to understand intent — don't guess from filenames.

## 2. Stage

- Stage relevant files by name — never `git add -A` or `git add .`
- Never stage files that look like secrets (`.env`, credentials, tokens)
- If unrelated changes are mixed, ask the user whether to split into multiple commits

## 3. Write the commit message

Follow conventional commits: `type(scope): description`

**Types** (pick the most accurate):

- `feat` — new functionality
- `fix` — bug fix
- `docs` — documentation only
- `refactor` — code change that neither fixes a bug nor adds a feature
- `test` — adding or updating tests
- `chore` — build, CI, dependencies, tooling

**Rules**:

- Subject line only — never add a body. One line is enough.
- Subject line: imperative mood, lowercase, no period
- Scope is optional — use it when it clarifies (e.g., `feat(auth):`, `fix(parser):`)
- Never use a scope just to have one
- Always add a `Co-Authored-By` trailer with your own model name and version (you know which model you are)
- In this repo, skills and agents **are** the product — changes to them are almost always `feat` or `fix`, not `refactor`. A rewording that changes behavior is a `fix`; a new capability is a `feat`. Reserve `refactor` for pure structural reorganization with no behavioral change.

## 4. Commit

Use a HEREDOC for the message:

```bash
git commit -m "$(cat <<'EOF'
type(scope): description

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

## 5. Confirm

Run `git status` after commit to verify clean state. Show the user the commit hash and message.
