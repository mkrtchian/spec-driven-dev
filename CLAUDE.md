# CLAUDE.md

## Conventions

- Skills go in `skills/<name>/SKILL.md` with YAML frontmatter
- Supporting prompts go alongside their skill as `<name>-prompt.md` (no frontmatter)
- Use `/commit` for all commits (conventional commits, no git add -A)

## Don't

- Don't add code or tooling — this repo is markdown prompts only
- Don't hardcode stack-specific commands in skills — they must discover project config dynamically
