# thanx-ds-claude-plugins

Claude Code plugin marketplace for the Thanx Dev Support team.

Shared repo for Dev Support-specific commands, agents, and skills.
Build and iterate on your local machine.
When something is ready to share, contribute it here.

## Quick Start

```bash
# Install the Dev Support plugin
claude plugin add github:thanx/thanx-ds-claude-plugins

# Use a command
/ds:triage <front_url_or_description>
```

This plugin is designed to run alongside `thanx-claude-plugins`.
Commands that already exist in the main plugin
(like `/investigate`, `/review`, `/commit`) are not duplicated here.

## What's Included

### Commands

| Command | Description |
|---------|-------------|
| `/ds:triage` | Triage an inbound partner support ticket - classify by Front tag, apply routing rules, draft response |
| `/ds:improve` | End-of-session improvement loop - capture learnings, patch commands, compound quality |
| `/ds:diagnose` | Diagnose a partner integration email - classify type, query docs, generate integration guide |
| `/ds:ask-keystone` | Research a Thanx platform question using Keystone, publish answer to Notion |
| `/ds:merchant-offboarder` | Guide the full merchant offboarding process - app store removal, admin deactivation, BReact cleanup |
| `/ds:certify` | Validate a partner integration against Thanx API docs using DataDog sandbox traces |

### Agents

_Add shared Dev Support agents here as they mature._

### Skills

_Add shared Dev Support knowledge bases here as they mature._

## Contributing

### Adding a Command

1. Create a new `.md` file in `plugins/ds/commands/`
2. Add YAML frontmatter at the top:

```yaml
---
description: Brief description of what the command does
---
```

1. Write the command body with numbered steps
1. Run `npm test` to validate structure
1. Open a PR

### Adding a Skill

1. Create a new directory in `plugins/ds/skills/`
2. Add a `SKILL.md` file inside it
3. Skills are knowledge bases that agents and commands reference

### Adding an Agent

1. Create a `.md` file in `plugins/ds/agents/`
2. Add YAML frontmatter with `description` and `capabilities`
3. Agents are launched via `Task()` calls from commands

## Development

```bash
# Install dependencies
npm ci

# Run all tests
npm test

# Lint markdown files
npm run lint:md

# Check links
npm run test:links

# Validate plugin structure
npm run validate
```

## CI

CircleCI runs on every push:

- Jest tests (schema validation, markdown linting)
- Markdown lint check
- Link validation
- Plugin structure validation (via Claude CLI)

## Relationship with thanx-claude-plugins

This repo is for **Dev Support-specific** workflows.
The main [thanx-claude-plugins](https://github.com/thanx/thanx-claude-plugins) repo
contains engineering-wide commands like `/investigate`, `/review`, `/commit`, etc.
Both plugins can be installed simultaneously without conflict.
