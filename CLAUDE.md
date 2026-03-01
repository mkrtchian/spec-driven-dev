# CLAUDE.md

## Conventions

- Skills go in `skills/<name>/SKILL.md` with YAML frontmatter
- Custom agent definitions go in `agents/sdd-*.md` with YAML frontmatter (plugin root, distributed via plugin install)
- Use `/commit` for all commits (conventional commits, no git add -A)
- Increment `version` in `.claude-plugin/plugin.json` on every functional change to skills or agents

## Don't

- Don't add code or tooling — this repo is markdown prompts only
- Don't hardcode stack-specific commands in skills — they must discover project config dynamically
