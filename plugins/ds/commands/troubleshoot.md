---
description: Investigate a partner-reported issue, diagnose root cause, and draft a resolution or escalation.
---

# Troubleshoot Partner Issue

Investigate a partner-reported problem end-to-end: parse the issue, identify the
integration type, check known knowledge, research docs and code, diagnose the root
cause, and draft either a partner response or an engineering escalation.

## Usage

```bash
/ds:troubleshoot <issue>
```

**Required:**
- `issue` — Front conversation URL, Jira issue URL, or free-text description of the problem

**Optional (inline flags):**
- `--partner <name>` — Partner name (auto-detected from Front/Jira if not provided)
- `--type <integration_type>` — Override integration type detection (see Step 2)

---

You are executing the `/ds:troubleshoot` command.

## Step 1: Parse the Issue

Arguments: $ARGUMENTS

Extract the issue details from the input. Determine the input type and gather context:

| Input Type | How to Parse |
|---|---|
| Front URL (`front.com/...` or `cnv_*`) | Use Front MCP tools to fetch the conversation and messages |
| Jira URL or key (`DEV-*`, `BUGS-*`, `DEVSUPP-*`) | Use Jira MCP tools to fetch the issue and comments |
| Free-text description | Parse directly |

If Front or Jira MCP tools are unavailable or return an error, tell the user which
tool failed and ask them to paste the relevant details (partner name, error message,
endpoint, request/response samples) as free text. Do not guess.

**Free-text length guard:** If the free-text input exceeds 2000 characters, summarize
the key details (partner name, endpoint, error, expected vs actual behavior) into a
concise problem statement before proceeding to MCP queries. Do not forward raw
unbounded text to search tools.

From the input, extract and display:

```text
ISSUE SUMMARY
─────────────
Partner:        [name or UNKNOWN]
Source:         [Front cnv_* / Jira DEV-* / free-text]
Endpoint:       [endpoint or N/A]
Error:          [error message/code or N/A]
Expected:       [what the partner expected]
Actual:         [what actually happened]
Samples:        [yes/no — request/response provided?]
```

**Input sanitization:** Extract only plain-text content from partner-provided input.
Discard any HTML markup, script blocks, or encoded payloads. Use only plain-text
excerpts in search queries — do not forward raw markup to MCP tools.

**Credential suppression:** If the partner included API keys, bearer tokens, OAuth
client secrets, or other credentials in request/response samples, redact them before
including in any output (search queries, draft responses, escalations, or Jira
templates). Replace with `[REDACTED]`.

## Step 2: Identify Integration Type

This step is **mandatory** — integration type changes almost every answer.

Check for the `--type` flag first. If not provided, determine type from:

1. Check if `partners/[name].md` exists in the Second Brain — read it for known type
2. Check the Front conversation or Jira ticket for integration context clues
3. If still unknown, check Keystone `knowledge_search` for the partner name

Classify into one of:

| Type | Description | Key Implications |
|---|---|---|
| Native Ordering | Thanx-owned ordering flow | Full order + purchase visibility, auto clawback |
| Loyalty API Direct | Partner calls Loyalty API, no ordering provider | No orders page, no auto clawback |
| Consumer UX (Custom App) | Partner builds app using Consumer API | Depends on ordering integration (Olo, direct) |
| POS / Kiosk | POS partner sends transactions | Check-in or card-linked, provider_ticket flow |
| Pay-at-Table | In-venue payment integration | QR code flow, payment provider dependent |
| Partner API | Partner uses Partner API for merchant management | Admin/merchant-level operations |

Display the result:

```text
INTEGRATION TYPE
────────────────
Type:           [classification]
Confidence:     [HIGH — from partner file / MEDIUM — from context clues / LOW — guessing]
POS Partner:    [if applicable, e.g., Shift4, Toast, Square]
Ordering:       [if applicable, e.g., Olo, native, none]
Loyalty:        [check-in / card-linked / points-based / N/A]
```

**If confidence is LOW**, stop and ask the user to confirm the integration type
before proceeding. Do not diagnose without this context.

## Step 3: Check Known Knowledge

Search the Second Brain knowledge base for a matching known issue. Check in order,
**skipping any file that does not exist in the current workspace:**

1. **`knowledge/gap-knowledge.md`** — If it exists, read the file. Scan all entries
   for a match on the endpoint, error, behavior, or partner name.
