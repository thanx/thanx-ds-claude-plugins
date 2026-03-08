---
description: Validate a partner integration against Thanx API documentation and generate a certification report.
---

# Certify Partner Integration

Validate a partner's API integration by checking every API call they make against
the official Thanx documentation. Accepts partner-provided payloads, headers, and
endpoint references, verifies each one against thanx-docs MCP, and generates a
certification report with PASS/FAIL per check and actionable remediation steps.

## Usage

```bash
/ds:certify <partner> <evidence>
```

**Required:**
- `partner` — Partner name or Front/Jira URL with certification context
- `evidence` — API call details: pasted payloads, curl commands, Postman exports,
  code snippets, or a description of every API call the partner makes

**Optional (inline flags):**
- `--type <integration_type>` — Override integration type detection
- `--recheck` — Re-certification mode: validate only previously failed items plus
  a sanity check on passed items

---

You are executing the `/ds:certify` command.

## Step 1: Gather Partner Context

Arguments: $ARGUMENTS

Determine the partner and collect integration context:

| Input Type | How to Parse |
|---|---|
| Partner name | Check if `partners/[name].md` exists — read for integration type and history |
| Front URL (`front.com/...` or `cnv_*`) | Use Front MCP tools to fetch conversation context |
| Jira URL or key (`DEV-*`, `DEVSUPP-*`) | Use Jira MCP tools to fetch issue details |
| Free-text | Parse partner name and integration details directly |

If Front or Jira MCP tools are unavailable or return an error, tell the user which
tool failed and ask them to paste the relevant details. Do not guess.

**Credential suppression:** If the partner's API keys, bearer tokens, OAuth client
secrets, or other credentials are included in the evidence, redact them in all output.
Replace with `[REDACTED]`. Never include real credentials in the certification report,
emails, or any other output.

Display the partner summary:

```text
PARTNER CONTEXT
───────────────
Partner:        [name]
Integration:    [type — Native Ordering / Loyalty API / Consumer UX / POS / etc.]
POS Partner:    [if applicable]
Ordering:       [if applicable]
Certification:  [first / re-certification]
Source:         [Front / Jira / partner file / free-text]
```

If integration type is unknown, stop and ask the user before proceeding.

## Step 2: Parse API Evidence

Extract every API call from the partner-provided evidence. For each call, capture:

- HTTP method and endpoint URL
- Headers (especially `Accept-Version`, `Content-Type`, `Authorization` type)
- Request body (fields, values, structure)
- Response handling (if provided — how they process the response)

Organize into a numbered list:

```text
API CALLS IDENTIFIED
────────────────────
1. [METHOD] [endpoint] — [purpose]
   Headers: [key headers noted]
   Body fields: [list of fields sent]

2. [METHOD] [endpoint] — [purpose]
   ...
```

**If the evidence is ambiguous or incomplete**, list what was found and ask the user
to clarify before proceeding. Do not assume missing calls exist or fabricate payloads.

## Step 3: Fetch Documentation for Each Endpoint

For every API call identified in Step 2, search the official Thanx documentation
using `SearchThanx` (thanx-docs MCP).

**This step is mandatory.** The documentation is the single source of truth for
what the partner's integration should look like. Do not validate against assumptions
or general knowledge — only against what the docs specify.

For each endpoint, search for and record:

| Documentation Element | What to Capture |
|---|---|
| Endpoint URL and method | Correct path, HTTP method, URL parameters |
| Required headers | `Accept-Version`, `Content-Type`, auth type |
| Request body schema | Required fields, optional fields, field types, valid values |
| Response schema | Expected response structure, status codes |
| Authentication | OAuth flow, API key, scopes required |
| Rate limits | If documented, capture limits |
| Versioning | Which API version the endpoint belongs to |

**Max 15 `SearchThanx` queries** across this step. Batch related endpoints into
single queries where possible (e.g., "purchases endpoint request body fields").

If the thanx-docs MCP server is unavailable, errors, or times out, stop and tell the
user. Do not proceed with certification without documentation verification. The
entire value of this command depends on doc-based validation.

