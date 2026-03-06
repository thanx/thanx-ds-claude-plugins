# Development Guidelines

## Plugin Structure

```text
plugins/ds/
  .claude-plugin/plugin.json   # Plugin manifest
  commands/*.md                # Slash commands
  agents/*.md                  # Agents (launched via Task() from commands)
  skills/*/SKILL.md            # Knowledge bases
```

## Writing Commands

- Every command file must start with YAML frontmatter containing a `description` field
- Use numbered steps (## Step 1, ## Step 2, etc.)
- Reference `$ARGUMENTS` to access user input
- Keep each step deterministic and measurable
- Commands must be safe - no destructive actions without confirmation
- Prefer wrapping existing MCP tools and integrations over building from scratch

## Writing Agents

- Frontmatter must include `description` and `capabilities`
- Define clear input/output contracts
- Include "When to Use This Agent" and "Process" sections

## Writing Skills

- One directory per skill, with a `SKILL.md` inside
- Skills are reference material, not executable workflows
- Keep content factual and up to date

## MCP Access

Commands can use these MCP tools:

- **Front:** Partner conversation search, message history, ticket status
- **Jira:** Ticket lookup, search, status updates (BUGS-*, DEVSUPP-*)
- **Keystone:** Knowledge search, Datadog logs, Sentry issues, Snowflake queries
- **Thanx Docs:** API documentation search
- **Notion:** Documentation search, page creation
- **Slack:** Channel notifications (gated - draft only, human sends)

## Verification Commands

Run before every PR, in this order:

```bash
# 1. Markdown lint
npm run lint:md

# 2. Link validation (internal and external)
npm run test:links

# 3. Schema + frontmatter validation
npm test

# 4. Full plugin validation
npm run validate
```

```bash
# Run all checks in order, fail-fast
npm run lint:md && npm run test:links && npm test && npm run validate
```

All four must pass. Fix before pushing.

### Common Failures

| Error | Cause | Fix |
|-------|-------|-----|
| Missing frontmatter | New .md file without YAML header | Add `---\ndescription: ...\n---` |
| Trailing whitespace | Markdown lint rule MD009 | Run `npm run lint:md:fix` |
| Broken link | Reference to moved/deleted file | Update the link target |
| Schema error | Invalid JSON in plugin.json | Check JSON syntax |

## Configuration

**Source of truth:** `plugins/ds/skills/ds-team-config/SKILL.md`

All team roster data, Slack channel IDs, and Jira config live in the skill file.
Update the skill first, then update any hardcoded values in command files.

## Error Correction Log

When Claude makes a repeated mistake in this repo, add it here.

| Date | Mistake | Correction |
|------|---------|------------|

## Versioning

Update the version in `plugins/ds/.claude-plugin/plugin.json` when merging changes.
Follow semver: patch for fixes, minor for new commands/agents/skills, major for breaking changes.
