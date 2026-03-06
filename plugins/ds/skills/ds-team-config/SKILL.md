---
description: Dev Support team roster, channel IDs, Jira config, and key integrations. Reference this skill instead of hardcoding values in commands.
last_verified: "2026-03-05"
---

# Dev Support Team Configuration

This is the **single source of truth** for Dev Support team data. All commands and agents should reference this skill
instead of hardcoding values.

## Team Roster

| Name | Slack ID | Role | GitHub | Jira ID |
|------|----------|------|--------|---------|
| Andre Lunardi | `U08M3LNK01F` | Dev Support Engineer | `adrlunardi` | `ALUNARDI` |
| Mateo Francesco Mantovano | `U07QPK90QDV` | Dev Support Engineer | `mantovanoMateo` | `MMANTOVANO` |

**Engineering Manager:** Juliano Coimbra (`U06MDTUHKTP`, GitHub: `jmcoimbra`)

**IC Slack IDs (for filtering):** `U08M3LNK01F`, `U07QPK90QDV`

## Slack Channels

| Channel | ID | Type |
|---------|-----|------|
| `#rnd-dev-supp-internal` | `C08MRK8SXPA` | Internal - team coordination and discussions |
| `#developer-support-tickets` | `C01J28JKMGV` | Front bot ticket notifications |

## Jira Configuration

| Identifier | Value |
|------------|-------|
| Jira Cloud ID | `7d5d6532-069d-419b-bd1c-d8321b134435` |
| OPS project key | `OPS` |
| BUGS project key | `BUGS` |
| DEVSUPP project key | `DEVSUPP` |

## Key Integrations

| System | Purpose |
|--------|---------|
| Front | Partner support conversations, ticket management |
| Keystone | Internal knowledge search, Datadog logs, Sentry issues, Snowflake queries |
| Thanx Docs | API documentation search |
| Jira | Bug tracking (BUGS-*), dev support requests (DEVSUPP-*) |