Optionally use Keystone `search_code` (max 5 queries) to verify undocumented behavior
if a specific field or response pattern is not covered in the docs. Flag any answer
from Keystone as needing verification.

## Step 4: Validate Each API Call

For every API call from Step 2, compare against the documentation from Step 3.
Run these checks:

### 4a: Authentication and Headers

| Check | What to Validate |
|---|---|
| Auth type | Correct auth method (OAuth client credentials, API key, bearer token) |
| Auth flow | Token request uses correct grant type and scopes |
| Accept-Version | Correct API version header present |
| Content-Type | `application/json` or as required by endpoint |
| Required headers | All required headers present |

### 4b: Endpoint and Method

| Check | What to Validate |
|---|---|
| URL path | Correct endpoint path, no typos, correct resource IDs |
| HTTP method | Correct method (POST vs PUT vs PATCH) |
| URL parameters | Required query params present, correct format |

### 4c: Request Body

| Check | What to Validate |
|---|---|
| Required fields | All required fields present per docs |
| Field names | Correct field names (no typos, correct casing) |
| Field types | Correct types (string vs integer vs array) |
| Field values | Values match expected format (ISO dates, valid enums, etc.) |
| Points accrual field | Correct field used — `subtotal` not `amount` (common mistake) |
| Nested objects | Correct nesting structure for complex fields (items, payments) |
| Extra fields | Any fields sent that are not in the docs (may be ignored or cause errors) |

### 4d: Response Handling

| Check | What to Validate |
|---|---|
| Success handling | Partner processes 2xx responses correctly |
| Error handling | Partner handles 4xx/5xx responses appropriately |
| Required fields read | Partner reads the correct response fields |
| Idempotency | Handles duplicate requests gracefully if applicable |

### 4e: Integration-Type-Specific Checks

Based on the integration type from Step 1, apply additional checks:

| Integration Type | Additional Checks |
|---|---|
| Loyalty API Direct | No orders expected (only purchases). Refund clawback does NOT happen automatically. |
| POS / Kiosk | Provider ticket lifecycle (checkout → billed). Payment amount calculation. |
| Consumer UX | Auth flow for end users. Reward exchange flow. Bonus points activation delay (~20 min). |
| Native Ordering | Order creation flow. Menu sync if applicable. |

### 4f: Call Sequencing

Verify the partner's API calls happen in the correct order:

- Authentication before any data calls
- User lookup/creation before purchases
- Purchase creation before reward checks
- Proper webhook registration if event-driven

For each check, assign a result:

- **PASS** — Matches documentation
- **FAIL** — Does not match documentation (include expected vs actual)
- **WARNING** — Not wrong but could cause issues (e.g., deprecated field, missing
  optional but recommended field)
- **INSUFFICIENT DATA** — Partner did not provide enough detail to validate

## Step 5: Generate Certification Report

Compile all results into a structured report:

```text
CERTIFICATION REPORT
════════════════════
Partner:        [name]
Integration:    [type]
Date:           [today's date]
Mode:           [first certification / re-certification]
────────────────────

SUMMARY
───────
Total checks:   [N]
Passed:         [N] ✓
Failed:         [N] ✗
Warnings:       [N] ⚠
Insufficient:   [N] ?

OVERALL RESULT: [CERTIFIED / NOT CERTIFIED / INCOMPLETE]
────────────────────

DETAILED RESULTS
────────────────

1. Authentication and Headers
   [✓/✗/⚠/?] Auth type: [result]
   [✓/✗/⚠/?] Accept-Version: [result]
   [✓/✗/⚠/?] Content-Type: [result]
   ...

2. Endpoint: [METHOD] [path]
   [✓/✗/⚠/?] URL path: [result]
   [✓/✗/⚠/?] Required fields: [result]
   [✓/✗/⚠/?] Field mapping: [result]
   [✓/✗/⚠/?] Points accrual: [result]
   ...
   Documentation reference: [doc page or section from SearchThanx]

3. [repeat for each endpoint]
   ...

────────────────────

FAILURES — REMEDIATION REQUIRED
────────────────────────────────
[For each FAIL, list:]

F1: [check name]
    Endpoint:  [METHOD] [path]
    Expected:  [what the docs say]
    Actual:    [what the partner sent]
    Fix:       [specific remediation — field name, value, format]
    Doc ref:   [documentation reference]

F2: ...

────────────────────

WARNINGS
────────
[For each WARNING:]

W1: [description]
    Recommendation: [what to improve]

────────────────────

MISSING EVIDENCE
────────────────
[For each INSUFFICIENT DATA:]

M1: [what was not provided]
    Needed:    [what we need to validate this check]
```