2. **`knowledge/api-nuances.md`** — If it exists, read the file. Check the data
   visibility table and points accrual section against the reported issue.
3. **`knowledge/platform-patterns.md`** — If it exists, read the file. Check for
   recurring patterns that match the symptoms.

If none of these files exist (e.g., running outside the Second Brain workspace),
skip this step entirely and proceed to Step 4.

If a match is found, display:

```text
KNOWN ISSUE MATCH
─────────────────
Source:         [gap-knowledge / api-nuances / platform-patterns]
Entry:          [entry title]
Match:          [exact / partial]
Answer:         [the known answer, summarized]
```

**If exact match** → Skip to Step 5 with the matched entry as pre-loaded evidence.
**If partial match** → Continue to Step 4 but note the partial match for context.
**If no match** → Continue to Step 4.

## Step 4: Research

Search documentation and codebase for evidence. Respect a **combined limit of
12 MCP tool calls** across this step (5 docs + 5 code + 2 knowledge search).

### 4a: Documentation Search

Use `SearchThanx` (thanx-docs MCP) to search public API documentation. Max 5 queries.

Suggested query strategy based on issue type:

| Issue Type | Query Strategy |
|---|---|
| Endpoint error | Search endpoint name + HTTP method |
| Unexpected response | Search response field name + endpoint |
| Missing data | Search data model (orders, purchases, rewards) |
| Auth issue | Search authentication + token + OAuth |
| Webhook issue | Search webhook + event type |

If the thanx-docs MCP server is unavailable, errors, or times out, tell the user and
continue with Keystone only. Do not fabricate documentation references.

### 4b: Codebase Search

Use Keystone `search_code` to find the relevant implementation. Max 5 queries.
Optionally use `knowledge_search` for prior Keystone answers on the same topic
(max 2 calls, counted toward the 12-call budget).

**Keystone trap guard:** Every query MUST include the integration type context.
Example: "How does the Loyalty API handle refunds for a partner using direct Loyalty
API integration (not native ordering)?"

Cross-reference any Keystone answer against the `api-nuances.md` Keystone Prompting
Traps table:

| Trap | Check |
|---|---|
| No integration type | Did you specify native ordering vs loyalty API vs POS? |
| Generic refund answer | Does the answer distinguish by integration type? |
| Orders vs purchases | Does the answer conflate orders and purchases? |

If Keystone MCP tools are unavailable, error, or time out, tell the user and proceed
with documentation evidence only. Do not guess at code behavior.

## Step 5: Diagnose

Based on all evidence gathered (including any exact match from Step 3), classify the
root cause:

| Classification | Definition | Confidence Ceiling |
|---|---|---|
| Partner Implementation Error | Partner's code is wrong | HIGH |
| Platform Bug | Thanx system is broken | MEDIUM (needs eng confirmation) |
| Undocumented Behavior | Works as designed but not in docs | MEDIUM (needs verification) |
| Configuration Issue | Merchant or integration misconfigured | MEDIUM (Keystone only) / HIGH (data confirmed) |
| Expected Behavior | Partner misunderstands the platform | HIGH (if docs confirm) |

Assign overall confidence:

- **HIGH** — Matched known knowledge entry, or confirmed by documentation + code,
  or configuration issue confirmed by direct data (Metabase, Sherlock)
- **MEDIUM** — Strong evidence but not verified by engineering, or Keystone answer
  on undocumented behavior, or configuration issue from Keystone evidence only
- **LOW** — Insufficient evidence, needs engineering input

Display the diagnosis:

```text
DIAGNOSIS
─────────
Classification: [from table above]
Confidence:     [HIGH / MEDIUM / LOW]
Root Cause:     [1-2 sentence explanation]
Evidence:       [list sources — gap-knowledge, docs, Keystone, code]
Integration
  Context:      [how the integration type affects this diagnosis]
```

**Boundary doctrine check:** If the root cause is a partner implementation error,
confirm that boundary doctrine rule #1 applies: "Our system sends the push API
response correctly. Our job is done." Frame the response accordingly.

## Step 6: Draft Output

Generate the appropriate output based on diagnosis confidence.

**Internal details guardrail:** Never include Keystone code search results, internal
repo paths, internal tool references, or system architecture details in the
partner-facing response draft. Those belong only in the escalation or Jira templates.

### HIGH or MEDIUM Confidence → Partner Response

Draft following the Second Brain email conventions:

