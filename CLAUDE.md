# CLAUDE.md

## Conventions

- Skills go in `skills/<name>/SKILL.md` with YAML frontmatter
- Custom agent definitions go in `agents/sdd-*.md` with YAML frontmatter (plugin root, distributed via plugin install)
- Use `/commit` for all commits (conventional commits, no git add -A)
- Increment `version` in `.claude-plugin/plugin.json` on every functional change to skills or agents

## Releasing

Distribution is driven by the `version` in `plugin.json`, not by GitHub Releases, but the public Releases page must stay truthful.

- After a version bump, when the user wants to push, propose creating the matching GitHub release (`vX.Y.Z` tag on the pushed commit) and wait for their confirmation. Never create it automatically.
- Release notes: English, and limited to functional changes in the distributed skills and agents. Exclude docs-only and internal-tooling changes.

## Don't

- Don't add code or tooling — this repo is markdown prompts only
- Don't hardcode stack-specific commands in skills — they must discover project config dynamically