## Step 6: Draft Communication

Based on the certification result, draft the appropriate communication.

**Internal details guardrail:** Never include Keystone code search results, internal
repo paths, or system architecture details in the partner-facing email.

### CERTIFIED → Confirmation Email

```text
DRAFT CERTIFICATION EMAIL
─────────────────────────
Subject: [Partner Name] — Integration Certified

[Confirmation that integration passed all checks]
[Outline next steps for launch]
[Any warnings to address before going live — not blockers but recommendations]

─────────────────────────
Next step: To send via Front, use /ds:draft-email with the conversation ID.
```

### NOT CERTIFIED → Failure Email with Remediation

```text
DRAFT REMEDIATION EMAIL
───────────────────────
Subject: [Partner Name] — Certification Results: Action Required

[Brief summary: N checks passed, N need remediation]

[For each failure:]
Issue [N]: [clear description]
- What we found: [actual behavior/payload]
- What is expected: [per documentation]
- How to fix: [specific steps — endpoint, field, value]

[Set expectation for re-submission]
[Offer to answer questions]

───────────────────────
Boundary Doctrine: applied (specific remediation, no scope creep)
Next step: To send via Front, use /ds:draft-email with the conversation ID.
```

### INCOMPLETE → Missing Evidence Request

```text
DRAFT EVIDENCE REQUEST
──────────────────────
Subject: [Partner Name] — Additional Information Needed for Certification

[List what was validated and passed so far]
[List what could not be validated and why]
[Specific request for the missing evidence — exact format needed]

──────────────────────
```

## Step 7: Update Tracking

Propose updates for the user to review and manually apply:

```text
PROPOSED PARTNER FILE UPDATE
─────────────────────────────
File: partners/[name].md
Update certification status to: [CERTIFIED / NOT CERTIFIED / INCOMPLETE]
Add certification date: [today]
Add certification notes: [summary of results]
```

If there are action items (re-certification deadline, follow-up needed), propose
an `action-items.md` entry.

If re-certification (`--recheck` flag), note which previously failed items now pass
and which still need work.

## Rules

1. **thanx-docs MCP is the single source of truth.** Every validation must be checked
   against the official documentation. Do not validate against assumptions, general
   knowledge, or Keystone code alone.
2. **If thanx-docs MCP is unavailable, stop.** Do not proceed with certification
   without documentation verification. Tell the user and suggest retrying later.
3. **Max 15 SearchThanx queries + 5 Keystone queries** in Step 3. Batch related
   endpoints to stay within budget.
4. **Keystone answers on undocumented behavior need verification.** If a field or
   behavior is not in the docs but found in code, flag it as needing Darren
   verification before including in the certification result.
5. **Redact all credentials.** Never include API keys, bearer tokens, or secrets
   in any output — report, email, or tracking update.
6. **Never include internal details in partner-facing output.** Keystone code
   results, internal repo paths, and system architecture belong only in the
   certification report, not in the partner email.
7. **Apply boundary doctrine on every communication.** Provide specific remediation
   steps but do not extend scope beyond what was submitted for certification.
8. **Knowledge base files are optional.** If `partners/[name].md`, `knowledge/*.md`,
   or other Second Brain files do not exist, skip them silently and continue.
9. **Do not send any email, update any ticket, or post to Slack.** Present
   everything for human review and approval.