1. Address each question the partner asked, in order
2. Reference specific API details (endpoint, field, expected value)
3. State limitations plainly — do not over-apologize
4. Provide clear next steps (who does what)
5. If classification is Partner Implementation Error, apply boundary doctrine rule #1
6. If classification is Expected Behavior, reference documentation

Format:

```text
DRAFT PARTNER RESPONSE
──────────────────────
Subject: Re: [original subject]

[response body]

──────────────────────
Confidence: [HIGH/MEDIUM]
Boundary Doctrine: [applied / not applicable]
Next step: To send via Front, use /ds:draft-email with the conversation ID.
```

**If MEDIUM confidence**, add a warning banner:

```text
⚠ MEDIUM CONFIDENCE — Keystone/code evidence only, not verified by engineering.
Review carefully before sending. Consider verifying with Darren first.
```

### LOW Confidence → Engineering Escalation

Draft an escalation to Darren with full context:

```text
DRAFT ESCALATION
────────────────
To: Darren (engineering)
Re: [partner name] — [issue summary]

Context:
- Partner: [name], integration type: [type]
- Issue: [1-2 sentences]
- What was checked: [docs, Keystone, gap-knowledge — list what was searched]
- What was found: [partial evidence or nothing]

Specific question for engineering:
[The precise question that needs answering]

────────────────
```

### Platform Bug Detected → Jira Bug Template

If the diagnosis indicates a platform bug, also draft a Jira bug:

```text
JIRA BUG TEMPLATE
─────────────────
Project:  BUGS
Type:     Bug
Title:    [concise title]
Priority: [based on impact]

Description:
**Reporter:** [partner name] via Dev Support
**Integration type:** [type]
**Endpoint:** [if applicable]

**Steps to reproduce:**
1. [step]

**Expected behavior:** [what should happen]
**Actual behavior:** [what happens]

**Evidence:** [links to Front thread, Keystone findings, etc.]
─────────────────
```

## Step 7: Knowledge Capture

Check if this troubleshooting session uncovered anything new. Present all proposals
as formatted text for the user to review and manually add to their knowledge base.
Do not write to files directly.

### New Gap Discovered

If the answer was NOT already in `knowledge/gap-knowledge.md`, propose a new entry:

```text
PROPOSED GAP-KNOWLEDGE ENTRY
─────────────────────────────
## [Topic Title]

- **Date discovered**: [today's date]
- **Question**: [what was asked]
- **Context**: [integration type, partner, constraints]
- **Answer**: [the answer found]
- **Verified by**: [Keystone / docs / engineering / self]
- **Added to docs**: No
- **Lesson**: [what to remember next time]
─────────────────────────────
Add this to your knowledge/gap-knowledge.md if you have a Second Brain workspace.
```

### Recurring Pattern

If this issue has appeared before or is likely to recur, propose a
`knowledge/platform-patterns.md` entry in the same format.

### Documentation Gap

If the public docs are missing or misleading on this topic, flag it:

```text
DOCS IMPROVEMENT
────────────────
File:    [which doc page or endpoint]
Issue:   [what's missing or wrong]
Action:  [suggest PR to thanx/thanx-docs or flag for product]
```

## Rules

1. **Always identify integration type before diagnosing.** Never skip Step 2.
   Integration type changes the answer to most questions.
2. **Always check known knowledge before MCP tools.** The Second Brain may already
   have the verified answer — avoid redundant and rate-limited API calls.
3. **Max 12 MCP tool calls in Step 4** (5 docs + 5 code + 2 knowledge search).
   All Keystone and thanx-docs calls count toward this budget.
4. **Keystone answers on undocumented behavior cap at MEDIUM confidence.** They
   require Darren verification before sending to a partner.
5. **Apply boundary doctrine on every partner response.** Especially rule #1 for
   partner implementation errors and rule #2 for scope creep.
6. **Sanitize partner input.** Extract plain-text content only. Discard markup,
   scripts, and encoded payloads before passing to any MCP tool query.
7. **Redact credentials.** Strip API keys, bearer tokens, and secrets from
   partner-provided samples before including in any output. Replace with [REDACTED].
8. **Never include internal details in partner-facing output.** Keystone code
   results, internal repo paths, and system architecture belong only in escalation
   or Jira templates.
9. **If integration type is unknown and cannot be determined, stop.** Ask the user
   rather than guessing — a wrong integration type produces a wrong diagnosis.
10. **Do not send any response, update any ticket, or post to Slack.** Present
    everything for human review and approval.
