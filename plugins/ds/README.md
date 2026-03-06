# Dev Support Plugin

Claude Code plugin for Thanx Dev Support team workflows.

## Commands

| Command | Description |
|---------|-------------|
| `/ds:triage` | Triage an inbound partner support ticket - classify, apply routing rules, draft response |
| `/ds:improve` | End-of-session improvement loop - capture learnings, patch commands, compound quality |
| `/ds:diagnose` | Diagnose a partner integration email - classify type, query docs, generate integration guide |
| `/ds:ask-keystone` | Research a Thanx platform question using Keystone, publish answer to Notion |
| `/ds:weekly-status` | Generate sync talking points (Mon/Wed/Fri) or async standup updates (Tue/Thu) from Front, Slack, and Jira |
| `/ds:troubleshoot` | Investigate a partner-reported issue, diagnose root cause, and draft a resolution or escalation |

## Adding New Commands

1. Create a `.md` file in `commands/`
2. Add YAML frontmatter with a `description` field
3. Follow the numbered step pattern (see existing commands)
4. Run `npm test` to validate

## Adding New Skills

1. Create a directory in `skills/` with a `SKILL.md` file
2. Skills are knowledge bases that agents and commands can reference

## Adding New Agents

1. Create a `.md` file in `agents/`
2. Add YAML frontmatter with `description` and `capabilities`
3. Agents are launched via `Task()` calls from commands
