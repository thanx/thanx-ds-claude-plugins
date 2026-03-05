---
description: Generate sync talking points (Mon/Wed/Fri) or async standup updates (Tue/Thu) from Front, Slack, and Jira data.
---

# Weekly Status

Generate a status update for Dev Support syncs or async standups by pulling live data from Front, Slack, and Jira.

## Usage

```bash
/ds:weekly-status [--sync | --async]
```

**Options:**

- `--sync` - Generate talking points for the live sync call (Mon/Wed/Fri)
- `--async` - Generate Yesterday / Today / Blockers for the Slack thread (Tue/Thu)
- No flag: auto-detect based on the current day of the week

---

You are executing the `/ds:weekly-status` command.

## Step 1: Determine Mode

Arguments: $ARGUMENTS

Check for an explicit `--sync` or `--async` flag. If neither is provided, auto-detect based on the current day of the week:

- **Monday, Wednesday, Friday** → Sync mode
- **Tuesday, Thursday** → Async mode
- **Saturday, Sunday** → Prompt user to specify `--sync` or `--async`

Announce the selected mode:

> Generating **sync prep** for today's call.

or

> Generating **async standup** for the Slack thread.

## Step 2: Gather Front Data

Pull the current state of the Dev Support inbox using Front MCP tools. To avoid rate limit exhaustion (each Front search costs ~40% of the rate limit budget), consolidate into a single broad query and classify results client-side.

1. **Single search**: Use `front_search_conversations` to query the Dev Support inbox for all recent conversations across all states (open, waiting, resolved). Limit to the 10 most recent conversations to stay within rate limits.
2. **Classify results** from the single search into:
   - **Open tickets**: Conversations still awaiting a DS response. For each, capture conversation ID, subject, partner/merchant name, last message date and sender, and Front tags.
   - **Waiting tickets**: Conversations where DS sent the last message. Note how many business days since the last DS reply.
   - **Recently resolved**: Conversations resolved in the last 2 business days. Capture what was resolved and for which partner.
3. **Escalation signals**: From the classified results, flag any ticket where:
   - Partner has been waiting more than 2 business days for a DS response
   - The conversation has a "Bug" or "Certification Process" tag with no recent activity
   - Multiple follow-ups from the partner without a DS reply

If Front MCP tools are unavailable, report this and proceed with Slack and Jira data only.

## Step 3: Gather Slack Data

Search recent Slack activity relevant to Dev Support:

1. Use `slack_search_public_and_private` to find messages in #rnd-dev-supp-internal from the last 2 business days
2. Look for:
   - Threads mentioning partner names or ticket IDs from Step 2
   - Engineering responses to escalations
   - Action items assigned to DS members
   - Announcements or process changes affecting the team
3. Use `slack_read_channel` on #developer-support-tickets for recent emoji status updates (eyes, envelope, checkmark)

If Slack MCP tools are unavailable, report this and proceed with the data already gathered.

## Step 4: Gather Jira Data

Search for relevant Jira activity:

1. Use `jira_search_issues` with JQL to find:
   - DEVSUPP tickets updated recently: `project = DEVSUPP AND updated >= -4d ORDER BY updated DESC`
   - BUGS tickets created or updated by DS recently: `project = BUGS AND updated >= -4d AND comment ~ "Dev Support" ORDER BY updated DESC`
   - **Note:** JQL `-Nd` counts calendar days, not business days. Use `-4d` to over-fetch, then filter results to the last 2 business days during output generation.
2. For each ticket, capture:
   - Issue key, summary, status, assignee
   - Whether it is blocked or waiting on engineering
   - Recent status transitions (e.g., moved to "In Progress", "Done", "Needs Info")

If Jira MCP tools are unavailable, report this and proceed with the data already gathered.

## Step 5: Generate Output

### Sync Mode (Mon/Wed/Fri)

Organize the gathered data into talking points ordered by priority:

```text
Sync Prep — [Day, Month Date]
==============================

HOT TOPICS
- [Partner] — [Issue summary]. [Why it's hot: SLA risk / partner escalating / engineering-blocked / etc.]
  Front: [cnv_id] | Jira: [ticket key if applicable]

- [Partner] — [Issue summary]. [Why it's hot]
  Front: [cnv_id] | Jira: [ticket key if applicable]

ACTIVE ITEMS
- [Partner] — [Status update, what's in progress, next step]
  Front: [cnv_id]

- [Partner] — [Status update]
  Front: [cnv_id]

NEWS & UPDATES
- [What happened: PR merged, process change, command shipped, etc.]
  Ref: [Jira: key | PR: link | N/A for general updates]
- [Resolved ticket worth mentioning]
  Front: [cnv_id] | Jira: [ticket key if applicable]
- [Jira ticket status change]
  Jira: [ticket key]
```

**Hot topic criteria** (any of these qualifies):
- Partner waiting more than 2 business days for a DS response
- Ticket tagged as Bug or Certification with no progress
- Engineering escalation with no response
- Multiple partner follow-ups without a DS reply
- SLA close to breach

### Async Mode (Tue/Thu)

Generate the standup in Yesterday / Today / Blockers format:

```text
Yesterday
- [Specific action with partner name and ticket reference]
- [Specific action]

Today
- [Planned action with partner name and ticket reference]
- [Planned action]

Blockers
- [Blocker with context — what's blocking and who can unblock]
```

If there are no blockers, write:

```text
Blockers
- None
```

**Rules for async output:**
- Each bullet must reference a specific partner, ticket ID, or Jira key
- "Yesterday" = the most recent business day (Mon-Fri), regardless of how async mode was triggered. If today is Tuesday, yesterday is Monday. If `--async` is forced on a Monday, yesterday is Friday.
- "Today" items should be actionable and specific, not vague ("Investigate Giordanos DEV-7467", not "Work on tickets")
- Keep it concise — this gets pasted into a Slack thread

## Step 6: Present for Review

Present the generated output to the user:

```text
Status Update Draft
-------------------
Mode: [Sync | Async]
Sources checked: [Front ✓/✗] [Slack ✓/✗] [Jira ✓/✗]
Date: [today's date]

[Generated output from Step 5]
```

If any data source was unavailable, note it:

> **Note:** [Source] was unavailable. This update may be incomplete — review before posting.

Do not send any message, post to any Slack channel, or update any ticket. Present everything for human review and approval.

## Rules

1. **Never post directly to Slack or any external service.** This command is draft-only. Present the output for the user to copy and post manually.
2. **Always include ticket references.** Every bullet point should reference a Front conversation ID, Jira key, or both. Vague bullets without references are not useful.
3. **Hot topics go first in sync mode.** The team lead wants to address urgent items first. Do not bury escalation-risk items under routine updates.
4. **Be honest about data gaps.** If a data source was unavailable, say so. Do not fabricate status from memory or assumptions.
5. **Keep async updates concise.** The Slack thread format should be scannable — aim for 3-7 bullets per section, not a wall of text.
6. **Use business days for timing.** "2 days waiting" means 2 business days, not calendar days. Weekends do not count.
7. **Do not include automated noise.** Exclude Apple membership notifications, automated system emails, automated Jira transitions, CI/CD build notifications, bot-generated comments, and other non-actionable items from the status update.
